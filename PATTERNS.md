# The checklist

Every family below was exercised against a real bridge (two programs +
shared library + relayer) and a launch standard, instruction by
instruction. Each entry states the pattern, what to check, and — where
the audit reached a verdict — what it found. Patterns marked
**checked, cleared** failed to produce an attack *in that codebase*;
that is evidence about that code, not a promise about yours.

Deep-dives for the six patterns that produced a finding, a lead, or a
lesson worth a page: [patterns/](patterns/).

---

## 1. Account confusion & PDAs

The classic Solana family: a program trusts an account to be something
it never verified.

- **Unpinned account identity.** For every account a handler reads
  state from, check all three: `owner == your program`, the
  discriminator/type byte, and — where exactly one such account should
  exist — the PDA key itself. *Audit: config accounts were checked on
  owner + discriminator everywhere, key-pinned on the value paths;
  cleared.*
- **Unpacked state trusted without binding checks.** Parsing a token
  account's bytes tells you what the bytes say, not that the account is
  the right one. Bind parsed fields to the identities that matter:
  `state.owner == signer`, `state.mint == config mint`, vault key
  pinned in config. *Audit: cleared — every unpack result was bound.*
- **Substitute program in a CPI slot.** If a handler takes a
  `token_program` account and never compares its key, can an attacker
  pass a no-op program and fake the transfer? → Deep-dive
  [05](patterns/05-token-program-pinning.md). *Audit: hypothesis
  killed — but the defense lived inside a dependency. Pin it yourself.*
- **Account aliasing.** The same account passed in two roles (user
  account *and* vault, destination *and* source). Check whether your
  per-role validations jointly exclude it. *Audit: cleared — the
  owner checks on each role contradict the alias.*
- **PDA re-derivation.** Bumps taken from `find_program_address` at
  use time, seed tuples carrying every identity dimension (direction +
  nonce; mint; creator). A stored bump that disagrees with the derived
  one is a bug even if today's seeds agree. *Audit: cleared.*
- **`remaining_accounts` smuggling.** Untyped trailing accounts are an
  attack surface with no compiler help. *Audit: not present — the
  programs take no `remaining_accounts` at all. Consider whether you
  can say the same.*
- **Closure and revival.** Closed accounts can be re-funded and
  resurrected with stale state in the same transaction window.
  *Audit: not present — no account is ever closed.*

## 2. Signer & authority

- **Key equality instead of `is_signer`.** Comparing a passed pubkey
  to a stored authority proves nothing — anyone can pass anyone's
  pubkey. Collect authorities by filtering the account list on the
  runtime `is_signer` flag, then compare. *Audit: cleared — signer
  collection was `is_signer`-filtered on every admin path.*
- **Threshold counting without dedupe.** An m-of-n check that counts
  the same signer twice is an (m−1)-of-n check. Dedupe by identity
  (or by registered-set position) before counting. *Audit: cleared on
  the attestation path; the related set-*storage* bug is pattern
  [03](patterns/03-duplicate-member-set.md).*
- **Authority that can never rotate.** If admins are fixed at
  initialization with no setter, say so in the docs as a design
  decision — an auditor will find it anyway, and "deliberate" reads
  better than "missing". *Audit: noted as a design decision, not a
  finding.*

## 3. Attestation & the instructions sysvar

The Wormhole class: forged cross-chain messages.

- **Sysvar account not key-checked.** The instructions sysvar must be
  compared against `sysvar::instructions::ID` before a single byte is
  parsed. *Audit: cleared.*
- **Unbounded sysvar parsing.** Upstream convenience helpers have
  panicked on malformed sysvar data. Parse with every read
  bounds-checked; malformed input must be an error, never a panic.
  *Audit: cleared — the parser is hand-rolled and fuzzed; the panic
  this guards against was found by the project's own fuzzing earlier.*
- **Cross-instruction signature references.** An ed25519 precompile
  entry whose offsets point into a *different* instruction lets an
  attacker borrow signatures. Accept only self-referential entries.
  *Audit: cleared.*
- **Counting future instructions.** Only attestation entries in
  instructions *before* the current one may count — later ones haven't
  been constrained by your handler's position in the transaction.
  *Audit: cleared.*
- **Message-hash binding.** The signed bytes must equal a hash of
  *every* field of the message your program acts on — amount,
  recipient, nonce, direction, both chains. One unbound field is a
  field an attacker can rewrite. *Audit: cleared — all 11 fields
  bound, each mutation tested.*
- **Replay.** Direction-bound nonce accounts plus an explicit
  "unprocessed" check (owner == program ⇒ already processed) on every
  value path, not just the happy one. *Audit: cleared on all four
  crossing paths.*

## 4. Arithmetic & caps

- **Unchecked math on value paths.** Every add/sub on balances,
  totals, and accumulators should be `checked_*`. Search your value
  paths for `as` casts and `saturating_*` — both hide exactly the
  failures you care about. *Audit: cleared.*
