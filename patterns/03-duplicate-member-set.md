# 03 — The setter validates, the initializer doesn't

**Family:** sibling-path asymmetry · **Found in the audit:** Low ·
**Components:** both bridge programs

## The bug shape

The bridge programs have two paths that write the relayer set:

- `SetRelayers` (the update path) rejects duplicate keys and the
  default pubkey.
- `Initialize` (the creation path) checked only for default keys —
  **not** duplicates.

So a config initialized with `[R, R, R, R2, R3]` was stored exactly
as supplied. The same gap existed for the admin set in both
programs' initializers.

Why it isn't worse: attestation counting dedupes by registered-set
position, so duplicates never inflated quorum — three signatures
from `R` still counted once. The damage is quieter: a deployment
that believes it runs "3-of-5" actually runs 3-of-3-*distinct*. One
relayer's downtime or key loss now halts the bridge, and nobody
notices the discrepancy until that day. It's a deployment footgun,
not an exploit — which is exactly the class of bug that survives
review, because every individual line looks fine.

## How the PoC proved it

Initialize a config with a duplicated relayer list, read the stored
config back, assert equality with the duplicated input. It matched.
The bug is the absence of an error.

## The fix: one validator, called from both paths

Extract the member-set validation into the shared library and call
it from `Initialize` (relayers **and** admins) and from
`SetRelayers`, in both programs:

```rust
// In the shared crate — the only place this rule lives now.
pub fn validate_member_set(members: &[Pubkey]) -> ProgramResult {
    if members.is_empty() {
        return Err(BridgeError::InvalidMemberSet.into());
    }
    for (i, member) in members.iter().enumerate() {
        if *member == Pubkey::default() {
            return Err(BridgeError::InvalidMemberSet.into());
        }
        if members[..i].contains(member) {
            return Err(BridgeError::DuplicateMember.into());
        }
    }
    Ok(())
}
```

Two properties to preserve when you do this:

- **Error identity.** The shared validator should return the same
  errors the stricter path historically returned, so existing tests
  and downstream error handling don't silently change meaning.
- **Every set, not just the famous one.** The audit found the gap
  in relayers first; admins had it too. When you unify validation,
  enumerate *all* the sets the initializer accepts.

## The general lesson

Sibling paths drift. `Initialize` and `SetRelayers`, `Deposit` and
`Unlock`, `Mint` and `Burn` are written at different times, often by
copy-paste, and then only one of them gets the fix. The audit
technique that catches this class is boring and effective: build a
table — instructions as rows, validations as columns — and require
an explanation for every asymmetry. Some asymmetries are design
(a kill switch that blocks mints but not burns is an intentional
exit path). The ones nobody can explain are findings.

## Checklist for your program

- [ ] List every pair of paths that write the same state (create /
      update, deposit / withdraw, mint / burn).
- [ ] For each validation on either path, is it present — or
      explicitly, documentedly absent — on the other?
- [ ] Is the validation implemented **once**, in a shared function,
      rather than copied? Copies drift; calls don't.
- [ ] Does a test initialize with the malformed input your setter
      rejects, and assert the initializer rejects it too?
