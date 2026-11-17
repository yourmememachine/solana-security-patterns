# 01 — Permissionless `Initialize`: first caller seizes the config

**Family:** deployment window · **Found in the audit:** High (deployment
window) · **Components:** both bridge programs

## The bug shape

Most Solana programs create their global config with an `Initialize`
instruction. The vulnerable shape is one where `Initialize` checks
only that *someone* signed:

```rust
// Vulnerable shape — generalized
fn process_initialize(accounts: &[AccountInfo], params: InitParams) -> ProgramResult {
    let payer = next_account_info(accounts)?;
    if !payer.is_signer {
        return Err(ProgramError::MissingRequiredSignature);
    }
    // ...create the config PDA and store params.relayers,
    //    params.admins, params.caps exactly as supplied.
}
```

Nothing binds the caller to the deployer. Between `solana program
deploy` and your `Initialize` transaction landing, the program is a
loaded gun with the safety off: **whoever calls `Initialize` first
sets the relayer set, the admin multisig, and the caps.** On one side
of the audited bridge, the initializer also chose the vault token
account — and because SPL `initialize_account` records an arbitrary
owner without that owner's signature, an attacker can mint themselves
a perfectly "genuine" vault. On the other side it was worse: the
wrapped mint was taken from instruction data with no account
validation at all, so the initializer could point the bridge at a
mint whose mint authority they personally held.

Your own `Initialize` then fails with `AccountAlreadyInitialized`.
That is detectable — but only if your deployment process verifies the
*contents* of the stored config before announcing or funding
anything. If it doesn't, the attacker's relayers attest arbitrary
crossings and their admins control pause and caps, and you built it
for them.

## How the PoC proved it

A program-test with a brand-new keypair — no relationship to the
deployer — called `Initialize` with attacker-chosen relayers, admins,
and caps. The test asserted the stored config equalled the attacker's
sets. It did, because nothing in the handler objected.

## The fix

Bind `Initialize` to the program's **upgrade authority**. Pass the
program's ProgramData account and require the payer to be the
authority recorded in it:

```rust
// ProgramData account data (BPF upgradeable loader, bincode layout):
//   [0..4]   variant tag (2 = ProgramData)
//   [4..12]  slot the program was last extended at (u64 LE)
//   [12]     option tag: 1 = upgrade authority present
//   [13..45] upgrade authority pubkey
pub fn verify_upgrade_authority(
    programdata: &AccountInfo,
    program_id: &Pubkey,
    payer: &AccountInfo,
) -> ProgramResult {
    let (expected, _) = Pubkey::find_program_address(
        &[program_id.as_ref()],
        &bpf_loader_upgradeable::id(),
    );
    if programdata.key != &expected || programdata.owner != &bpf_loader_upgradeable::id() {
        return Err(BridgeError::InvalidProgramData.into());
    }
    let data = programdata.try_borrow_data()?;
    if data.len() < 45 || data[12] != 1 {
        return Err(BridgeError::Unauthorized.into()); // no authority set
    }
    let authority = Pubkey::new_from_array(data[13..45].try_into().unwrap());
    if authority != *payer.key || !payer.is_signer {
        return Err(BridgeError::Unauthorized.into());
    }
    Ok(())
}
```

Notes on the fix as shipped:

- Deriving the expected ProgramData address matters. Accepting "a"
  programdata-shaped account lets an attacker fabricate one naming
  themselves.
- If the program's upgrade authority has been set to `None`
  (immutable program), `Initialize` can never run — correct: an
  immutable program you didn't initialize at deploy time is not
  yours to claim later, either.
- Validate **every** account you store at init, not just the
  caller's identity. The wrapped mint is now checked as a real mint
  account (token-program owner + parsed mint state) before it is
  written into config.
- Keep the operational backstop anyway: post-init config-content
  verification is an explicit step in the deployment gate. Defense
  in depth is cheap at deployment time, when you're already being
  careful.

## Checklist for your program

- [ ] Does `Initialize` require a specific identity — upgrade
      authority or a deployer key committed at build time — or just
      *a* signer?
- [ ] Is the ProgramData (or deployer-proof) account itself
      validated: derived address, loader owner, authority present?
- [ ] Is every stored parameter account (mint, vault, oracle)
      validated at init — owner, type, parsed state — rather than
      copied from instruction data?
- [ ] Can a stranger's `Initialize` be tested to fail? Write that
      test. If a fresh keypair *can* initialize your program in a
      test, it can on mainnet, first.
- [ ] Does your deployment runbook verify the stored config's
      contents before funding or announcing?
