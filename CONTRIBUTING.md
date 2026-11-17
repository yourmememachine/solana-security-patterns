# Contributing

This guide grows by proof, not by opinion. A pattern earns its page
one of two ways:

1. **A runnable PoC** — a test, against a real program (yours, or a
   purpose-built minimal one), that demonstrates the vulnerable
   behavior: the stranger initializes the config, the pre-funded
   PDA blocks the instruction, the 5.26% lock passes the 10% gate.
2. **A killed-hypothesis write-up** — you were sure something was
   exploitable, you built the test, and the test proved the defense
   instead. These are first-class contributions: say what defended
   it, precisely, and what would remove the defense. Pattern
   [05](patterns/05-token-program-pinning.md) is the model.

## Rules

- **Prove it or label it.** Anything not backed by a PoC or a
  disproof gets marked *unverified lead* in the text, with what's
  missing stated plainly. No exceptions, no "reportedlys."
- **Honest severity.** Rate by demonstrated impact, not by the
  scariest shape the code suggests. A deployment-window High and a
  steady-state High are different claims; say which you mean.
- **Generalize the code, keep the specifics honest.** Excerpts in a
  pattern page should be short and teach the shape of the bug and
  the fix. The audit context (which program, which handler, what
  the PoC asserted) should be accurate — don't inflate a finding's
  reach to make the page more dramatic.
- **No hype.** No "critical vulnerability in Solana" framing for a
  bug in one program's helper. This guide's credibility is the
  product.
- **Fixes included.** A pattern page without the fix — and a
  checklist a reader can run against their own program — is half a
  contribution.

## Process

Open an issue describing the pattern and your evidence, or a PR
adding `patterns/NN-your-pattern.md` plus an index row in
`README.md` and the relevant family section in `PATTERNS.md`.
Review will ask one question first: *where's the test?*
