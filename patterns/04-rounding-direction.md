# 04 — Rounding direction: the 10% guarantee that was 5.26%

**Family:** arithmetic · **Found in the audit:** Low ·
**Component:** LaunchGuard (launch standard)

## The bug shape

LaunchGuard's headline guarantee: a registered launch locks **≥10%
of supply** for ≥30 days. The registration check computed the
minimum with integer floor division:

```rust
// Vulnerable shape
let min_lock = total_supply * MIN_LOCK_BPS / 10_000;   // floor division
if lock_amount < min_lock { return Err(LockTooSmall.into()); }
```

Floor division rounds the *requirement down*. For a supply of
1,000,000,000 the error is invisible. For small supplies it eats
the guarantee:

| Supply | `supply × 1000 / 10_000` (floor) | Actual % locked at the minimum |
|---|---|---|
| 100 | 10 | 10.0% |
| 19 | **1** | **5.26%** |
| 11 | **1** | **9.09%** |
| 10 | 1 | 10.0% |

At supply 19, locking **1 token of 19** passed a gate documented as
"≥10% of supply locked." The relative shortfall grows as supply
shrinks; at supplies ≤ 9 the computed minimum hits 0 and only a
separate zero-lock rejection kept the gate from vanishing entirely.

Is a 19-token launch economically meaningful? No. Does that matter?
Also no — for two reasons. First, the invariant as documented was
literally false, and a standard whose headline number is false at
the edges will be quoted at the edges. Second, rounding direction
is a habit, not a spot fix: the same floor-division instinct applied
to a fee, a cap, or a collateral ratio at real scale is a real
loss. Auditors flag the direction precisely because it reveals
which way the code leans everywhere.

## How the PoC proved it

Create a mint with both authorities revoked and total supply 19.
Register with `lock_amount = 1`. Registration **succeeded**; the
stored record showed `locked_amount × 10_000 / total_supply = 526`
basis points — under the 1,000 bps the standard promises.

## The fix: minimums that protect the protocol round up

Ceiling division, with checked arithmetic (use a u128 intermediate
so `supply × bps` can't overflow on the way):

```rust
pub fn min_lock_for_supply(total_supply: u64) -> Option<u64> {
    // ceil(supply * MIN_LOCK_BPS / 10_000)
    let numerator = (total_supply as u128)
        .checked_mul(MIN_LOCK_BPS as u128)?
        .checked_add(9_999)?;
    u64::try_from(numerator / 10_000).ok()
}
```

Now supply 19 requires 2 (10.5% ≥ 10%), supply 11 requires 2, and
the guarantee holds at every supply. Keep the explicit zero-lock
rejection as well — a computed minimum of 0 should never be
reachable, and belt-and-braces on a value gate is fine.

The rule, stated generally:

> **When a computed threshold protects the protocol or the user —
> a minimum lock, a minimum collateral, a fee owed *to* the
> protocol — round it up. When a computed amount is what the
> protocol *pays out*, the direction question deserves the same
> explicit thought. What you may not do is let the direction be an
> accident of `/`.**

## Checklist for your program

- [ ] Grep for `/` on value paths. For each division, write down
      which direction it rounds and who benefits from the error.
- [ ] For every documented guarantee ("≥ X%", "at most Y"), test
      it at the smallest inputs where it should hold — 19, 11, 10,
      1 — not just at realistic scale.
- [ ] Are percentage comparisons done by cross-multiplication in a
      wider type (`amount × 10_000 > reference × bps`) instead of
      dividing first and comparing the truncated result?
- [ ] Is the zero case handled by an explicit check, not by the
      arithmetic happening to avoid it?
