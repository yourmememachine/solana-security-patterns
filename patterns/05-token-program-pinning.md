# 05 — The Critical that wasn't: safety inherited from a dependency

**Family:** account confusion · **Audit verdict:** hypothesis
**killed with evidence** — and the lesson is why it's in this guide
anyway · **Component:** both bridge programs

## The hypothesis

The bridge processors accept a `token_program` account in every
value-moving handler — and never compare its key to
`spl_token::ID`. The relayer's own code even commented on it: the
programs "check nothing about it beyond passing it to CPI."

The attack writes itself: pass a malicious no-op program as
`token_program`. The bridge builds a "transfer" CPI, the fake
program returns success without moving anything, and the bridge's
books record a deposit that never happened — or an unlock that never
paid out. Rated on shape alone: Critical.

## Why it failed — in two layers

The PoCs asserted the *secure* outcome, for both a mock substitute
program and Token-2022, on the deposit, unlock, and mint paths. All
passed: no record created, no nonce consumed, no balances moved —
and the honest transaction for the same message succeeded
afterwards. The defense:

1. **The builders check.** The programs construct token CPIs with
   spl-token's checked builders (`transfer`, `mint_to`, `burn`;
   spl-token 4.0.2). Those builders call
   `check_spl_token_program_account`, which returns
   `IncorrectProgramId` for **any** program id other than SPL Token
   and Token-2022 — before a CPI is even constructed. An arbitrary
   attacker program is never invoked.
2. **Token-2022 checks ownership.** Token-2022 passes layer 1, but
   its processor enforces account ownership. The bridge's vault,
   user, and destination accounts are classic-token accounts, so
   the CPI fails inside Token-2022 and the whole transaction rolls
   back atomically.

Hypothesis dead. The audit recorded it as killed, with the PoCs
kept as regression tests.

## So why is this a pattern in a security guide?

Because the safety was **invisible**. It rested on:

- a check inside a dependency's builder functions — behavior that a
  refactor to hand-built instructions, or a different builder
  version, would silently remove;
- account-ownership enforcement in a *second* program the bridge
  doesn't control;
- and nothing in the project's own test suite would have noticed
  either layer disappearing — until the PoCs written for this
  hypothesis were kept.

Meanwhile the project's launch standard pinned the token program
explicitly in its handlers and documented why. Two halves of one
codebase, two different answers to the same question — and the
half that *looked* vulnerable was safe by accident, while the half
that was explicit was safe on purpose. "On purpose" is the only
kind that survives maintenance.

## The fix (hygiene, shipped anyway)

Pin it yourself, in every handler that accepts the account:

```rust
if token_program.key != &spl_token::id() {
    return Err(ProgramError::IncorrectProgramId);
}
```

One comparison per handler. Now the invariant lives in *your* code,
your tests can assert it directly, and a dependency upgrade can
change many things but not this. Decide explicitly whether
Token-2022 is in or out for your program — "the builders happen to
allow it" is not a decision.

## Checklist for your program

- [ ] For every program-id account your handlers accept
      (`token_program`, system program, any oracle or AMM program),
      is the key compared in *your* code?
- [ ] Which of your security properties are actually a dependency's
      behavior? List them. For each: what breaks if that crate
      changes, and would your tests notice?
- [ ] When a scary hypothesis turns out to be defended, keep the
      PoC as a regression test asserting the secure outcome. The
      test is the tripwire that tells you the defense is gone.
