# 06 — The relayer check that wasn't bound to its record

**Family:** off-chain relayer trust · **Audit verdict:** hardening
lead (informational) — not reachable today, one invariant away ·
**Component:** bridge relayer

## The setup

A bridge relayer must not sign an attestation just because an event
record exists on the source chain. The audited relayer does the
right first thing: before signing, it fetches the transaction that
created the record and verifies the backing token movement — a
transfer of exactly the record's amount into the vault for a
deposit, a burn of the wrapped mint for a burn — and attributes the
event's sender from that transfer's authority. No matching transfer,
no signature. This is the check whose absence has drained real
bridges.

## The gap

The verification scanned the parsed transaction — top-level
instructions, then all inner instructions, flattened — and accepted
the **first** transfer matching (amount, destination-vault | mint).
It carried no binding to the specific record being verified, nor to
the inner-instruction group belonging to the instruction that
created that record.

Consequence, proven by a unit PoC: **one genuine transfer satisfies
the check for any number of same-amount records created in the same
transaction.** Two records, one transfer, two signatures.

## Why it is not exploitable today — stated precisely

A `Deposit`/`Burn` record cannot be created without its own
transfer executing in the same transaction: the on-chain programs
build their token CPIs through checked builders that reject
substitute token programs (see [05](05-token-program-pinning.md)),
and the real token programs enforce the transfer itself. Records
and backing transfers are therefore created in lockstep, one for
one, and the counts implicitly match.

Read that paragraph again, because it is the whole lesson: the
relayer's correctness rests on an **on-chain invariant it does not
check**. The day someone adds a batch-deposit instruction, refactors
a builder, or introduces any path where a record can exist without
its own transfer in the same transaction, the relayer starts
signing double credits — and nothing in the relayer's own code or
tests would object, because the assumption was never written down
in it.

## The fix

Bind the check to the record. Two sound shapes:

- **Group binding (what Machine shipped).** Locate the top-level
  instruction that created *this* record's PDA, and scan only that
  instruction's inner-instruction group for the backing transfer.
  No creating instruction, or no matching transfer inside its
  group → fail closed.
- **Count coverage.** Count the records created in the transaction
  that share the match profile (amount, destination | mint) and
  require the number of matching transfers to cover them — the
  *n*-th record needs an *n*-th distinct transfer.

Either way, the property to assert in tests is: *k records require
k transfers.* The audit's isolation PoC — one transfer, two
records — becomes the regression test that must now fail
verification on the second record.

## The general lesson for off-chain signers

Any off-chain component that signs, attests, or releases value on
the strength of on-chain evidence should be able to answer, in its
own code: **which exact evidence authorizes this exact action?**
"A matching thing exists somewhere in this transaction/block/log"
is a search, not a binding. Searches match the attacker's padding
as happily as the real event. Bind to the record, the instruction
group, the account — the specific thing — or write down, in the
signer's own repo, the external invariant you're trusting and a
test that screams if it changes.

## Checklist for your relayer / signer / keeper

- [ ] Does every signature require evidence bound to the specific
      record being signed — not just a matching event in the same
      transaction?
- [ ] Could one genuine source event satisfy the check for two
      records? Write the test that tries.
- [ ] Are message fields derived deterministically (e.g. expiry
      from source slot) so honest signers build byte-identical
      messages and disagreement fails closed?
- [ ] Is the off-chain verification at least as strong as the
      on-chain one — ideally the *same code*?
- [ ] Is finality enforced (finalized commitment + safety margin)
      before any signature is produced?
