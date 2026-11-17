# Solana Security Patterns

A field guide to the patterns that break Solana programs — drawn from a
full self-audit of a real bridge and launch standard, where every pattern
was either **proven with a runnable proof-of-concept** or **killed with
evidence**. No pattern here is included on vibes.

The audited code is [Machine](https://github.com/yourmememachine/mememachine):
two bridge programs (`solana-lock`, `machine-bridge`), a shared library
(`bridge-common`), an off-chain relayer, and LaunchGuard, a launch
standard that custodies locked creator tokens. The audit swept the
programs instruction by instruction — every handler checked across
owner / discriminator / PDA key / signer / mint / authority — and took
every candidate lead to a test that either confirmed it or disproved it.

## What the audit found

| Result | Count |
|---|---|
| Confirmed findings | 4 — one High (deployment window), one Medium, two Low |
| Hardening leads | 1 — not exploitable today, one invariant away from mattering |
| Hypotheses killed with evidence | 6 families checked and cleared, including one initially-Critical idea that turned out to be defended by a dependency's check |

All four findings have since been fixed in Machine. The deep-dives below
describe each bug as it was found and the fix as it shipped.

## Pattern index

| # | Pattern | Family | Audit verdict |
|---|---|---|---|
| [01](patterns/01-permissionless-initialize.md) | Permissionless `Initialize` — first caller seizes the config | Deployment window | **Found (High)** |
| [02](patterns/02-prefunded-pda-griefing.md) | Pre-funded PDA griefing — 1 lamport permanently blocks a crossing | Liveness / griefing | **Found (Medium)** |
| [03](patterns/03-duplicate-member-set.md) | `Initialize` skips the validation its sibling enforces | Sibling-path asymmetry | **Found (Low)** |
| [04](patterns/04-rounding-direction.md) | Floor division quietly breaks a "≥10% locked" guarantee | Arithmetic | **Found (Low)** |
| [05](patterns/05-token-program-pinning.md) | Safety inherited from a dependency's internals | Account confusion | **Hypothesis killed** — defense verified, pin anyway |
| [06](patterns/06-relayer-source-binding.md) | Off-chain check not bound to the record it verifies | Relayer trust | **Hardening lead** |

The full sweep checklist — every pattern family checked, including the
ones that came back clean — is in [PATTERNS.md](PATTERNS.md).

## How to use this guide

- **Before an audit:** run the [PATTERNS.md](PATTERNS.md) checklist
  against your own instruction table. The findings here were not exotic;
  they were ordinary code that nobody had attacked on paper first.
- **Before deployment:** patterns 01 and 02 are deployment-window and
  liveness bugs. They cost nothing in steady state and everything on
  day one. Read those first.
- **Writing a relayer or any off-chain signer:** pattern 06 is the one
  that generalizes. An off-chain check that isn't bound to the exact
  record it verifies is a signature waiting for the wrong transaction.

## Honest framing

This is a **self-audit**: the same project auditing its own code, with
its own tooling. That is worth doing — it found real bugs — and it is
not a substitute for an independent audit. Machine remains **unaudited**
by any third party, and nothing in this guide changes that. Severity
ratings are the auditor's own, assigned by the same standard the
project applies to itself in its public docs: prove it with a test or
don't claim it.

Method, briefly: check-table first (every instruction × every account
dimension), divergent entries read line by line, every lead taken to a
runnable PoC or killed with evidence. The cleared sections matter as
much as the findings — knowing *why* an attack fails is what lets you
keep it failing after your next refactor.

## Contributing

New patterns are welcome — with proof. See [CONTRIBUTING.md](CONTRIBUTING.md).

## License

Apache-2.0. See [LICENSE](LICENSE).