- **Rounding direction.** A minimum that protects the protocol must
  round **up**; floor division silently shaves it. → Deep-dive
  [04](patterns/04-rounding-direction.md). *Audit: **found** in the
  launch standard's minimum-lock check.*
- **Percentage caps without truncation.** Compare by
  cross-multiplication in a wider type (`amount * 10_000 >
  reference * bps` in u128), never by dividing first. *Audit:
  cleared.*
- **Cap windows that can be held open.** If a tightened-cap window
  relaxes by slot arithmetic, check that no instruction can extend
  or re-arm it, and that relaxation is automatic. *Audit: cleared.*
- **Exactly-at-cap is a policy, write it down.** Strict `>` vs `>=`
  decides whether the cap value itself passes. Either is defensible;
  inconsistency between paths is not. *Audit: cleared — strict `>`,
  applied uniformly.*

## 5. Sibling-path asymmetry

Programs rarely have one path. They have Deposit *and* Unlock, Mint
*and* Burn, `Initialize` *and* `SetRelayers` — and the paths drift.

- **Validation present in the setter, absent in the initializer.**
  Whatever your update path validates, your creation path must
  validate too — creation usually runs first and runs once, so its
  gaps are permanent. → Deep-dive
  [03](patterns/03-duplicate-member-set.md). *Audit: **found**.*
- **Gate checked on some value paths.** Pause, kill-switch, and
  replay checks must be enumerated per path, not assumed. Build the
  table: rows are instructions, columns are gates, and explain every
  empty cell (some are deliberate — a kill switch that blocks mints
  but not burns is an exit path, and that's a design, not a gap).
  *Audit: cleared except pattern 03.*

## 6. Deployment window

The minutes between "program deployed" and "program initialized" are
part of your attack surface, and the easiest to forget because your
tests always initialize politely.

- **Permissionless `Initialize`.** If creating your config requires
  only *a* signature — anyone's — the first caller owns your
  relayers, admins, and caps. → Deep-dive
  [01](patterns/01-permissionless-initialize.md). *Audit: **found
  (High)**.*
- **Initialization parameters taken on faith.** A mint, vault, or
  oracle address read from instruction data with no account
  validation lets the initializer point your program at an asset
  they control. Validate every account you store: owner, type,
  parsed state. *Audit: found as part of pattern 01.*
- **No post-init verification step.** Even with a bound initializer,
  the deployment runbook should verify the *contents* of the stored
  config (sets, caps, mint, vault) before anything is announced or
  funded. A front-run you detect is an inconvenience; one you fund
  is a loss. *Audit: added to the deployment gate.*

## 7. Liveness & griefing

Not every attack steals. Some just stop the machine, cheaply and
permanently.

- **Pre-funded PDA blocks creation.** `system_instruction::create_account`
  fails if the target already holds lamports — and every PDA your
  program will ever create is computable by anyone, today. One
  lamport, sent in advance, can brick a crossing, a registration, or
  the deployment itself. → Deep-dive
  [02](patterns/02-prefunded-pda-griefing.md). *Audit: **found
  (Medium)**.*
- **Deterministic means public.** Nonces are sequential and emitted
  in source-chain events; PDA seeds are in your source. Assume the
  attacker has pre-computed every address you will ever use and
  plan creation accordingly (create-or-adopt, pattern 02).
- **No recovery path.** Ask of every multi-step flow: if step two
  can be blocked forever after step one has moved value, where does
  the value go? If the answer is "nowhere," the block *is* the
  attack.

## 8. Off-chain relayer trust

A relayer's signature is a value transfer with extra steps. Audit it
like one.

- **Signing without verifying the source event.** A relayer that
  signs because a record *exists*, without verifying the backing
  transfer actually happened (right amount, right destination or
  mint, finalized), will eventually sign for a phantom deposit.
  *Audit: cleared — the watcher verifies the source transfer before
  signing and refuses otherwise.*
- **Check not bound to its record.** Verifying "a matching transfer
  exists somewhere in this transaction" is not verifying "*this*
  record's transfer." → Deep-dive
  [06](patterns/06-relayer-source-binding.md). *Audit: hardening
  lead — safe today only because of an on-chain invariant.*
- **Finality shortcuts.** Sign only at finalized commitment, plus a
  per-direction safety margin; derive message fields (like expiry)
  deterministically from the source slot so honest relayers build
  byte-identical messages and disagreement fails closed instead of
  forking the quorum. *Audit: cleared.*
- **Collector weaker than the chain.** The off-chain quorum
  collector should re-verify with the *same* code the on-chain
  program uses, enforce one signature per member, and require full
  message equality across submissions. *Audit: cleared.*
- **Key hygiene.** Public-half consistency checked at load, no
  secret material in logs (grep your own crate and prove it),
  submission dry-run unless explicitly enabled. *Audit: cleared.*
