# 02 — Pre-funded PDA griefing: 1 lamport, permanently blocked

**Family:** liveness / griefing · **Found in the audit:** Medium ·
**Components:** all three programs (bridge × 2, launch standard)

## The bug shape

The standard helper for creating a program-owned PDA:

```rust
// Vulnerable shape — the helper almost everyone writes
invoke_signed(
    &system_instruction::create_account(
        payer.key, target.key, rent_lamports, space as u64, program_id,
    ),
    accounts,
    &[seeds_with_bump],
)?;
```

`create_account` **fails if the target account already holds any
lamports**. And every PDA your program will ever create is a
deterministic address that anyone can compute from your public seeds
— and fund with a plain transfer, no signature from you, no
interaction with your program.

So an attacker can pre-fund, with a single lamport:

- **A bridge nonce PDA** for (direction, nonce). Nonces are
  sequential and visible in source-chain events, so the attacker
  knows exactly which destination PDA an in-flight crossing needs.
  Funding it makes the destination `Mint`/`Unlock` fail *forever* —
  while the source-side funds are already locked or burned. The
  crossing can never complete, and the protocol has no recovery
  path. The value is simply gone.
- **The receipt PDA** written by the same handlers — same crossing,
  same result.
- **The config PDA**, before deployment — `Initialize` can never
  run for that program id.
- **A launch standard's record / vault / reputation PDAs** —
  registration for that mint or that creator is permanently blocked.

Cost to the griefer: one tiny transfer per target. Cost to you: a
bricked crossing with real funds already committed on the other
chain.

## How the PoCs proved it

Two program-tests. (1) A fully valid, attested unlock was attempted
against a nonce PDA pre-funded with lamports: it failed; the test
warped forward and retried: it failed again; the vault balance was
untouched. (2) A launch registration against a pre-funded record
PDA: registration failed. Both confirmed the block is permanent, not
a transient error.

## The fix: create-or-adopt

Make account creation tolerant of a pre-funded shell. If the target
already holds lamports but is still a system-owned, zero-length
account — exactly what a plain transfer leaves behind — adopt it:
top it up to rent-exempt if needed, then `allocate` + `assign` with
the PDA as signer.

```rust
pub fn create_pda_account(
    payer: &AccountInfo,
    target: &AccountInfo,
    space: usize,
    owner: &Pubkey,
    seeds: &[&[u8]],
    bump: u8,
) -> ProgramResult {
    let rent = Rent::get()?;
    let required = rent.minimum_balance(space);
    let signer_seeds: &[&[u8]] = /* seeds + &[bump] */;

    if target.lamports() == 0 {
        // Fresh address: the normal path.
        invoke_signed(
            &system_instruction::create_account(
                payer.key, target.key, required, space as u64, owner,
            ),
            accounts, &[signer_seeds],
        )?;
    } else if target.owner == &system_program::id() && target.data_is_empty() {
        // Pre-funded shell: adopt it.
        if target.lamports() < required {
            invoke(
                &system_instruction::transfer(
                    payer.key, target.key, required - target.lamports(),
                ),
                accounts,
            )?;
        }
        invoke_signed(
            &system_instruction::allocate(target.key, space as u64),
            accounts, &[signer_seeds],
        )?;
        invoke_signed(
            &system_instruction::assign(target.key, owner),
            accounts, &[signer_seeds],
        )?;
    } else {
        // Already a real account of some other shape: not ours to take.
        return Err(ProgramError::AccountAlreadyInitialized);
    }
    Ok(())
}
```

The third branch matters: adopt only the exact shape a bare transfer
produces (system-owned, no data). Anything else — wrong owner, data
already present — is a genuine conflict and must still fail.

Apply the fix in the shared helper, everywhere: if your programs
each carry their own copy of `create_pda_account` (the audited ones
did — three copies), all three need the same change, and a test per
program, because the copies *will* drift (see
[03](03-duplicate-member-set.md) on sibling drift).

## Checklist for your program

- [ ] List every PDA your program creates. For each one: what
      happens if it already holds 1 lamport when your instruction
      runs?
- [ ] Are any of those PDAs needed *after* value has moved
      elsewhere (crossing receipts, escrow records)? Those are the
      permanent-loss cases.
- [ ] Does your creation helper handle the pre-funded shell
      (system-owned, empty) by adopting it — and reject every other
      pre-existing shape?
- [ ] Is there a test that pre-funds the target and asserts the
      instruction still succeeds?
