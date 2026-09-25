# Mutation Audit — Arcadia Finance — Lending Pool

**Repository**: [arcadia-finance/lending-v2](https://github.com/arcadia-finance/lending-v2)  
**Commit**: `def3c94`  
**Scope**: `src/LendingPool.sol` · 416 tests in the suite  
**Date**: 2026-09-25  
**Method**: Gambit (mutant generation) + Foundry (compile & test) + Claude (survivor assessment)

> Every finding below was reviewed by hand against the source. 4 severities were downgraded from the automated assessment; each says so in its entry.

---

## Summary

**Mutation score: 92.9%** — the test suite detected 510 of 549 behaviour-changing mutations.

| | Count |
|---|---:|
| Mutants generated | 549 |
| Killed by the suite | 500 |
| **Survived (undetected)** | **39** |
| Timed out (counted as killed) | 10 |

### Undefended invariants by severity

> Severity is the impact **if this behaviour broke and shipped unnoticed** — not a bug in the code as written. The code is correct today. The finding is that nothing in the test suite would catch it if it stopped being correct.

| Severity | Count |
|---|---:|
| CRITICAL | 6 |
| HIGH | 16 |
| MEDIUM | 6 |
| LOW | 2 |
| INFORMATIONAL | 9 |

**Least defended**: CRITICAL — Deleted _settleLiquidationHappyFlow call in auctionRepay early-termination (`src/LendingPool.sol:529`)

---

## Undefended invariants

Each entry is a change to your contracts that the suite does not detect. The contract is correct as written; what follows is the test that is missing.

### 1. [CRITICAL] Deleted _settleLiquidationHappyFlow call in auctionRepay early-termination

**Location**: `src/LendingPool.sol:529` in `auctionRepay()`  
**Type**: Logic Error / Broken Invariant (missing state update)  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 129)

**What the code is supposed to enforce**

When a bidder's repayment during an auction fully covers the outstanding debt plus surplus, the original code calls _settleLiquidationHappyFlow to (1) pay out the termination reward to the terminator, (2) sync the liquidation penalty to the junior tranche/treasury, (3) credit any surplus to the account owner, (4) correctly bump totalRealisedLiquidity by the sum of these amounts, and (5) call _endLiquidation() which decrements auctionsInProgress and unlocks the most junior tranche.

**What would go wrong if it stopped doing so**

The mutant replaces the entire settlement call with a no-op `assert(true)`. As a result: (1) auctionsInProgress is never decremented for this liquidation, permanently blocking addTranche, setLiquidationParameters and skim (all guarded by `if (auctionsInProgress > 0) revert AuctionOngoing()`), effectively bricking core admin/treasury functionality for the whole pool after a single early-terminated auction; (2) the most junior tranche's `setAuctionInProgress(false)` hook is never called, so ITranche remains locked, preventing all deposits/withdrawals in that tranche indefinitely; (3) terminationReward, liquidationPenalty and surplus are never credited to realisedLiquidityOf for the terminator, LPs/treasury, or the account owner, while the corresponding underlying assets were already pulled into the pool from the bidder — these funds become permanently untracked/unclaimable since skim() (the only recovery mechanism) is itself blocked by the stuck auctionsInProgress counter; (4) no AuctionFinished event is emitted, breaking off-chain accounting/monitoring.

**Attack scenario (hypothetical — requires the change above)**

1. A position under auction has accountDebt = 100. 2. A bidder submits `auctionRepay` with amount = 150 (overpaying to cover debt + take collateral). 3. `accountDebt <= amount` branch executes: earlyTerminate=true, but the mutated line does nothing instead of settling rewards/penalty/surplus and calling _endLiquidation(). 4. auctionsInProgress remains at its pre-liquidation count (never decremented), and the junior tranche remains locked via setAuctionInProgress(true) set during startLiquidation. 5. From this point on, `addTranche`, `setLiquidationParameters`, and `skim` permanently revert with AuctionOngoing for the entire pool (unless another auction happens to independently decrement the counter to 0 via the unhappy flow, which itself would then falsely report the counter incorrectly since it was never truly closed). 6. The 50 excess (surplus) plus termination reward plus liquidation penalty amounts collected from the bidder are never credited to any realisedLiquidityOf balance, so those funds are stuck in the contract, unclaimable by anyone, and totalRealisedLiquidity under-reports the pool's actual claimable liquidity — a permanent accounting break with real economic loss to terminator, LPs, treasury and the account owner.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -526,7 +526,8 @@
             // -> Terminate the auction and make the surplus available to the Account-Owner.
             earlyTerminate = true;
             unchecked {
-                _settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount - accountDebt));
+                /// DeleteExpressionMutation(`_settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount - accountDebt))` |==> `assert(true)`) of: `_settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount - accountDebt));`
+                assert(true);
             }
             amount = accountDebt;
         }
```

**Recommended test**

Add a unit test for `auctionRepay` in the early-termination branch (amount > accountDebt) that asserts: (a) auctionsInProgress decreases by exactly 1 after the call, (b) the junior tranche's `setAuctionInProgress(false)` is invoked when it is the last ongoing auction, (c) realisedLiquidityOf[terminator], realisedLiquidityOf[account owner] (surplus) and the junior tranche/treasury liquidation-fee balances increase by the expected reward/penalty/surplus amounts, (d) totalRealisedLiquidity increases by terminationReward + liquidationPenalty + surplus, and (e) the `AuctionFinished` event is emitted with correct parameters. This directly kills the mutant since `assert(true)` performs none of these state changes.

---

### 2. [CRITICAL] Bad debt not written off from tranche balances in unhappy liquidation flow

**Location**: `src/LendingPool.sol:1060` in `settleLiquidationUnhappyFlow()`  
**Type**: Broken Accounting Invariant / Logic Error  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 454)

**What the code is supposed to enforce**

When accrued debt exceeds available liquidation incentives (bad debt scenario), _processDefault(badDebt) is meant to write off the bad debt from the realisedLiquidityOf balances of tranches, starting with the most junior tranche, and lock/pop any tranche that is fully wiped out. This keeps the invariant that sum(realisedLiquidityOf[x]) == totalRealisedLiquidity (backed by actual underlying assets in the pool plus outstanding debt).

**What would go wrong if it stopped doing so**

The mutation removes the call to _processDefault(badDebt) entirely (replaced with a no-op assert(true)), while totalRealisedLiquidity is still decremented by badDebt. This breaks the core solvency invariant: total realised liquidity (the pool's global accounting of claimable assets) is reduced, but the per-tranche realisedLiquidityOf balances are left untouched at their pre-default (inflated) values. As a result, the sum of all realisedLiquidityOf balances across tranches/treasury will exceed totalRealisedLiquidity by exactly the badDebt amount. LPs of the tranche that should have absorbed the loss retain claim to funds that no longer exist (were never recovered from the defaulted Account), and can withdraw them via withdrawFromLendingPool, effectively draining assets that back other LPs' deposits. This is a direct path to protocol insolvency and fund loss for other stakeholders.

**Attack scenario (hypothetical — requires the change above)**

1. An Account's debt is liquidated but the auction proceeds are insufficient to cover both the pending liquidation incentives and the open debt (openDebt > terminationReward + liquidationPenalty), triggering the bad debt branch in settleLiquidationUnhappyFlow. This is a normal, not-uncommon scenario during undercollateralized liquidations (e.g., in a market crash). 2. The Liquidator calls settleLiquidationUnhappyFlow, which computes badDebt and reduces totalRealisedLiquidity by badDebt, but with the mutation, never reduces the junior tranche's realisedLiquidityOf balance. 3. LPs in the junior tranche now hold realisedLiquidityOf balances that overstate their actual claim (since the badDebt was never deducted from them). 4. Junior tranche LPs (or the tranche contract making pro-rata redemptions to its holders) call withdrawFromLendingPool and successfully withdraw underlying assets that should have been forfeited to cover the bad debt. 5. Because totalRealisedLiquidity was already reduced but the actual asset balance in the pool was not increased to compensate, later withdrawers (senior tranche LPs or treasury) may find the pool's underlying asset balance insufficient, effectively transferring the loss to them instead of the intended junior tranche, or causing reverts/insolvency for legitimate late withdrawals.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1057,7 +1057,8 @@
             // unsafe cast: uint128 - uint256 is always smaller than uint128.
             // forge-lint: disable-next-line(unsafe-typecast)
             totalRealisedLiquidity = uint128(totalRealisedLiquidity - badDebt);
-            _processDefault(badDebt);
+            /// DeleteExpressionMutation(`_processDefault(badDebt)` |==> `assert(true)`) of: `_processDefault(badDebt);`
+            assert(true);
             (terminationReward, liquidationPenalty) = (0, 0);
         } else {
             uint256 remainder = liquidationPenalty + terminationReward - openDebt;
```

**Recommended test**

Add a unit test for settleLiquidationUnhappyFlow that: (1) sets up an Account default scenario where openDebt > terminationReward + liquidationPenalty, (2) records realisedLiquidityOf of the most junior tranche before the call, (3) invokes settleLiquidationUnhappyFlow, and (4) asserts that realisedLiquidityOf[juniorTranche] decreased by exactly badDebt (or that the tranche was popped/locked if fully wiped out), and that sum(realisedLiquidityOf across all tranches+treasury) == totalRealisedLiquidity after the call. This directly kills the mutant since assert(true) leaves the tranche balance unchanged.

---

### 3. [CRITICAL] Deleted assignment causes maxBurnable to stay 0, wiping out all tranches on default

**Location**: `src/LendingPool.sol:1129` in `_processDefault()`  
**Type**: Logic Error / State Corruption leading to Fund Loss  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 470)

**What the code is supposed to enforce**

In `_processDefault`, `maxBurnable` must be assigned the current realised liquidity of the tranche being processed (`realisedLiquidityOf[tranche]`) at each loop iteration, so the function can correctly decide whether the tranche's balance is sufficient to absorb the remaining bad debt (partial write-off, loop break) or must be fully wiped out and popped (full write-off, continue to next tranche).

**What would go wrong if it stopped doing so**

With the assignment replaced by a no-op `assert(true)`, `maxBurnable` is never updated and remains at its default value of 0 for every iteration of the loop. Consequently the condition `badDebt < maxBurnable` (i.e. `badDebt < 0`) is always false for any non-zero bad debt, so the function never takes the 'partial write-off, break' branch. Instead, on every iteration it unconditionally executes the 'full wipe-out' branch: it sets `realisedLiquidityOf[tranche] = 0`, pops the tranche via `_popTranche`, decrements `badDebt` by `maxBurnable` (0, so badDebt never actually decreases), and locks the tranche. Because `badDebt` never decreases, the loop proceeds through every tranche from most junior to most senior, zeroing and popping ALL of them regardless of how small the actual bad debt was. This destroys the funds (accounting-wise) of every Liquidity Provider in every tranche, not just the amount needed to cover the bad debt, causing catastrophic and unjustified loss of LP claims and breaking the core solvency invariant of the protocol.

**Attack scenario (hypothetical — requires the change above)**

1. Under normal protocol operation, an Account becomes undercollateralized (e.g., due to market price movement) and is liquidated. 2. The liquidator calls `auctionRepay`/`settleLiquidationUnhappyFlow` and the auction proceeds are insufficient to cover `openDebt`, so `openDebt > terminationReward + liquidationPenalty`, triggering `_processDefault(badDebt)` with some (possibly very small) `badDebt`. 3. Inside `_processDefault`, due to the mutation, `maxBurnable` stays 0, so the condition `badDebt < maxBurnable` is always false. 4. The function enters the 'unhappy' branch for the most junior tranche: sets its `realisedLiquidityOf` to 0 (losing all its LP claims, even though it may have had far more liquidity than the badDebt), pops it, and (because `maxBurnable` is 0) subtracts nothing from `badDebt`. 5. The loop continues to the next tranche (now the new 'most junior') and repeats the same full wipe-out, and so on for every remaining tranche, because `badDebt` never gets reduced. 6. End result: every tranche's LP liquidity claim is zeroed and the tranches are popped/locked, even for a trivially small bad debt event. LPs across the entire pool lose their claimable liquidity while the underlying assets remain locked in the contract, creating a severe, unjustified insolvency and loss of funds.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1126,7 +1126,8 @@
                 --i;
             }
             tranche = tranches[i];
-            maxBurnable = realisedLiquidityOf[tranche];
+            /// DeleteExpressionMutation(`maxBurnable = realisedLiquidityOf[tranche]` |==> `assert(true)`) of: `maxBurnable = realisedLiquidityOf[tranche];`
+            assert(true);
             if (badDebt < maxBurnable) {
                 // Deduct badDebt from the balance of the most junior Tranche.
                 unchecked {
```

**Recommended test**

Add a unit test for `_processDefault` (invoked via `settleLiquidationUnhappyFlow`) that sets up at least two tranches with known non-zero `realisedLiquidityOf` balances, then triggers a bad debt amount strictly smaller than the most junior tranche's realised liquidity. Assert that only the most junior tranche's balance is decremented by exactly `badDebt`, that no tranche is popped/locked, and that all other tranches' balances are unchanged. This test fails on the mutant because `maxBurnable` never reflects the tranche's actual balance, causing the tranche to be fully wiped and popped instead of partially decremented.

---

### 4. [CRITICAL] Swapped comparison in _processDefault causes underflow & asset inflation

**Location**: `src/LendingPool.sol:1130` in `_processDefault()`  
**Type**: Integer Underflow / Logic Error leading to Fund Drain  
**Confidence**: HIGH  
**Mutation**: `SwapArgumentsOperatorMutation` (id 471)

**What the code is supposed to enforce**

In _processDefault, when badDebt is smaller than the realised liquidity of the most junior tranche (`badDebt < maxBurnable`), the code should simply deduct badDebt from that tranche's balance and stop (happy path). When badDebt is equal to or exceeds the tranche's balance, the tranche must be fully wiped out, popped, locked, and the remaining badDebt propagated to the next (more senior) tranche. This ordering ensures that only tranches with sufficient balance ever have `realisedLiquidityOf[tranche] -= badDebt` executed, since that subtraction is performed inside an `unchecked` block and relies on the invariant `badDebt < maxBurnable` to be safe.

**What would go wrong if it stopped doing so**

The mutation swaps the comparison to `maxBurnable < badDebt`, which inverts which branch executes. When the tranche's balance is smaller than the outstanding bad debt (the very case the original code intended to route into the 'wipe out tranche completely' branch), the mutant instead takes the 'deduct badDebt from balance' branch. Because this subtraction runs inside `unchecked { realisedLiquidityOf[tranche] -= badDebt; }`, subtracting a larger badDebt from a smaller maxBurnable silently underflows, wrapping `realisedLiquidityOf[tranche]` to a value near `2^256`. The loop then `break`s, so no further tranches absorb the remaining bad debt and the corrupted tranche is never popped/locked. The affected tranche's LPs now have an astronomically large claimable balance, entirely decoupled from the pool's real (limited) reserves, allowing them to drain all funds via `withdrawFromLendingPool`. Additionally, for the case where badDebt < maxBurnable (the normal happy path), the mutant now incorrectly wipes out the tranche entirely and cascades the debt to more senior tranches, which is a correctness/economic bug even absent the underflow, since some LPs lose more assets than justified while others in senior tranches are unfairly harmed.

**Attack scenario (hypothetical — requires the change above)**

1. Protocol accumulates a liquidation event whose `badDebt` (computed in `settleLiquidationUnhappyFlow`) exceeds the realised liquidity of the most junior tranche (a realistic 'unhappy flow' scenario the code explicitly anticipates in comments). 2. `_processDefault(badDebt)` is invoked. With the mutant, since `maxBurnable < badDebt` is true for the junior tranche, the code executes `realisedLiquidityOf[tranche] -= badDebt` inside an unchecked block, underflowing to a value close to `type(uint256).max`. 3. The loop breaks; the corrupted tranche's `realisedLiquidityOf` is never reset to zero, and no other tranche absorbs the bad debt. 4. Any LP with realisedLiquidityOf indexed at that tranche address (or the tranche contract itself, which owns the balance in the mapping) can call `withdrawFromLendingPool(assets, receiver)` requesting a very large `assets` amount (bounded only by the corrupted mapping value and available pool liquidity from other depositors). 5. The Lending Pool's underlying asset balance is transferred out via `asset.safeTransfer(receiver, assets)`, draining funds deposited by other, unrelated tranches/LPs, resulting in insolvency of the protocol.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1127,7 +1127,8 @@
             }
             tranche = tranches[i];
             maxBurnable = realisedLiquidityOf[tranche];
-            if (badDebt < maxBurnable) {
+            /// SwapArgumentsOperatorMutation(`badDebt < maxBurnable` |==> `maxBurnable < badDebt`) of: `if (badDebt < maxBurnable) {`
+            if (maxBurnable < badDebt) {
                 // Deduct badDebt from the balance of the most junior Tranche.
                 unchecked {
                     // forge-lint: disable-next-item(unsafe-typecast)
```

**Recommended test**

Add a unit test for `_processDefault`/`settleLiquidationUnhappyFlow` where `badDebt` is strictly greater than the realised liquidity of the most junior tranche (and ideally spans multiple tranches). Assert that: (a) the junior tranche's `realisedLiquidityOf` is set to exactly 0 (not underflowed), (b) the tranche is popped from `tranches` and `isTranche` is cleared, (c) `ITranche(tranche).lock()` is called, and (d) the remaining `badDebt - maxBurnable` is correctly propagated and deducted from the next tranche. Also add a boundary test where `badDebt == maxBurnable` to lock in the exact-equality branch behavior.

---

### 5. [CRITICAL] Bad debt write-off skipped for junior tranche, breaking accounting invariant

**Location**: `src/LendingPool.sol:1134` in `_processDefault()`  
**Type**: Logic Error / Accounting Invariant Violation  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 483)

**What the code is supposed to enforce**

In `_processDefault`, when bad debt is smaller than the most junior Tranche's realised liquidity, the bad debt must be deducted from that Tranche's `realisedLiquidityOf` balance so the junior Tranche's LPs absorb the loss proportional to the actual bad debt written off, keeping `sum(realisedLiquidityOf[x])` in sync with `totalRealisedLiquidity` (which was already decremented by `badDebt` in the caller `settleLiquidationUnhappyFlow`).

**What would go wrong if it stopped doing so**

The mutation replaces `realisedLiquidityOf[tranche] -= badDebt;` with a no-op (`assert(true)`). As a result, the junior Tranche's claimable balance is never reduced, even though: (1) `totalRealisedLiquidity` was already decremented by `badDebt` in `settleLiquidationUnhappyFlow` before calling `_processDefault`, and (2) the underlying debt asset backing that liquidity was genuinely lost (the borrower defaulted). This creates a permanent mismatch: `sum(realisedLiquidityOf[all_holders]) > totalRealisedLiquidity`. The junior Tranche keeps a claim on funds that no longer exist in the pool's real accounting, meaning the loss that should be socialized to junior LPs is instead silently absorbed by whichever other liquidity providers withdraw last (since `withdrawFromLendingPool` will eventually underflow/revert on `totalRealisedLiquidity -= assets` once the real backing runs out) — or by protocol insolvency if too many claims exceed real liquidity. This is not a rare edge case: it triggers on ANY unhappy-flow liquidation where bad debt is smaller than the junior tranche's balance, which is one of the two normal code branches in `_processDefault`.

**Attack scenario (hypothetical — requires the change above)**

1. Protocol has multiple tranches; junior tranche has realisedLiquidityOf = 1000 (backed 1:1 by real assets in totalRealisedLiquidity).
2. A borrower's Account becomes deeply undercollateralized and liquidation goes through the unhappy flow with, say, badDebt = 200 (less than the junior tranche's 1000 balance).
3. `settleLiquidationUnhappyFlow` decrements `totalRealisedLiquidity` by 200 and calls `_processDefault(200)`.
4. Due to the mutation, `_processDefault` fails to decrement `realisedLiquidityOf[juniorTranche]` — it remains 1000, even though total pool accounting now reflects 200 less liquidity.
5. Junior Tranche LPs, unaware of the loss, redeem their full (unreduced) shares via the Tranche's ERC4626 `withdraw`/`redeem`, which ultimately calls `withdrawFromLendingPool` for the full 1000 — draining real assets that are insufficient to also cover other LPs' legitimate claims.
6. Later withdrawals by senior tranche LPs or treasury will underflow/revert on `totalRealisedLiquidity -= assets` (since real backing was depleted by the junior tranche's unreduced withdrawal), causing a DoS on withdrawals and effectively transferring the bad debt loss to other stakeholders instead of the intended junior tranche — the protocol becomes insolvent relative to its own accounting.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1131,7 +1131,8 @@
                 // Deduct badDebt from the balance of the most junior Tranche.
                 unchecked {
                     // forge-lint: disable-next-item(unsafe-typecast)
-                    realisedLiquidityOf[tranche] -= badDebt;
+                    /// DeleteExpressionMutation(`realisedLiquidityOf[tranche] -= badDebt` |==> `assert(true)`) of: `realisedLiquidityOf[tranche] -= badDebt;`
+                    assert(true);
                 }
                 break;
             } else {
```

**Recommended test**

Add a test for `_processDefault`/`settleLiquidationUnhappyFlow` covering the branch where `badDebt < realisedLiquidityOf[mostJuniorTranche]`: assert that after the call, `realisedLiquidityOf[juniorTranche]` decreases by exactly `badDebt`, and add an invariant test asserting `sum(realisedLiquidityOf[all tranches] + treasury balance) == totalRealisedLiquidity` after any liquidation settlement (happy or unhappy flow) with nonzero bad debt.

---

### 6. [CRITICAL] Bad-debt write-off loop never executes due to inverted loop condition

**Location**: `src/LendingPool.sol:1124` in `_processDefault()`  
**Type**: Logic Error / Broken Invariant (Denial of Service / Insolvency)  
**Confidence**: HIGH  
**Mutation**: `SwapArgumentsOperatorMutation` (id 484)

**What the code is supposed to enforce**

The `for (uint256 i = length; i > 0;)` loop in `_processDefault` is meant to iterate from the most junior tranche to the most senior, deducting `badDebt` from each tranche's `realisedLiquidityOf` balance (and popping/locking fully wiped-out tranches) until all bad debt is absorbed. This keeps `totalRealisedLiquidity` consistent with the sum of individual `realisedLiquidityOf` balances after a liquidation results in unrecovered debt.

**What would go wrong if it stopped doing so**

Since `i` is a `uint256`, the mutated condition `0 > i` is never true (an unsigned integer can never be less than 0). As a result the entire for-loop body is skipped unconditionally, regardless of `length`. `_processDefault` becomes a silent no-op: no tranche's `realisedLiquidityOf` is decremented, no tranche is popped/locked when fully wiped out, and no `setAuctionInProgress` hook fires for wipe-out cascades. Meanwhile, the caller `settleLiquidationUnhappyFlow` has already decremented `totalRealisedLiquidity` by `badDebt` *before* calling `_processDefault`. This creates a permanent, unrecoverable mismatch: `totalRealisedLiquidity` (the pool's accounting of claimable liquidity) is reduced, but the sum of all individual `realisedLiquidityOf` mapping entries is not. The tranche that should have absorbed the loss keeps its full claimable balance, effectively socializing the bad debt onto the protocol's actual underlying asset reserves instead of the intended junior tranche. This is a core solvency-accounting invariant break: the pool now systematically over-promises liquidity relative to real backing, creating a race condition/bank-run risk where early withdrawers are made whole while later withdrawers face reverts (insufficient underlying balance) or under-collateralized tranches keep operating as if healthy.

**Attack scenario (hypothetical — requires the change above)**

1. An Account accumulates debt that becomes severely undercollateralized; a liquidation auction is initiated and proceeds are insufficient to cover both debt and incentives, so `openDebt > terminationReward + liquidationPenalty`.
2. The Liquidator calls `settleLiquidationUnhappyFlow(account, startDebt, minimumMargin_, terminator)`. Inside, `badDebt = openDebt - terminationReward - liquidationPenalty` is computed and `totalRealisedLiquidity` is decremented by `badDebt`.
3. `_processDefault(badDebt)` is called but the mutated loop condition prevents any iteration — the most junior tranche's `realisedLiquidityOf` is NOT reduced, and the tranche is not popped/locked even though its liquidity should have been wiped out.
4. Now `totalRealisedLiquidity` (global claimable liquidity) is understated relative to `sum(realisedLiquidityOf[tranche])`, i.e., the junior tranche LPs still believe they hold the full (unimpaired) balance.
5. Junior tranche LPs (or any LP who withdraws first) call `withdrawFromLendingPool`, draining real underlying assets from the pool based on their (unreduced) `realisedLiquidityOf` balance.
6. Because the actual underlying asset reserves were reduced by the bad debt event but the accounting wasn't correspondingly written off from the responsible tranche, the pool eventually cannot satisfy all legitimate withdrawal claims — later LPs (including senior tranches) experience reverted withdrawals or reduced redemption value, effectively realizing losses that should have been isolated to the junior tranche. This is a direct, protocol-wide insolvency/bank-run vector triggered automatically on the very first unhappy-flow liquidation.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1121,7 +1121,8 @@
         address tranche;
         uint256 maxBurnable;
         uint256 length = tranches.length;
-        for (uint256 i = length; i > 0;) {
+        /// SwapArgumentsOperatorMutation(`i > 0` |==> `0 > i`) of: `for (uint256 i = length; i > 0;) {`
+        for (uint256 i = length; 0 > i;) {
             unchecked {
                 --i;
             }
```

**Recommended test**

Add a unit test for `settleLiquidationUnhappyFlow` (or directly for `_processDefault`) that: (1) sets up multiple tranches with known `realisedLiquidityOf` balances, (2) triggers a liquidation where `openDebt > terminationReward + liquidationPenalty` producing a non-zero `badDebt`, and (3) asserts that after the call, the most junior tranche's `realisedLiquidityOf` has been reduced by `badDebt` (or fully zeroed and popped/locked if `badDebt >= trancheBalance`, with the next tranche partially absorbing the remainder). Also assert `tranches.length` decreases and `ITranche.lock()`/`setAuctionInProgress(true)` were called when a tranche is fully wiped out. This directly exercises the for-loop body and would fail under the mutant since no state changes occur.

---

### 7. [HIGH] totalRealisedLiquidity decremented instead of incremented in startLiquidation

**Location**: `src/LendingPool.sol:937` in `startLiquidation()`  
**Type**: Accounting/Arithmetic Error (Broken Invariant)  
**Confidence**: HIGH  
**Mutation**: `BinaryOpMutation` (id 364)

**What the code is supposed to enforce**

When a liquidation is initiated, the initiationReward is credited to the initiator's individual claimable balance (realisedLiquidityOf[initiator]) AND added to the global totalRealisedLiquidity tracker, keeping the invariant sum(realisedLiquidityOf) == totalRealisedLiquidity intact (minted debt tokens back this reward as new liability).

**What would go wrong if it stopped doing so**

The mutant changes `totalRealisedLiquidity += initiationReward` to `totalRealisedLiquidity -= initiationReward`, while the initiator's individual balance is still correctly incremented. This desynchronizes the sum of all realisedLiquidityOf balances from totalRealisedLiquidity by 2x initiationReward per liquidation. totalRealisedLiquidity is a core state variable used in: (1) _updateInterestRate()'s utilisation calculation (totalDebt/totalLiquidity), which will report inflated utilisation and thus overcharge all borrowers; (2) skim(), where `delta = balanceOf(this) + realisedDebt - totalRealisedLiquidity` will become artificially inflated, letting the treasury siphon real LP-owned liquidity as 'surplus'; and (3) totalLiquidity()/liquidityOf() views used by Tranches for share pricing, corrupting ERC4626 exchange rates. Additionally, because totalRealisedLiquidity is a uint128 and the subtraction is unchecked at the Solidity level (SafeCastLib.safeCastTo128 only checks upper bound, not underflow), once cumulative initiationRewards exceed totalRealisedLiquidity, the subtraction underflows and reverts, permanently bricking startLiquidation() and preventing any future liquidations — a protocol-wide DoS that allows bad debt to accumulate unchecked and threatens insolvency.

**Attack scenario (hypothetical — requires the change above)**

1. Any user with a healthy or slightly undercollateralized position exists, or an attacker/keeper triggers startLiquidation() repeatedly on unhealthy accounts (a normal, permissionless operation). 2. Each call credits realisedLiquidityOf[initiator] correctly but decrements totalRealisedLiquidity instead of incrementing it, growing a discrepancy of 2*initiationReward per call. 3. After several liquidations, an attacker (or the treasury owner) calls skim(): the artificially low totalRealisedLiquidity inflates `delta`, transferring real LP funds to realisedLiquidityOf[treasury] that are not backed by actual surplus assets. 4. LPs then cannot fully withdraw their real liquidity via withdrawFromLendingPool/Tranche redemptions, since totalRealisedLiquidity and per-tranche balances no longer reflect actual claimable assets, or eventually startLiquidation itself reverts on underflow, freezing the liquidation mechanism entirely while bad debt accrues unchecked, leading to insolvency of the pool.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -934,7 +934,8 @@
         // The other incentives will only be added as realised liquidity for the respective actors
         // after the auction is finished.
         realisedLiquidityOf[initiator] += initiationReward;
-        totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity + initiationReward);
+        /// BinaryOpMutation(`+` |==> `-`) of: `totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity + initiationReward);`
+        totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity-initiationReward);
 
         // If this is the sole ongoing auction, prevent any deposits and withdrawals in the most jr tranche
         if (auctionsInProgress == 0 && tranches.length > 0) {
```

**Recommended test**

Add a unit test for startLiquidation that asserts `totalRealisedLiquidity` increases by exactly `initiationReward` (not decreases), e.g. `assertEq(totalRealisedLiquidityAfter, totalRealisedLiquidityBefore + initiationReward)`. Additionally add an invariant/fuzz test that after any sequence of startLiquidation calls, `totalRealisedLiquidity == sum(realisedLiquidityOf[all tranches] + realisedLiquidityOf[treasury] + realisedLiquidityOf[initiator])` holds, which would immediately catch this sign-flip mutation.

> **Reviewer note**: Downgraded from CRITICAL on hand review. The accounting desync is real, but there is no direct theft.

---

### 8. [HIGH] Missing realisedDebt decrement in settleLiquidationUnhappyFlow

**Location**: `src/LendingPool.sol:1081` in `settleLiquidationUnhappyFlow()`  
**Type**: Logic Error / Accounting Invariant Violation  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 457)

**What the code is supposed to enforce**

After burning the account's debt shares and writing off the open debt during an unhappy liquidation flow, the code must decrement the pool-wide `realisedDebt` accumulator by `openDebt` to keep total realised debt in sync with the sum of individual account debts. This maintains the core invariant that `realisedDebt` reflects the actual outstanding debt tracked via debt tokens.

**What would go wrong if it stopped doing so**

By deleting `realisedDebt -= openDebt;` and replacing it with a no-op `assert(true)`, the pool's `realisedDebt` state variable is never reduced even though the account's debt shares were burned (`_burn(account, debtShares)`) and the debt was written off via `_processDefault(badDebt)` or partial repayment. This causes `realisedDebt` to permanently overstate actual outstanding debt by `openDebt` on every unhappy-flow liquidation. Since `totalAssets()` and `calcUnrealisedDebt()` derive unrealised interest from `realisedDebt`, this inflates the reported total debt of the pool, which cascades into interest rate calculations (`_updateInterestRate`), interest accrual to LPs/treasury (`_syncInterestsToLiquidityProviders`), and the overall LP claimable liquidity accounting (`totalLiquidity()`). Over repeated unhappy liquidations, `realisedDebt` drifts upward without bound relative to the real debt tokens outstanding (whose supply already reflects the burn), permanently corrupting the core debt/liquidity invariant. This can lead to LPs being paid interest on phantom debt that doesn't exist, treasury/tranche shares miscalculated, and eventually insolvency-style accounting bugs (total realised liquidity not matching real claims) since more interest keeps getting synthesized from an inflated `realisedDebt` base than the pool actually collects from borrowers.

**Attack scenario (hypothetical — requires the change above)**

1. Attacker (or any borrower) takes on debt and their Account becomes undercollateralized, triggering `startLiquidation`.
2. During the auction, insufficient proceeds are recovered, so `settleLiquidationUnhappyFlow` is invoked by the Liquidator with `openDebt > terminationReward + liquidationPenalty`, hitting the bad debt branch.
3. `_burn(account, debtShares)` removes the debt tokens (so the borrower's shares are gone) but `realisedDebt` is NOT decremented due to the mutation.
4. `realisedDebt` now permanently overstates the actual total debt (sum of all debt token balances). This inflated `realisedDebt` is used every block in `calcUnrealisedDebt()` (compounding on an inflated principal) and in `_updateInterestRate` for utilisation calculations.
5. Repeating this liquidation flow multiple times over the life of the pool causes `realisedDebt` to grow unboundedly disconnected from real debt token supply, artificially inflating interest paid out to LPs/treasury from `totalRealisedLiquidity`, which is funded by real underlying assets. Eventually LPs may be able to withdraw more assets than the pool actually holds (insolvency), since interest synced to `realisedLiquidityOf` mappings is computed from a phantom, ever-growing `realisedDebt` figure, while actual underlying asset reserves do not grow proportionally.
6. This is a systemic, protocol-wide accounting corruption exploitable simply by allowing normal unhappy liquidations to occur (no special attacker privileges needed) — over time it can be intentionally triggered by allowing/causing under-collateralized positions to go through the unhappy flow repeatedly.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1078,7 +1078,8 @@
 
         // Remove the remaining debt from the Account now that it is written off from the liquidation incentives/Liquidity Providers.
         _burn(account, debtShares);
-        realisedDebt -= openDebt;
+        /// DeleteExpressionMutation(`realisedDebt -= openDebt` |==> `assert(true)`) of: `realisedDebt -= openDebt;`
+        assert(true);
         emit Withdraw(msg.sender, account, account, openDebt, debtShares);
 
         _endLiquidation();
```

**Recommended test**

Add a unit test for `settleLiquidationUnhappyFlow` that asserts `realisedDebt` decreases by exactly `openDebt` after the call (e.g., capture `realisedDebt` before and after the call in both the bad-debt branch and the non-bad-debt branch, and assert `realisedDebtBefore - realisedDebtAfter == openDebt`). Additionally add an invariant/fuzz test verifying that after any liquidation flow, `realisedDebt` equals `convertToAssets(totalSupply)` of the debt token (i.e., debt token supply and realisedDebt stay in sync), which would fail immediately when this line is removed.

> **Reviewer note**: Downgraded from CRITICAL on hand review. Corrupts realisedDebt and propagates to interest rates, but is not direct fund loss.

---

### 9. [HIGH] Missing array pop desynchronizes tranches[] and interestWeightTranches[]

**Location**: `src/LendingPool.sol:279` in `_popTranche()`  
**Type**: Logic Error / State Corruption (broken invariant)  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 29)

**What the code is supposed to enforce**

`_popTranche` is meant to atomically remove a fully-defaulted (wiped-out) Tranche from all three parallel data structures: `isTranche`, `interestWeightTranches[]`, and `tranches[]`, so that index `i` in `tranches[]` always corresponds to the same index `i` in `interestWeightTranches[]`. This 1:1 index correspondence is relied upon everywhere interest and liquidation fees are distributed (`_syncInterestsToLiquidityProviders`, `setInterestWeightTranche`, `addTranche`).

**What would go wrong if it stopped doing so**

By deleting `interestWeightTranches.pop()` (replaced with a no-op `assert(true)`), the `tranches[]` array shrinks by one element on a full tranche write-off, but `interestWeightTranches[]` keeps its stale trailing entry (the weight of the just-removed tranche). The two arrays permanently desynchronize. When a new tranche is subsequently added via `addTranche`, the new interest weight is appended to the *end* of `interestWeightTranches[]`, while the new tranche address occupies the index vacated by the previous tranche in `tranches[]`. From that point on, `_syncInterestsToLiquidityProviders` (which loops `i < tranches.length` and reads `interestWeightTranches[i]`) will apply the *stale weight of the deleted tranche* to the newly-added tranche instead of its actual configured weight, permanently corrupting interest-fee distribution across tranches and the treasury. This is a silent, persistent accounting bug that misallocates real underlying-asset value between LPs of different tranches and the treasury, with no way to self-correct short of manually calling `setInterestWeightTranche` on every affected index.

**Attack scenario (hypothetical — requires the change above)**

1. Protocol has two tranches: A (senior, index 0) and B (junior, index 1) with weights W0 and W1.
2. A default occurs large enough to fully wipe out tranche B's realised liquidity; `_processDefault` calls `_popTranche(1, B)`. `totalInterestWeight` is correctly decremented by W1, `tranches` becomes length 1 ([A]), but `interestWeightTranches` remains length 2 ([W0, W1]) due to the missing pop.
3. Governance/owner calls `addTranche(C, Wc)` to onboard a new junior tranche C. This pushes Wc onto `interestWeightTranches`, making it [W0, W1, Wc] (length 3), while `tranches` becomes [A, C] (length 2). `totalInterestWeight` is now W0 + Wc (correct sum), but the array is misaligned.
4. On the next interest sync, `_syncInterestsToLiquidityProviders` loops i=0..1: for i=1 it reads `tranches[1]=C` but `interestWeightTranches[1]=W1` (tranche B's stale weight) instead of Wc. Tranche C's LPs receive interest proportional to the wrong (potentially much smaller or larger) weight indefinitely, while `remainingAssets` calculations (based on `totalInterestWeight` = W0+Wc) no longer match what was actually distributed, causing over/under-payment that silently accrues to or is stolen from the treasury/other tranches over time.
5. Impact compounds every block interest is synced, resulting in systematic misallocation of protocol yield, which existing LPs and the treasury can exploit or be harmed by depending on whether the stale weight is more or less favorable than the newly configured one.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -276,7 +276,8 @@
             totalInterestWeight -= interestWeightTranches[index];
         }
         isTranche[tranche] = false;
-        interestWeightTranches.pop();
+        /// DeleteExpressionMutation(`interestWeightTranches.pop()` |==> `assert(true)`) of: `interestWeightTranches.pop();`
+        assert(true);
         tranches.pop();
         interestWeight[tranche] = 0;
```

**Recommended test**

Add a unit test that: (1) adds two tranches, (2) triggers `_processDefault` with badDebt large enough to fully wipe and pop the junior tranche, (3) asserts `interestWeightTranches.length == tranches.length` after the pop, (4) calls `addTranche` again and asserts that `interestWeightTranches[index]` for the new tranche's index equals the newly supplied `interestWeight_` (not a stale value), and (5) runs an interest sync and asserts the interest paid to the new tranche matches `assets * newWeight / totalInterestWeight` rather than the old tranche's weight.

---

### 10. [HIGH] Incorrect surplus calculation in auctionRepay corrupts liquidation payout

**Location**: `src/LendingPool.sol:529` in `auctionRepay()`  
**Type**: Logic Error / Incorrect Arithmetic Operator  
**Confidence**: HIGH  
**Mutation**: `BinaryOpMutation` (id 132)

**What the code is supposed to enforce**

When a liquidation auction bid over-repays the outstanding debt (accountDebt <= amount), the surplus that must be returned to the Account owner and added to totalRealisedLiquidity is defined as the excess payment: amount - accountDebt. This preserves the pool's accounting invariant that totalRealisedLiquidity plus outstanding debt equals the actual underlying asset balance held by the contract.

**What would go wrong if it stopped doing so**

The mutation replaces subtraction with integer division (amount / accountDebt). Since bidders can (and normally do) pay far more than the tiny remaining debt in the final bid of an auction, amount / accountDebt (floor division) will almost always be dramatically smaller than the true surplus amount - accountDebt for any accountDebt > 1, and can even be exactly 1 wei larger when accountDebt == 1. In the typical case this causes the account owner to be credited far less than the actual overpayment, while the real, un-credited tokens remain stuck in the contract's balance without being reflected in totalRealisedLiquidity or realisedLiquidityOf. This breaks the core invariant that totalRealisedLiquidity + realisedDebt must track the pool's real asset holdings, results in silent fund loss for the liquidated Account owner (the rightful recipient of the surplus), and can also introduce discrepancies that get later swept to the treasury via skim(), further disadvantaging the affected user.

**Attack scenario (hypothetical — requires the change above)**

1. Account X accumulates debt D and gets liquidated, starting an auction (startLiquidation). 2. During the auction, the Liquidator processes several partial repayments through auctionRepay, driving Account X's remaining debt down to a small value, e.g. accountDebt = 100 (18-decimals token units). 3. A bidder submits a final overpayment of amount = 10,000 to close out the auction early (accountDebt <= amount triggers earlyTerminate). 4. The mutant computes surplus = amount / accountDebt = 100 instead of the correct amount - accountDebt = 9,900. 5. _settleLiquidationHappyFlow credits only 100 tokens to Account X's owner (realisedLiquidityOf[owner]) and only adds 100 to totalRealisedLiquidity, even though the bidder's full 10,000 tokens were transferred into the pool via safeTransferFrom at the start of auctionRepay. 6. The remaining 9,800 tokens sit in the contract's raw ERC20 balance but are not reflected in any account's realisedLiquidityOf nor in totalRealisedLiquidity, effectively lost to the rightful owner (and only recoverable, if at all, by the treasury via a later skim() call, still to the detriment of the Account owner).

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -526,7 +526,8 @@
             // -> Terminate the auction and make the surplus available to the Account-Owner.
             earlyTerminate = true;
             unchecked {
-                _settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount - accountDebt));
+                /// BinaryOpMutation(`-` |==> `/`) of: `_settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount - accountDebt));`
+                _settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount/accountDebt));
             }
             amount = accountDebt;
         }
```

**Recommended test**

Add a unit test for auctionRepay covering the early-termination branch (accountDebt < amount) that asserts the exact surplus value passed to _settleLiquidationHappyFlow equals amount - accountDebt (e.g., using an amount that is not an exact multiple of accountDebt, and a case where accountDebt > 1, to distinguish subtraction from integer division). The test should also assert that realisedLiquidityOf[accountOwner] increases by exactly amount - accountDebt and that totalRealisedLiquidity after the call reconciles with asset.balanceOf(pool) and realisedDebt, catching any deviation introduced by an incorrect arithmetic operator.

---

### 11. [HIGH] Liquidation initiator reward never credited to realisedLiquidityOf

**Location**: `src/LendingPool.sol:936` in `startLiquidation()`  
**Type**: Logic Error / Accounting Inconsistency  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 362)

**What the code is supposed to enforce**

When a liquidation is started, the initiator should be immediately credited with `initiationReward` in `realisedLiquidityOf[initiator]` so they can later withdraw this incentive via `withdrawFromLendingPool`. This is the immediate reward paid for triggering the liquidation of an unhealthy Account.

**What would go wrong if it stopped doing so**

The mutation replaces the increment `realisedLiquidityOf[initiator] += initiationReward;` with a no-op `assert(true);`. However, `totalRealisedLiquidity` is still increased by `initiationReward` on the very next line, and the debt tokens backing this reward (initiationReward + liquidationPenalty + terminationReward) are still minted to the Account via `_deposit`. This creates a permanent accounting mismatch: `totalRealisedLiquidity` is inflated by `initiationReward` but no individual balance (`realisedLiquidityOf[...]`) reflects it. The initiator can never withdraw their reward — the funds become permanently stuck/unaccounted for in the pool (not stealable by anyone, but also not recoverable, since `skim()` compares `balanceOf(this) + realisedDebt - totalRealisedLiquidity`, which will not detect this discrepancy because totalRealisedLiquidity was already inflated to match the minted debt). This breaks the core economic incentive that pays liquidators/initiators for keeping the protocol solvent, potentially discouraging liquidations of undercollateralized positions and increasing risk of bad debt accumulation.

**Attack scenario (hypothetical — requires the change above)**

1. An Account becomes undercollateralized. 2. A liquidator bot (the initiator) calls `startLiquidation(initiator, minimumMargin_)` expecting to earn `initiationReward` for triggering the liquidation. 3. The function mints debt tokens covering `initiationReward + liquidationPenalty + terminationReward` to the Account (increasing total debt/liquidity), and increments `totalRealisedLiquidity` by `initiationReward`, but due to the mutation, `realisedLiquidityOf[initiator]` is never incremented. 4. The initiator later calls `withdrawFromLendingPool` expecting to withdraw their `initiationReward`, but their `realisedLiquidityOf` balance shows zero (or whatever it was before), so the reward is unclaimable. 5. Over many liquidations, this steadily inflates `totalRealisedLiquidity` disconnected from any claimable balance, permanently locking those funds in the pool and removing the economic incentive to liquidate underwater positions, potentially causing under-liquidation and protocol insolvency risk.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -933,7 +933,8 @@
         // Increase the realised liquidity for the initiator.
         // The other incentives will only be added as realised liquidity for the respective actors
         // after the auction is finished.
-        realisedLiquidityOf[initiator] += initiationReward;
+        /// DeleteExpressionMutation(`realisedLiquidityOf[initiator] += initiationReward` |==> `assert(true)`) of: `realisedLiquidityOf[initiator] += initiationReward;`
+        assert(true);
         totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity + initiationReward);
 
         // If this is the sole ongoing auction, prevent any deposits and withdrawals in the most jr tranche
```

**Recommended test**

Add a unit test for `startLiquidation` that asserts `realisedLiquidityOf[initiator]` increases by exactly `initiationReward` after the call (e.g. `assertEq(realisedLiquidityOf(initiator), initiationRewardExpected)`), and a follow-up test that the initiator can successfully call `withdrawFromLendingPool(initiationReward, initiator)` and receive the underlying asset. Additionally add an invariant test asserting `totalRealisedLiquidity == sum(realisedLiquidityOf[all actors])` after `startLiquidation` to catch any accounting drift.

---

### 12. [HIGH] Missing totalRealisedLiquidity update in startLiquidation breaks pool accounting invariant

**Location**: `src/LendingPool.sol:937` in `startLiquidation()`  
**Type**: Accounting/Invariant Violation  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 363)

**What the code is supposed to enforce**

When a liquidation is started, the initiationReward is credited to the initiator's realisedLiquidityOf balance. The original code simultaneously increases totalRealisedLiquidity by the same amount, keeping the invariant that totalRealisedLiquidity equals (approximately) the sum of all individual realisedLiquidityOf balances, and that this sum is backed by the pool's underlying assets plus outstanding debt (realisedDebt).

**What would go wrong if it stopped doing so**

By deleting the totalRealisedLiquidity update (replacing it with a no-op assert(true)), realisedLiquidityOf[initiator] is incremented without a corresponding increase in totalRealisedLiquidity. This immediately desynchronizes the sum of all individual liquidity balances from the pool-wide totalRealisedLiquidity counter. Concretely: (1) skim() computes `delta = asset.balanceOf(address(this)) + realisedDebt - totalRealisedLiquidity`; since totalRealisedLiquidity is now understated by the missing initiationReward, skim() will compute an inflated 'surplus' and credit it entirely to the treasury even though part of that surplus is actually the initiator's already-earned (but unaccounted) reward — effectively double-allocating value to both the initiator and the treasury against the same underlying assets. (2) The utilisation calculation in _updateInterestRate uses totalRealisedLiquidity as totalLiquidity_; understating it artificially inflates utilisation and therefore the interest rate charged to all borrowers. (3) Persistent under-accounting compounds with every liquidation, degrading solvency guarantees of the pool over time and potentially causing safeCastTo128 underflow reverts (DoS) in withdrawFromLendingPool once accumulated balances exceed the understated totalRealisedLiquidity.

**Attack scenario (hypothetical — requires the change above)**

1. Attacker identifies (or creates) an undercollateralized Account and calls the liquidation flow, having themselves set as `initiator` in startLiquidation. 2. Each triggered liquidation credits realisedLiquidityOf[initiator] with initiationReward, but totalRealisedLiquidity is never incremented (due to the mutation). 3. Attacker repeats this across multiple liquidations to accumulate a growing accounting gap between the sum of realisedLiquidityOf balances and totalRealisedLiquidity. 4. Attacker (or anyone) calls the permissionless skim() function; because totalRealisedLiquidity is understated relative to actual asset balance + realisedDebt, skim() computes an inflated delta and credits it fully to the treasury's realisedLiquidityOf, effectively creating new claims on the same underlying assets that are already claimed by the initiator — a double counting of value. 5. Over time, redeemable claims (sum of realisedLiquidityOf) exceed the pool's real backing, and/or withdrawals begin to revert due to uint128 underflow in totalRealisedLiquidity arithmetic, leading to fund misallocation and potential denial of service for legitimate liquidity providers.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -934,7 +934,8 @@
         // The other incentives will only be added as realised liquidity for the respective actors
         // after the auction is finished.
         realisedLiquidityOf[initiator] += initiationReward;
-        totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity + initiationReward);
+        /// DeleteExpressionMutation(`totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity + initiationReward)` |==> `assert(true)`) of: `totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity + initiationReward);`
+        assert(true);
 
         // If this is the sole ongoing auction, prevent any deposits and withdrawals in the most jr tranche
         if (auctionsInProgress == 0 && tranches.length > 0) {
```

**Recommended test**

Add a unit test for startLiquidation that asserts totalRealisedLiquidity increases by exactly `initiationReward` after the call (in addition to the existing assertion on realisedLiquidityOf[initiator]). Additionally, add an invariant test that checks `totalRealisedLiquidity == sum(realisedLiquidityOf[all tranches] + realisedLiquidityOf[treasury] + realisedLiquidityOf[initiator])` after any liquidation-related state change, and a regression test combining startLiquidation followed by skim() to verify the treasury does not receive inflated surplus.

---

### 13. [HIGH] Sign flip in totalRealisedLiquidity update breaks accounting invariant in liquidation happy flow

**Location**: `src/LendingPool.sol:1005` in `_settleLiquidationHappyFlow()`  
**Type**: Arithmetic/Accounting Invariant Violation  
**Confidence**: HIGH  
**Mutation**: `BinaryOpMutation` (id 390)

**What the code is supposed to enforce**

After a successful liquidation settlement, totalRealisedLiquidity (the aggregate of all claimable liquidity) must be incremented by the exact sum of the terminationReward, liquidationPenalty, and surplus that are simultaneously credited to individual balances (terminator, treasury/tranche, account owner) via realisedLiquidityOf. This keeps totalRealisedLiquidity == sum(realisedLiquidityOf[*]) at all times, which is a core solvency invariant of the pool.

**What would go wrong if it stopped doing so**

The mutation flips the `+` before `terminationReward` to a `-`, so totalRealisedLiquidity is decreased by terminationReward instead of increased, while realisedLiquidityOf[terminator] is still credited with the full terminationReward in the same function. This creates a permanent divergence of 2x terminationReward between the aggregate totalRealisedLiquidity and the sum of individual claimable balances after every happy-flow liquidation settlement with a non-zero terminationReward. Consequences: (1) skim() computes `asset.balanceOf(this) + realisedDebt - totalRealisedLiquidity` as surplus to sweep to the treasury — an artificially low totalRealisedLiquidity inflates this delta, letting the treasury (or whoever triggers skim, since it's unprivileged) siphon off assets that rightfully belong to LPs/terminator, i.e. real fund loss for depositors. (2) _updateInterestRate uses totalRealisedLiquidity as totalLiquidity_ in the utilisation calculation; understating it inflates utilisation and thus interest rates charged to borrowers. (3) Repeated happy-flow settlements can drive totalRealisedLiquidity low enough that later subtraction in withdrawFromLendingPool (`totalRealisedLiquidity - assets`) underflows/reverts via SafeCastLib, causing a DoS on withdrawals for legitimate LPs even though the pool holds sufficient underlying assets.

**Attack scenario (hypothetical — requires the change above)**

1. A liquidation is initiated and settled via the happy flow (_settleLiquidationHappyFlow) with a non-zero terminationReward (e.g., terminator gets paid X tokens). realisedLiquidityOf[terminator] += X but totalRealisedLiquidity -= X instead of += X, creating a 2X gap. 2. After one or more such settlements, an attacker (or even the treasury manager, but skim is callable by anyone) calls skim(). Because totalRealisedLiquidity is now understated by cumulative amounts, `asset.balanceOf(address(this)) + realisedDebt - totalRealisedLiquidity` returns an inflated `delta`, which is credited entirely to realisedLiquidityOf[treasury] and added back to totalRealisedLiquidity. 3. The treasury (or an accomplice with access) withdraws this inflated delta via withdrawFromLendingPool, extracting real underlying assets that were actually owed to LPs/terminators, leaving the pool under-collateralized relative to legitimate claims. 4. Alternatively, over enough liquidations, totalRealisedLiquidity is driven low enough that legitimate LP withdrawals begin reverting due to underflow in `totalRealisedLiquidity - assets`, freezing funds for LPs despite sufficient underlying balance.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1002,7 +1002,8 @@
         _syncLiquidationFee(liquidationPenalty);
 
         totalRealisedLiquidity =
-            SafeCastLib.safeCastTo128(totalRealisedLiquidity + terminationReward + liquidationPenalty + surplus);
+            /// BinaryOpMutation(`+` |==> `-`) of: `SafeCastLib.safeCastTo128(totalRealisedLiquidity + terminationReward + liquidationPenalty + surplus);`
+            SafeCastLib.safeCastTo128(totalRealisedLiquidity-terminationReward + liquidationPenalty + surplus);
 
         unchecked {
             // Pay out any surplus to the current Account Owner.
```

**Recommended test**

Add a unit test for _settleLiquidationHappyFlow (or the public wrappers settleLiquidationHappyFlow / auctionRepay early-terminate path) that asserts totalRealisedLiquidity increases by exactly (terminationReward + liquidationPenalty + surplus) and that sum(realisedLiquidityOf[terminator] + realisedLiquidityOf[owner] + tranche/treasury shares) equals the new totalRealisedLiquidity delta. Also add an invariant/fuzz test that checks `totalRealisedLiquidity == realisedLiquidityOf[treasury] + sum(realisedLiquidityOf[tranches]) + other credited balances` holds after any liquidation settlement sequence, and a regression test calling skim() after a happy-flow settlement with non-zero terminationReward to ensure no unintended surplus is skimmed to treasury.

---

### 14. [HIGH] Termination reward not credited in happy-flow liquidation settlement

**Location**: `src/LendingPool.sol:1011` in `_settleLiquidationHappyFlow()`  
**Type**: Logic Error / Broken Accounting Invariant  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 397)

**What the code is supposed to enforce**

After a successful (happy-flow) liquidation auction, the code is meant to credit the auction terminator's `realisedLiquidityOf` balance with the `terminationReward` so that they can later withdraw the fee they earned for closing out the auction.

**What would go wrong if it stopped doing so**

The mutation replaces the credit to `realisedLiquidityOf[terminator]` with a no-op `assert(true)`. However, `totalRealisedLiquidity` is still incremented by `terminationReward` (via the line `totalRealisedLiquidity = SafeCastLib.safeCastTo128(totalRealisedLiquidity + terminationReward + liquidationPenalty + surplus)`), which executes before the deleted line. This creates a permanent accounting mismatch: `totalRealisedLiquidity` grows as though the terminator was paid, but no individual balance reflects that value. The terminator is silently denied their rightful termination reward, and the corresponding underlying assets become orphaned in the pool (only recoverable later by the treasury via `skim()`, since skim() reconciles `asset.balanceOf(this) + realisedDebt - totalRealisedLiquidity`, but the orphaned funds sit inside totalRealisedLiquidity as unclaimed, so they are actually inaccessible to anyone — no one's realisedLiquidityOf entry accounts for them). This breaks the core invariant that the sum of all `realisedLiquidityOf` balances should equal `totalRealisedLiquidity`, and directly costs the liquidation terminator their fee, discouraging third parties from finishing auctions (a service critical to the liquidation mechanism).

**Attack scenario (hypothetical — requires the change above)**

1. An Account becomes undercollateralized and `startLiquidation` is called, starting an auction.
2. A bidder repays enough debt during the auction such that `auctionRepay` triggers the happy flow (`accountDebt <= amount`), calling `_settleLiquidationHappyFlow` with a `terminator` address (the address that ends the auction, possibly a bot/keeper).
3. `_calculateRewards` computes a nonzero `terminationReward` for the terminator.
4. Due to the mutation, `realisedLiquidityOf[terminator]` is never incremented, even though `totalRealisedLiquidity` already includes `terminationReward`.
5. The terminator later calls `withdrawFromLendingPool` expecting to withdraw their reward, but their balance shows zero increase — they receive nothing for terminating the auction.
6. Over repeated liquidations, the pool accumulates 'lost' value baked into `totalRealisedLiquidity` that is not attributable to any account, silently corrupting the pool's internal accounting and permanently under-compensating liquidation terminators (a griefing/DoS on the incentive mechanism, and value that becomes stuck/unclaimable in the protocol).

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1008,7 +1008,8 @@
             // Pay out any surplus to the current Account Owner.
             if (surplus > 0) realisedLiquidityOf[IAccount(account).owner()] += surplus;
             // Pay out the "terminationReward" to the "terminator".
-            realisedLiquidityOf[terminator] += terminationReward;
+            /// DeleteExpressionMutation(`realisedLiquidityOf[terminator] += terminationReward` |==> `assert(true)`) of: `realisedLiquidityOf[terminator] += terminationReward;`
+            assert(true);
         }
 
         _endLiquidation();
```

**Recommended test**

Add a unit test for `_settleLiquidationHappyFlow` (via `settleLiquidationHappyFlow` and via `auctionRepay`'s early-terminate branch) that asserts `realisedLiquidityOf[terminator]` increases by exactly `terminationReward` after the call, and that the sum of all `realisedLiquidityOf` balances (tranches + treasury + terminator + account owner) exactly matches the increase in `totalRealisedLiquidity`. This would kill the mutant since `assert(true)` leaves the terminator's balance unchanged while `totalRealisedLiquidity` still increases.

---

### 15. [HIGH] Multiplication instead of addition breaks bad-debt threshold check

**Location**: `src/LendingPool.sol:1050` in `settleLiquidationUnhappyFlow()`  
**Type**: Logic Error / Arithmetic Underflow (Denial of Service)  
**Confidence**: HIGH  
**Mutation**: `BinaryOpMutation` (id 401)

**What the code is supposed to enforce**

The condition `openDebt > terminationReward + liquidationPenalty` determines whether the outstanding debt after an auction exceeds the total pending liquidation incentives. If it does, the excess is written off as bad debt via `_processDefault`, incentives are zeroed, and accounting is adjusted safely. If it does not, the remaining incentives (`remainder = terminationReward + liquidationPenalty - openDebt`) are distributed to the terminator/tranche/treasury.

**What would go wrong if it stopped doing so**

Replacing the sum with a product (`terminationReward * liquidationPenalty`) produces a value on a completely different (and vastly larger, since both are absolute asset amounts, not fractions) scale than `openDebt`. For essentially all realistic reward/penalty values, `terminationReward * liquidationPenalty` will be orders of magnitude larger than `openDebt`, so the `if` branch (which handles genuine bad debt and safely calls `_processDefault`) will almost never be entered — even in the exact scenario it is designed for (openDebt actually exceeding the sum of incentives). Execution then falls into the `else` branch, where `uint256 remainder = liquidationPenalty + terminationReward - openDebt;` is computed without an `unchecked` block. When real bad debt exists (openDebt > terminationReward + liquidationPenalty), this subtraction underflows and reverts under Solidity 0.8 checked arithmetic. The transaction reverts, `_endLiquidation()` is never reached, `auctionsInProgress` is never decremented, the most-junior Tranche remains locked (deposits/withdrawals blocked), and the Account's bad debt is never written off — permanently corrupting the protocol's ability to resolve insolvent positions.

**Attack scenario (hypothetical — requires the change above)**

1. Market conditions cause an Account's collateral value to crash such that after full liquidation, `openDebt` (via convertToAssets(debtShares)) genuinely exceeds `terminationReward + liquidationPenalty` (a real, non-malicious bad-debt event). 2. The Liquidator contract calls `settleLiquidationUnhappyFlow(account, startDebt, minimumMargin_, terminator)` to close out the auction and write off the bad debt. 3. Due to the mutated multiplication, the `if` condition evaluates to false (since the product of the two reward amounts vastly exceeds any realistic openDebt), routing execution into the `else` branch. 4. `remainder = liquidationPenalty + terminationReward - openDebt` underflows because openDebt actually exceeds the sum, causing an automatic revert. 5. The liquidation can never be settled: `auctionsInProgress` stays incremented, the junior Tranche remains locked indefinitely, and the Account's toxic debt is never removed from the system, potentially cascading into protocol-wide insolvency or a stuck state requiring emergency intervention.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1047,7 +1047,8 @@
         uint256 debtShares = balanceOf[account];
         uint256 openDebt = convertToAssets(debtShares);
         uint256 badDebt;
-        if (openDebt > terminationReward + liquidationPenalty) {
+        /// BinaryOpMutation(`+` |==> `*`) of: `if (openDebt > terminationReward + liquidationPenalty) {`
+        if (openDebt > terminationReward*liquidationPenalty) {
             // "openDebt" is bigger than pending liquidation incentives.
             // No incentives will be paid out, and a default event is triggered.
             unchecked {
```

**Recommended test**

Add a unit/fuzz test for `settleLiquidationUnhappyFlow` that constructs a scenario where `openDebt` (derived from `startDebt`/mocked debt balance) is strictly greater than `terminationReward + liquidationPenalty` (true bad-debt case) and asserts: (a) the call does not revert, (b) `_processDefault` is invoked with the correct `badDebt` amount, (c) `terminationReward` and `liquidationPenalty` are zeroed, and (d) `auctionsInProgress` is decremented afterwards. Also add a boundary test where openDebt exactly equals the sum, and one where it is smaller, verifying the `remainder` distribution logic — these would all fail under the `*` mutation.

---

### 16. [HIGH] Incorrect badDebt calculation in settleLiquidationUnhappyFlow

**Location**: `src/LendingPool.sol:1054` in `settleLiquidationUnhappyFlow()`  
**Type**: Logic Error / Incorrect Arithmetic  
**Confidence**: HIGH  
**Mutation**: `BinaryOpMutation` (id 441)

**What the code is supposed to enforce**

When open debt exceeds the pending liquidation incentives (terminationReward + liquidationPenalty), the code should calculate badDebt as the excess debt not covered by incentives: openDebt - terminationReward - liquidationPenalty. This badDebt is then subtracted from totalRealisedLiquidity and used to write off losses pro-rata across tranches via _processDefault.

**What would go wrong if it stopped doing so**

The mutation changes the subtraction of terminationReward into an addition, making badDebt = openDebt + terminationReward - liquidationPenalty instead of openDebt - terminationReward - liquidationPenalty. This inflates badDebt by 2x terminationReward. Consequences: (1) totalRealisedLiquidity is decremented by an inflated amount via `uint128(totalRealisedLiquidity - badDebt)`, which can underflow/revert or corrupt pool accounting; (2) _processDefault(badDebt) will write off more liquidity from tranches than actually lost, harming LPs by socializing losses that shouldn't exist; (3) since the branch also zeroes out terminationReward and liquidationPenalty afterward, the terminator/tranche/treasury lose their legitimate share while extra funds are incorrectly burned from tranche liquidity. This directly corrupts the pool's internal accounting invariant (sum of realisedLiquidityOf == totalRealisedLiquidity) and causes real fund loss to LPs.

**Attack scenario (hypothetical — requires the change above)**

1. An Account accumulates debt and becomes eligible for liquidation with startDebt above the sum of pending incentives (bad debt case). 2. Liquidator (or any actor triggering the unhappy flow via the Liquidator contract) calls settleLiquidationUnhappyFlow with parameters such that openDebt > terminationReward + liquidationPenalty. 3. The mutated code computes badDebt = openDebt + terminationReward - liquidationPenalty, which is larger than the correct value by 2*terminationReward. 4. This inflated badDebt is subtracted from totalRealisedLiquidity (potentially underflowing/reverting due to unsafe cast, or if it doesn't revert, silently reducing the pool's recorded liquidity beyond what was actually lost). 5. _processDefault(badDebt) then wipes out more tranche liquidity than truly lost, transferring losses to LPs (especially junior tranche holders) that exceed the real bad debt, benefiting no one but corrupting protocol solvency and enabling repeated triggering of this flow to systematically drain tranche liquidity beyond actual losses.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1051,7 +1051,8 @@
             // "openDebt" is bigger than pending liquidation incentives.
             // No incentives will be paid out, and a default event is triggered.
             unchecked {
-                badDebt = openDebt - terminationReward - liquidationPenalty;
+                /// BinaryOpMutation(`-` |==> `+`) of: `badDebt = openDebt - terminationReward - liquidationPenalty;`
+                badDebt = openDebt+terminationReward - liquidationPenalty;
             }
 
             // unsafe cast: uint128 - uint256 is always smaller than uint128.
```

**Recommended test**

Add a unit test for settleLiquidationUnhappyFlow that covers the 'openDebt > terminationReward + liquidationPenalty' branch and asserts the exact computed badDebt value equals openDebt - terminationReward - liquidationPenalty (not a value inflated by 2x terminationReward). Also assert that totalRealisedLiquidity decreases by exactly the expected badDebt amount and that realisedLiquidityOf mapping sums remain consistent with totalRealisedLiquidity after the call, using distinct nonzero values for terminationReward and liquidationPenalty so the mutation would produce a detectably different badDebt/output.

---

### 17. [HIGH] Stale realisedLiquidityOf balance retained after Tranche wipeout in _processDefault

**Location**: `src/LendingPool.sol:1142` in `_processDefault()`  
**Type**: Accounting/Invariant Violation (Improper State Cleanup)  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 472)

**What the code is supposed to enforce**

When a Tranche's liquidity is fully consumed by bad debt, the code must zero out `realisedLiquidityOf[tranche]` before popping the Tranche from the `tranches` array and locking it, so the internal ledger (`realisedLiquidityOf` mapping) stays consistent with `totalRealisedLiquidity` and the Tranche has no residual claim on pool funds.

**What would go wrong if it stopped doing so**

The mutation replaces `realisedLiquidityOf[tranche] = 0;` with a no-op (`assert(true)`). As a result, after a Tranche is fully wiped out due to bad debt, its `realisedLiquidityOf[tranche]` entry keeps its pre-wipeout balance (`maxBurnable`) even though: (1) `totalRealisedLiquidity` was already decremented by the corresponding `badDebt` amount in the caller (`settleLiquidationUnhappyFlow`), and (2) the Tranche is removed from `isTranche`/`tranches` and `lock()`-ed. This breaks the core invariant that the sum of all `realisedLiquidityOf` balances must equal `totalRealisedLiquidity`. The wiped Tranche address retains a stale claim equal to the wiped-out amount. If the Tranche is ever reactivated (the code comments explicitly anticipate 'DAO or insurance might refund (Part of) the losses, and add Tranche back') or if the Tranche's lock is later lifted for any reason, that address can still redeem its now-incorrect stale balance via `withdrawFromLendingPool`, which has no `onlyTranche` restriction and only checks `realisedLiquidityOf[msg.sender] >= assets`. This would allow double-counted withdrawal of funds that were already written off as bad debt, draining assets that back other LPs/treasury and pushing the pool into insolvency.

**Attack scenario (hypothetical — requires the change above)**

1. A Tranche T's underlying position defaults; `_processDefault` is invoked with `badDebt >= realisedLiquidityOf[T]`, fully wiping T. Due to the mutation, `realisedLiquidityOf[T]` is NOT reset to 0 (it stays at the pre-wipeout value, e.g. 100,000 tokens), while `totalRealisedLiquidity` has already been reduced by the same amount in the caller. 2. T is popped from `tranches`, `isTranche[T]=false`, and `ITranche(T).lock()` is called. 3. Later, per the documented recovery path, the DAO/insurance compensates part of the loss and re-adds/unlocks Tranche T (e.g., via `addTranche` or an unlock call on the Tranche contract) without manually resetting `realisedLiquidityOf[T]`. 4. Tranche T (or anyone able to trigger a call from T's address, e.g. through its own redeem flow) calls `withdrawFromLendingPool(100000, receiver)`. Since `realisedLiquidityOf[T]` still shows 100,000 (never zeroed), the check passes and 100,000 tokens are transferred out of the pool, even though this liquidity was already accounted as lost/written-off. 5. This creates a shortfall: `totalRealisedLiquidity` and other LPs'/treasury's claims are no longer backed by actual pool assets, leading to insolvency for the last withdrawers.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1139,7 +1139,8 @@
                 // badDebt is bigger than the balance of most junior Tranche -> tranche is completely wiped out
                 // and temporarily locked (no new deposits or withdraws possible).
                 // DAO or insurance might refund (Part of) the losses, and add Tranche back.
-                realisedLiquidityOf[tranche] = 0;
+                /// DeleteExpressionMutation(`realisedLiquidityOf[tranche] = 0` |==> `assert(true)`) of: `realisedLiquidityOf[tranche] = 0;`
+                assert(true);
                 _popTranche(i, tranche);
                 unchecked {
                     badDebt -= maxBurnable;
```

**Recommended test**

Add a unit test that: (a) sets up a Tranche with a small realised liquidity balance, (b) triggers `_processDefault`/`settleLiquidationUnhappyFlow` with `badDebt >= trancheBalance` so the Tranche is fully wiped and popped, and (c) asserts `realisedLiquidityOf[trancheAddress] == 0` after the call, in addition to verifying `isTranche[trancheAddress] == false` and the tranche was removed from the `tranches` array. Also add an invariant test that `totalRealisedLiquidity == sum(realisedLiquidityOf[all tranches] + realisedLiquidityOf[treasury])` after a full default wipeout, which the current suite does not check and thus fails to kill this mutant.

---

### 18. [HIGH] Deleted _popTranche() call corrupts tranche accounting after tranche wipeout

**Location**: `src/LendingPool.sol:1143` in `_processDefault()`  
**Type**: Broken Invariant / Accounting Corruption  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 473)

**What the code is supposed to enforce**

When a Tranche's realised liquidity is fully wiped out by bad debt during _processDefault, _popTranche() must remove the tranche from the `tranches` array and `interestWeightTranches` array, zero out its `interestWeight` mapping entry, subtract its weight from `totalInterestWeight`, and mark `isTranche[tranche] = false`. This keeps the pool's bookkeeping (totalInterestWeight, tranche array ordering, isTranche flags) consistent so that future interest/liquidation-fee distributions and 'most junior tranche' lookups correctly skip the defaulted tranche.

**What would go wrong if it stopped doing so**

With `_popTranche(i, tranche)` replaced by a no-op `assert(true)`, the wiped-out tranche remains in the `tranches` array (still occupying the last/most-junior index or a middle index), `isTranche[tranche]` stays `true`, `interestWeight[tranche]` is never zeroed, and `totalInterestWeight` is never decremented by the dead tranche's weight. Concretely: (1) `_syncInterestsToLiquidityProviders` still counts the dead tranche's weight inside `totalInterestWeight` when computing every other tranche's share, permanently diluting the interest paid to surviving LPs and inflating the treasury's take, breaking the documented interest-distribution invariant; (2) `_syncLiquidationFee` and any subsequent `_processDefault` call use `tranches[tranches.length - 1]` to identify the 'most junior tranche' — since the dead/locked tranche was never popped, it can still be selected as the most-junior tranche, causing future liquidation penalties to be credited to `realisedLiquidityOf[deadTranche]`, a tranche that has been `.lock()`ed and can no longer be withdrawn from without manual DAO intervention, effectively freezing those funds; (3) `isTranche[tranche]` staying true means the LendingPool still treats the defaulted tranche as authorized for `onlyTranche`-gated functions like `depositInLendingPool`, relying entirely on the external Tranche contract's own lock to prevent misuse — a second line of defense that should not be the only one.

**Attack scenario (hypothetical — requires the change above)**

1) A junior Tranche's realised liquidity is fully consumed by a large default (`_processDefault` badDebt >= tranche balance), triggering the wipe-out branch. 2) With the mutation, `tranches` still contains the wiped tranche and `totalInterestWeight` is not reduced. 3) A subsequent liquidation completes and calls `_syncLiquidationFee`, which reads `tranches[tranches.length - 1]` — still the defaulted, locked tranche — and credits `realisedLiquidityOf[deadTranche]` with the liquidation penalty share intended for LPs. 4) Because the Tranche contract is locked, no one can withdraw this credited amount, permanently stranding LP funds. 5) Simultaneously, every future interest sync distributes proportionally less interest to remaining live tranches (since `totalInterestWeight` still includes the dead tranche's weight) and proportionally more to the treasury, silently siphoning value away from LPs over time.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1140,7 +1140,8 @@
                 // and temporarily locked (no new deposits or withdraws possible).
                 // DAO or insurance might refund (Part of) the losses, and add Tranche back.
                 realisedLiquidityOf[tranche] = 0;
-                _popTranche(i, tranche);
+                /// DeleteExpressionMutation(`_popTranche(i, tranche)` |==> `assert(true)`) of: `_popTranche(i, tranche);`
+                assert(true);
                 unchecked {
                     badDebt -= maxBurnable;
                 }
```

**Recommended test**

Add a unit test that: (a) forces a default large enough to fully wipe a tranche's realised liquidity via `_processDefault`, (b) asserts `isTranche[tranche] == false`, `interestWeight[tranche] == 0`, `tranches.length` decreased by one, and `totalInterestWeight` decreased by exactly the popped tranche's weight after the call; and (c) triggers a subsequent `_syncLiquidationFee`/liquidation settlement and asserts the liquidation penalty is credited to the new (correct) most-junior tranche rather than the defaulted one. This would fail under the mutant since `_popTranche` is never invoked.

---

### 19. [HIGH] Missing badDebt decrement causes over-write-off across multiple tranches in _processDefault

**Location**: `src/LendingPool.sol:1145` in `_processDefault()`  
**Type**: Logic Error / Accounting Invariant Violation  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 474)

**What the code is supposed to enforce**

In _processDefault, when bad debt exceeds a junior tranche's realised liquidity, that tranche is fully wiped and popped, and the remaining bad debt (badDebt -= maxBurnable) must be reduced by the amount already absorbed by that tranche before checking/absorbing the shortfall from the next (more senior) tranche. This ensures the total amount written off across all tranches equals exactly the original badDebt passed into the function, preserving the invariant totalRealisedLiquidity == sum(realisedLiquidityOf[*]).

**What would go wrong if it stopped doing so**

By replacing `badDebt -= maxBurnable;` with a no-op (`assert(true)`), the loop variable `badDebt` never decreases as tranches are fully wiped out. On the next iteration (next more senior tranche), the code re-compares the *original, undiminished* badDebt against that tranche's balance instead of the actual remaining shortfall. This causes the function to either (a) wipe out more tranches than necessary, or (b) once it reaches a tranche whose balance exceeds the stale (still-full) badDebt, deduct far more than the true remaining shortfall from that tranche via `realisedLiquidityOf[tranche] -= badDebt`. The net effect is that the sum of amounts subtracted from realisedLiquidityOf mappings across tranches exceeds the true badDebt that was already subtracted once from totalRealisedLiquidity before the loop began. This breaks the core solvency invariant (sum of individual claims == totalRealisedLiquidity) and causes tranche LPs (particularly more senior tranches that would not otherwise have needed to absorb any loss) to have their claimable liquidity reduced by far more than their fair pro-rata share of the actual bad debt — a direct, non-recoverable loss of LP funds.

**Attack scenario (hypothetical — requires the change above)**

1. Multiple tranches exist, e.g. Tranche A (junior, balance 30), Tranche B (mezzanine, balance 50), Tranche C (senior, balance 200).
2. A liquidation ends in the unhappy flow with badDebt = 100 (i.e., openDebt exceeds terminationReward+liquidationPenalty by 100), triggering `_processDefault(100)`.
3. Loop i=A: maxBurnable=30; since 100 >= 30, Tranche A is fully wiped (balance set to 0, popped, locked). Correct behavior: badDebt should become 70; with the mutation it stays 100.
4. Loop i=B: maxBurnable=50; since 100 >= 50 (mutant) vs correct 70 >= 50, Tranche B is also fully wiped either way in this example, but badDebt should become 20 (70-50); with the mutation it stays 100.
5. Loop i=C: maxBurnable=200; correct check is 20 < 200 → deduct 20 from Tranche C's realisedLiquidityOf and break. With the mutation, check is 100 < 200 → deduct 100 from Tranche C instead of 20, an 80-unit excess loss imposed on the senior tranche's LPs that was never actually incurred as real bad debt.
6. This excess 80 units silently disappears from realisedLiquidityOf[TrancheC] without any corresponding reduction having been applied to totalRealisedLiquidity (which was already correctly reduced by exactly 100 before the loop). The pool's accounting now shows totalRealisedLiquidity higher than the sum of all individual realisedLiquidityOf balances, meaning some legitimate LP claims will not be redeemable later (funds effectively burned from senior LPs), or conversely other balances become inconsistent, ultimately harming senior Tranche LPs who withdraw less than they are entitled to.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1142,7 +1142,8 @@
                 realisedLiquidityOf[tranche] = 0;
                 _popTranche(i, tranche);
                 unchecked {
-                    badDebt -= maxBurnable;
+                    /// DeleteExpressionMutation(`badDebt -= maxBurnable` |==> `assert(true)`) of: `badDebt -= maxBurnable;`
+                    assert(true);
                 }
                 // forge-lint: disable-next-item(reentrancy-no-eth)
                 ITranche(tranche).lock();
```

**Recommended test**

Add a unit test for `_processDefault`/`settleLiquidationUnhappyFlow` that triggers a badDebt event large enough to fully wipe out at least two tranches in a single call (badDebt > balance of junior tranche + balance of next tranche, but < balance of the third/most senior tranche). Assert that: (1) each wiped tranche's realisedLiquidityOf is exactly 0, (2) the surviving senior tranche's realisedLiquidityOf is reduced by exactly `badDebt - sum(maxBurnable of wiped tranches)` (not by the full original badDebt), and (3) the sum of all realisedLiquidityOf balances after the call equals `totalRealisedLiquidity`. This test would fail under the mutant since the senior tranche would be over-debited.

---

### 20. [HIGH] Underflow revert in _processDefault when wiping ≥3 tranches (i-1 replaced by 1-i)

**Location**: `src/LendingPool.sol:1151` in `_processDefault()`  
**Type**: Arithmetic Underflow / Denial of Service  
**Confidence**: HIGH  
**Mutation**: `SwapArgumentsOperatorMutation` (id 482)

**What the code is supposed to enforce**

When a Tranche is fully wiped out due to bad debt, the loop must inform the next-most-junior Tranche (index i-1) that an auction/default is in progress via setAuctionInProgress(true), while guarding against underflow when i==0 with the `if (i != 0)` check.

**What would go wrong if it stopped doing so**

The mutation swaps `tranches[i - 1]` for `tranches[1 - i]`. Since `i` is a `uint256` and this expression sits outside any `unchecked` block, Solidity 0.8's default checked arithmetic will revert whenever `i > 1` (i.e., whenever more than two Tranches must be sequentially wiped out during a single _processDefault call). Only for i==1 does `1-i` coincidentally equal `i-1` (=0), masking the bug in simple 2-tranche test scenarios. For pools configured with 3 or more Tranches, a sufficiently large default that wipes out the two most junior Tranches in the same call will cause the third iteration (i>=2) to underflow and revert the entire transaction. Because _processDefault is invoked from settleLiquidationUnhappyFlow (guarded by onlyLiquidator/processInterests), a revert here reverts the whole liquidation settlement call. This leaves the auction unterminated, auctionsInProgress never decremented, the most junior tranche permanently locked (setAuctionInProgress stuck true), and addTranche/setInterestWeightTranche etc. blocked by the `AuctionOngoing` check indefinitely — effectively bricking core pool operations for any protocol configured with 3+ tranches.

**Attack scenario (hypothetical — requires the change above)**

1. Protocol operator configures 3 Tranches (senior, mezzanine, junior) as intended by design.
2. A large Account under-collateralizes such that upon liquidation, `openDebt` after the auction exceeds `terminationReward + liquidationPenalty` by an amount large enough to fully wipe out both the junior and mezzanine tranches' realised liquidity (badDebt cascades across 2+ tranches).
3. Liquidator calls `settleLiquidationUnhappyFlow`, which invokes `_processDefault(badDebt)`.
4. The loop processes i = length-1 (junior tranche) fully wiped, decrements i to a value >=2, and attempts `tranches[1 - i]` — with i>=2 this underflows and reverts.
5. The entire `settleLiquidationUnhappyFlow` transaction reverts. The liquidation can never be settled through the unhappy-flow path, `auctionsInProgress` stays elevated, the junior tranche remains permanently locked (no deposits/withdrawals), and `addTranche`/parameter-setting functions guarded by `AuctionOngoing` become permanently unusable — a protocol-wide denial of service triggered simply by normal (adverse) market conditions, no privileged access required.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1148,7 +1148,8 @@
                 ITranche(tranche).lock();
                 // Hook to the new most junior Tranche to inform that auctions are ongoing.
                 // forge-lint: disable-next-item(reentrancy-no-eth)
-                if (i != 0) ITranche(tranches[i - 1]).setAuctionInProgress(true);
+                /// SwapArgumentsOperatorMutation(`i - 1` |==> `1 - i`) of: `if (i != 0) ITranche(tranches[i - 1]).setAuctionInProgress(true);`
+                if (i != 0) ITranche(tranches[1 - i]).setAuctionInProgress(true);
             }
         }
     }
```

**Recommended test**

Add a unit test for `_processDefault`/`settleLiquidationUnhappyFlow` with at least 3 tranches configured, where badDebt is large enough to fully wipe out the two most junior tranches in a single call. Assert that the call does not revert, that `ITranche(tranches[i-1]).setAuctionInProgress(true)` is called with the correct (i-1) index for i>=2, and that the third-most-junior tranche's auction-in-progress state is correctly set to true after the call.

---

### 21. [HIGH] Missing minReward floor for liquidation initiationReward

**Location**: `src/LendingPool.sol:1214` in `_calculateRewards()`  
**Type**: Logic Error / Broken Invariant  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 515)

**What the code is supposed to enforce**

The line `initiationReward = initiationReward > minReward ? initiationReward : minReward;` guarantees that the liquidation initiator always receives at least `minReward` (a value sized to cover the initiator's gas costs), even if the weight-based calculation (`debt * initiationWeight / ONE_4`) yields a smaller amount for low-debt Accounts.

**What would go wrong if it stopped doing so**

By deleting this floor enforcement, `initiationReward` can end up below `minReward` whenever `debt.mulDivDown(initiationWeight, ONE_4) < minReward`. This is exactly the case documented in the code comments: 'minimumMargin should be set big enough such that minimumMargin * minRewardWeight can cover any possible gas cost to initiate/terminate the liquidation.' If the reward calculation is not floored, liquidators may receive rewards that do not cover their gas costs, especially for Accounts close to the minimumMargin threshold. Rational liquidators will decline to call startLiquidation() on such Accounts, allowing undercollateralized positions to remain open, accrue further losses, and eventually generate bad debt that must be socialized across tranches (via _processDefault). This breaks a core economic safety invariant of the lending pool (guaranteed liquidation incentive), which underlies protocol solvency.

**Attack scenario (hypothetical — requires the change above)**

1. Protocol owner configures initiationWeight, minRewardWeight and minimumMargin under the assumption the floor logic works as documented. 2. An Account accrues debt just above minimumMargin, becoming eligible for liquidation, but with `debt * initiationWeight / ONE_4` computing to less than `minReward` (e.g. small debt, low initiationWeight). 3. Due to the mutation, `initiationReward` is left at the smaller weight-based value instead of being raised to `minReward`. 4. Because the reward is insufficient to cover the gas cost of calling `startLiquidation`, no external party (bot/keeper) is incentivized to trigger liquidation. 5. The undercollateralized Account remains open, its debt/losses grow, and when eventually liquidated (if ever) the protocol absorbs a larger bad debt via `_processDefault`, harming LPs and the treasury. This is a passive/indirect exploit (denial of liquidation incentive) rather than direct fund extraction, but it undermines protocol solvency over time.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1211,7 +1211,8 @@
 
         // Initiation reward must be between minReward and maxReward.
         initiationReward = debt.mulDivDown(initiationWeight, ONE_4);
-        initiationReward = initiationReward > minReward ? initiationReward : minReward;
+        /// DeleteExpressionMutation(`initiationReward = initiationReward > minReward ? initiationReward : minReward` |==> `assert(true)`) of: `initiationReward = initiationReward > minReward ? initiationReward : minReward;`
+        assert(true);
         initiationReward = initiationReward > maxReward_ ? maxReward_ : initiationReward;
 
         // Termination reward must be between minReward and maxReward.
```

**Recommended test**

Add a unit test for `_calculateRewards` (or an integration test around `startLiquidation`) that sets `debt` and `initiationWeight` such that `debt.mulDivDown(initiationWeight, ONE_4) < minReward`, and asserts that the returned `initiationReward` equals `minReward` (bounded by `maxReward`). The current test suite apparently only exercises cases where the weight-based reward already exceeds `minReward`, so it never asserts the floor behavior; a targeted low-debt test case would kill this mutant.

---

### 22. [HIGH] setRiskManager silently fails to update risk manager

**Location**: `src/LendingPool.sol:1289` in `setRiskManager()`  
**Type**: Logic Error / Access Control  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 543)

**What the code is supposed to enforce**

The setRiskManager function is intended to call _setRiskManager(riskManager_) (inherited from Creditor) to update the protocol's risk manager address, which is a privileged role responsible for setting collateral/risk parameters and managing account version validity.

**What would go wrong if it stopped doing so**

The mutation replaces the call to _setRiskManager(riskManager_) with a no-op assert(true). As a result, calling setRiskManager() as the owner has zero effect: the risk manager address is never updated, no state change occurs, and no revert happens (so the transaction appears to succeed). This breaks a critical administrative control path. If the current risk manager needs to be rotated (e.g., due to key compromise, migration, or decommissioning of an old risk manager contract), the owner's fix silently fails, leaving the old/compromised risk manager in control. Since the risk manager can set collateral factors, LTV ratios, and asset risk parameters (in the broader Arcadia protocol), a stale or malicious risk manager continuing to operate can be leveraged to manipulate risk parameters, enabling undercollateralized borrowing or blocking legitimate liquidations, ultimately leading to protocol insolvency or fund loss.

**Attack scenario (hypothetical — requires the change above)**

1. Protocol governance discovers that the current riskManager address has been compromised (e.g., its private key leaked) or needs replacement due to a bug. 2. The owner calls setRiskManager(newSafeRiskManager) expecting the risk manager role to be transferred. 3. Due to the mutation, the call executes assert(true) and returns successfully, but the underlying riskManager storage variable in the Creditor base contract is never updated. 4. The compromised/old risk manager retains full privileges and can set malicious risk parameters (e.g., artificially inflate collateral values or disable liquidation thresholds) via its still-active role. 5. Attackers exploit the manipulated risk parameters to borrow more than their collateral supports or to prevent liquidation of undercollateralized positions, resulting in bad debt and protocol losses that are socialized to LPs and the treasury.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1286,7 +1286,8 @@
      * @param riskManager_ The address of the new Risk Manager.
      */
     function setRiskManager(address riskManager_) external onlyOwner {
-        _setRiskManager(riskManager_);
+        /// DeleteExpressionMutation(`_setRiskManager(riskManager_)` |==> `assert(true)`) of: `_setRiskManager(riskManager_);`
+        assert(true);
     }
 
     /**
```

**Recommended test**

Add a unit test for setRiskManager() that (a) calls the function as owner with a new address, and (b) asserts that the contract's riskManager() (or equivalent public getter/state variable inherited from Creditor) equals the new address afterward. Additionally, add a test verifying that an old risk manager loses its privileges (e.g., attempting a risk-manager-only action after rotation reverts). This would kill the mutant since the state change would no longer occur.

---

### 23. [MEDIUM] Surplus miscalculated via modulo instead of subtraction in auctionRepay

**Location**: `src/LendingPool.sol:529` in `auctionRepay()`  
**Type**: Logic Error / Arithmetic Miscalculation  
**Confidence**: MEDIUM  
**Mutation**: `BinaryOpMutation` (id 133)

**What the code is supposed to enforce**

When a liquidation auction bid (amount) fully covers the Account's outstanding debt (accountDebt), the excess funds (amount - accountDebt) constitute a surplus that must be credited back to the Account owner via _settleLiquidationHappyFlow's surplus parameter, ensuring correct fund accounting between the bidder's payment, the debt repaid, and the owner's rightful residual.

**What would go wrong if it stopped doing so**

The mutation replaces subtraction with modulo (amount % accountDebt). Both operations coincide only when accountDebt <= amount < 2*accountDebt (quotient = 1), so many typical small-overpayment auctions are unaffected. However, whenever the bid amount is at least twice the outstanding debt (e.g., amount = 2*accountDebt or higher), the modulo yields a value far smaller than the true surplus (even zero when amount is an exact multiple of accountDebt). Since the full 'amount' was already pulled from the bidder via safeTransferFrom, but only the (incorrect, smaller) modulo-based surplus is credited to totalRealisedLiquidity/realisedLiquidityOf[owner], the difference between the real surplus and the computed one becomes untracked, stuck ERC20 balance in the contract. This breaks the core invariant that all transferred-in assets are accounted for in realisedLiquidityOf balances, causing the Account owner to lose their rightful surplus (up to the entire surplus amount if amount is an exact multiple of accountDebt).

**Attack scenario (hypothetical — requires the change above)**

1. An Account undergoes liquidation with accountDebt = X.
2. During the auction, assets are sold for a bid such that the Liquidator calls auctionRepay with amount = 2X (or 3X, etc.), e.g., due to strong market demand for the collateral or a large single bid covering more than the debt.
3. Since accountDebt <= amount, the happy-flow branch executes: with the mutation, surplus = amount % accountDebt = 0 (when amount = 2X exactly) instead of the correct surplus = amount - accountDebt = X.
4. _settleLiquidationHappyFlow credits 0 (or an incorrect smaller value) to totalRealisedLiquidity and to realisedLiquidityOf[accountOwner], even though the contract's ERC20 balance increased by the full 'amount' from the bidder.
5. The Account owner receives none (or far less) of their rightful surplus; the leftover funds sit unaccounted for in the contract, effectively benefiting whoever later calls skim() (which routes any balance surplus vs. totalRealisedLiquidity to the treasury) rather than the liquidated owner.
6. Net effect: liquidated users are silently deprived of surplus funds they are entitled to under specific (but realistic) auction outcomes.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -526,7 +526,8 @@
             // -> Terminate the auction and make the surplus available to the Account-Owner.
             earlyTerminate = true;
             unchecked {
-                _settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount - accountDebt));
+                /// BinaryOpMutation(`-` |==> `%`) of: `_settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount - accountDebt));`
+                _settleLiquidationHappyFlow(account, startDebt, minimumMargin_, bidder, (amount%accountDebt));
             }
             amount = accountDebt;
         }
```

**Recommended test**

Add a unit test for auctionRepay/_settleLiquidationHappyFlow where amount is set to at least 2x accountDebt (e.g., amount = 3*accountDebt) and assert that realisedLiquidityOf[accountOwner] increases by exactly (amount - accountDebt), and that totalRealisedLiquidity accounts for the full transferred amount. This case is not covered by existing tests that likely only test amount slightly greater than accountDebt (where % and - coincide).

> **Reviewer note**: Downgraded from HIGH on hand review: this mutant is NEAR-EQUIVALENT. Where `accountDebt <= amount < 2*accountDebt`, `amount % accountDebt` equals `amount - accountDebt` exactly. It only diverges if the auction recovers more than twice the debt.

---

### 24. [MEDIUM] Incorrect tranche indexing corrupts auction-lock state during cascading default

**Location**: `src/LendingPool.sol:1151` in `_processDefault()`  
**Type**: Logic Error / Broken Invariant (Incorrect Index Computation)  
**Confidence**: MEDIUM  
**Mutation**: `BinaryOpMutation` (id 480)

**What the code is supposed to enforce**

In `_processDefault`, when a tranche is completely wiped out by bad debt and popped from the `tranches` array, the function must mark the *new* most-junior tranche (now at index `i-1` after the pop) as having an auction in progress via `setAuctionInProgress(true)`. This prevents deposits/withdrawals into the tranche that is now bearing further potential losses, preserving the pro-rata loss-absorption invariant across tranches.

**What would go wrong if it stopped doing so**

The mutation replaces `tranches[i - 1]` with `tranches[i % 1]`. Since the branch is only reached when `i != 0` (i.e., `i >= 1`), `i % 1` always evaluates to `0`, regardless of the actual value of `i`. As a result: (1) whenever more than one tranche is wiped out in a cascading default (i.e., `i > 1` so that the correct target `i-1 != 0`), the wrong tranche (always the most senior tranche at index 0) is flagged with `setAuctionInProgress(true)` instead of the tranche that actually needs to be locked; (2) the correct next-most-junior tranche never gets its auction-in-progress flag set, so LPs of that tranche can deposit or withdraw while the tranche is still exposed to being wiped out or otherwise implicated in the ongoing liquidation/default resolution; (3) the senior tranche (index 0) gets erroneously locked out of deposits/withdrawals, and because `_endLiquidation()` only resets the flag on the tranche at `tranches[tranches.length-1]` (the real last index), tranche 0's incorrect `true` flag is never cleared through the normal liquidation-ending flow, causing a persistent, unintended freeze of deposits/withdrawals for the most senior tranche's LPs.

**Attack scenario (hypothetical — requires the change above)**

1. Lending pool has 3+ tranches (e.g., senior=0, mezzanine=1, junior=2). 2. A severe under-collateralization event occurs and `settleLiquidationUnhappyFlow` is called with `badDebt` large enough to fully wipe out both the junior tranche (index 2) and the mezzanine tranche (index 1) in the same `_processDefault` call. 3. When tranche 2 (junior) is wiped: loop sets i=2, `_popTranche(2, tranche)` pops it, then attempts to mark the new last tranche in progress: correct behavior would target index `i-1=1` (mezzanine), but the mutant computes `i % 1 = 0` and calls `setAuctionInProgress(true)` on tranche 0 (senior) instead. 4. The mezzanine tranche (now the last tranche in the array) never gets its auction flag set, so its LPs can freely withdraw liquidity even though bad debt processing continues to affect it in this same cascading default, or during the remainder of the liquidation window before `_endLiquidation()` is called. Meanwhile, the senior tranche is incorrectly locked from deposits/withdrawals — and since `_endLiquidation()` will only unset the flag on the actual last tranche address (not index 0), the senior tranche remains stuck with auctionInProgress == true, permanently denying its LPs access to deposit/withdraw functionality until manually remedied by governance/another liquidation flow that happens to touch tranche 0.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1148,7 +1148,8 @@
                 ITranche(tranche).lock();
                 // Hook to the new most junior Tranche to inform that auctions are ongoing.
                 // forge-lint: disable-next-item(reentrancy-no-eth)
-                if (i != 0) ITranche(tranches[i - 1]).setAuctionInProgress(true);
+                /// BinaryOpMutation(`-` |==> `%`) of: `if (i != 0) ITranche(tranches[i - 1]).setAuctionInProgress(true);`
+                if (i != 0) ITranche(tranches[i%1]).setAuctionInProgress(true);
             }
         }
     }
```

**Recommended test**

Add a unit test for `_processDefault` (or an integration test via `settleLiquidationUnhappyFlow`) with at least 3 tranches where `badDebt` is large enough to fully wipe out two or more junior tranches in a single call. Assert that `setAuctionInProgress(true)` is called on the correct tranche address at index `i-1` after each pop (e.g., verify via mock Tranche capturing call arguments), and assert that the tranche at index 0 is NOT flagged unless it is genuinely the new most-junior tranche. Also add a regression test checking that after `_endLiquidation()`, no unintended tranche remains with `auctionInProgress == true`.

> **Reviewer note**: Downgraded from HIGH on hand review. `i % 1` is always 0, so the wrong tranche is locked: a functional bug with no fund impact.

---

### 25. [MEDIUM] processInterests modifier no longer updates interest rate after actions

**Location**: `src/LendingPool.sol:180` in `processInterests()`  
**Type**: Logic Error / Stale State  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 2)

**What the code is supposed to enforce**

After syncing interests and executing the wrapped function (borrow, repay, deposit, withdraw, liquidation, etc.), the modifier recalculates and updates the pool's interestRate based on the new realisedDebt and totalRealisedLiquidity, keeping the utilisation-based interest rate accurate after every state-changing operation.

**What would go wrong if it stopped doing so**

With the call to _updateInterestRate removed and replaced by a no-op assert(true), the interestRate storage variable is never updated again after this mutation is applied to any function using the processInterests modifier. This means utilisation changes from borrowing, repaying, depositing, withdrawing, or liquidations no longer affect the interest rate curve. The rate becomes permanently stale (frozen at whatever value it had before deployment or last legitimate update), decoupling the interest charged to borrowers from actual pool utilisation. Over time this leads to: (1) borrowers being charged an interest rate divorced from real utilisation (potentially far too low when utilisation is high, incentivizing excess borrowing and risking pool insolvency for LPs; or too high when utilisation is low, discouraging borrowing and disadvantaging users), (2) mispriced risk for liquidity providers who rely on the interest curve to compensate for utilisation risk, and (3) potential for attackers to borrow heavily while interest rate remains artificially low, extracting value from LPs over time. This is a protocol-wide economic invariant violation rather than a single-transaction fund drain, but it can lead to gradual value extraction and mispricing across the whole lending pool.

**Attack scenario (hypothetical — requires the change above)**

1. Attacker (or any borrower) notices interestRate is stuck at a low value (e.g. from before pool was heavily utilised).
2. Attacker borrows the maximum amount possible against collateral in an Account, driving pool utilisation to a high percentage.
3. Normally, _updateInterestRate would sharply raise the rate (especially past utilisationThreshold using highSlopePerYear), discouraging further borrowing and compensating LPs for elevated risk.
4. Because the rate is frozen, the attacker continues borrowing at the artificially low rate even as utilisation approaches 100%, extracting cheap leverage while LPs bear undercompensated risk.
5. If utilisation stays high for a long duration, LPs earn far less interest than the risk they are taking, and in the event of defaults, the mispriced low rate exacerbates realized bad debt since the highSlope premium meant to discourage risky utilisation never kicks in.
6. Repeated over time, this results in gradual value transfer from LPs/treasury to borrowers.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -177,7 +177,8 @@
         _;
         // _updateInterestRate() modifies the state (effect), but can safely be called after interactions.
         // Cannot be exploited by reentrancy attack.
-        _updateInterestRate(realisedDebt, totalRealisedLiquidity);
+        /// DeleteExpressionMutation(`_updateInterestRate(realisedDebt, totalRealisedLiquidity)` |==> `assert(true)`) of: `_updateInterestRate(realisedDebt, totalRealisedLiquidity);`
+        assert(true);
     }
 
     /* //////////////////////////////////////////////////////////////
```

**Recommended test**

Add a test that calls a processInterests-wrapped function (e.g., borrow) that changes utilisation, then asserts that `interestRate` (public getter) changes to the value predicted by `_calculateInterestRate` based on new totalDebt/totalRealisedLiquidity. E.g., in LendingPoolTest, after calling `borrow()` with parameters that push utilisation above/below threshold, assert `lendingPool.interestRate() == expectedRate` computed independently, which would fail under the mutant since interestRate remains unchanged.

---

### 26. [MEDIUM] Missing Tranche.lock() call after tranche wipeout in _processDefault

**Location**: `src/LendingPool.sol:1148` in `_processDefault()`  
**Type**: Missing State Synchronization / Broken Invariant  
**Confidence**: LOW  
**Mutation**: `DeleteExpressionMutation` (id 475)

**What the code is supposed to enforce**

When bad debt fully wipes out the most junior Tranche's realised liquidity, the Tranche is removed from the LendingPool's active tranches (isTranche=false, interestWeight=0, tranches.pop()) and the external ITranche(tranche).lock() call is meant to permanently freeze the Tranche contract itself (disabling deposits/mints/redeems/transfers at the Tranche level) so that liquidity providers and third parties cannot continue to interact with a Tranche whose underlying economic value has been wiped to zero.

**What would go wrong if it stopped doing so**

By replacing `ITranche(tranche).lock()` with a no-op `assert(true)`, the Tranche contract is never notified that it has been defaulted and popped from the LendingPool. While the LendingPool's own `onlyTranche` modifier will reject any subsequent `depositInLendingPool` call from this tranche (since isTranche[tranche] is now false), the Tranche contract itself retains no internal state change: its own deposit/mint/withdraw/redeem entry points, share transfers, and any Tranche-level guardian/pause logic remain exactly as they were before default. This breaks the intended defense-in-depth invariant that a defaulted Tranche is permanently frozen at the source, relying solely on the LendingPool-side mapping to prevent misuse. Any additional Tranche-level logic that depends on the `locked` flag (e.g., blocking share transfers of now-worthless tokens, blocking mint() calls that don't route through depositInLendingPool, or off-chain integrations checking Tranche.locked()) will continue operating as if the tranche were healthy, potentially allowing LPs to trade/transfer worthless shares or trust misleading contract state.

**Attack scenario (hypothetical — requires the change above)**

1. A catastrophic under-collateralization event occurs causing badDebt to exceed the realised liquidity of the most junior Tranche. 2. _processDefault wipes the tranche's realisedLiquidityOf to 0 and pops it from the tranches array (isTranche=false), but due to the mutation, ITranche(tranche).lock() is never invoked. 3. The Tranche contract, unaware it has been defaulted, still reports itself as active/unlocked. Depending on Tranche implementation, LPs may still call Tranche.deposit()/mint() (which will revert only because of the LendingPool onlyTranche check, not because the Tranche itself blocks it) or may transfer/sell their now-worthless Tranche shares to unsuspecting third parties who believe the Tranche is still operative, or interact with any Tranche-level function gated only by its own `locked` flag rather than by calling into the LendingPool. 4. This can result in economic loss for parties who acquire or interact with shares of a Tranche that has already been wiped out, and creates an inconsistency between the LendingPool's and the Tranche's view of protocol state.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1145,7 +1145,8 @@
                     badDebt -= maxBurnable;
                 }
                 // forge-lint: disable-next-item(reentrancy-no-eth)
-                ITranche(tranche).lock();
+                /// DeleteExpressionMutation(`ITranche(tranche).lock()` |==> `assert(true)`) of: `ITranche(tranche).lock();`
+                assert(true);
                 // Hook to the new most junior Tranche to inform that auctions are ongoing.
                 // forge-lint: disable-next-item(reentrancy-no-eth)
                 if (i != 0) ITranche(tranches[i - 1]).setAuctionInProgress(true);
```

**Recommended test**

Add a unit test for `_processDefault`/`settleLiquidationUnhappyFlow` that triggers a full wipeout of the most junior Tranche and asserts that `ITranche(tranche).locked()` (or equivalent state-reading function) returns true afterward, and/or use a mock ITranche that records whether `lock()` was called, asserting `mockTranche.lockCalled() == true` after the wipeout path executes. This would directly kill the mutant since `assert(true)` never triggers the external call.

---

### 27. [MEDIUM] Missing auction-lock hook allows withdrawal from newly exposed junior tranche

**Location**: `src/LendingPool.sol:1151` in `_processDefault()`  
**Type**: Access Control / Broken Invariant  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 476)

**What the code is supposed to enforce**

When a fully wiped-out (most junior) Tranche is popped during bad-debt write-off cascading in `_processDefault`, the next tranche in line (now the new most-junior Tranche) must be flagged via `setAuctionInProgress(true)` so that deposits and withdrawals into/out of that Tranche are blocked while further liquidation-related bad debt write-offs may still occur against it (e.g. while other auctions are still in progress).

**What would go wrong if it stopped doing so**

The mutation replaces the call `ITranche(tranches[i - 1]).setAuctionInProgress(true)` with a no-op `assert(true)`. As a result, the newly-promoted most-junior Tranche never has its `auctionInProgress` flag set when a lower tranche is wiped out and popped during `_processDefault`. This breaks the invariant that a Tranche which is currently absorbing (or may still absorb) bad debt from an ongoing liquidation must be locked against deposits/withdrawals. LPs in that Tranche can continue to deposit or withdraw normally even though the Tranche is exposed to further bad-debt write-offs from concurrently ongoing auctions, enabling front-running of losses (withdraw before dilution) or unfair share-price manipulation via deposits while the Tranche's true collateral backing is in flux.

**Attack scenario (hypothetical — requires the change above)**

1. Multiple liquidations (auctions) are in progress simultaneously (auctionsInProgress > 1), so LendingPoolGuardian/startLiquidation only locked the single most-junior Tranche at the time the first auction began.
2. One auction settles via `settleLiquidationUnhappyFlow`, and the bad debt is large enough to fully wipe out the current most-junior Tranche (`badDebt >= maxBurnable`); that Tranche is popped and locked via `ITranche(tranche).lock()`.
3. `_processDefault` proceeds to the next Tranche (the new last index) and is supposed to call `setAuctionInProgress(true)` on it because a second (still ongoing) auction may still generate more bad debt that cascades into it.
4. Due to the mutation, this call is skipped, so the newly-promoted junior Tranche remains fully open for deposits/withdrawals.
5. An LP holding a large position in the newly-exposed junior Tranche observes (via mempool monitoring or off-chain knowledge) that further bad debt is likely, and immediately withdraws their liquidity before the second auction settles and writes off additional bad debt against that Tranche.
6. The withdrawing LP avoids taking their proportional share of the loss, and the remaining LPs in that Tranche absorb a disproportionately larger share of the bad debt than intended.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1148,7 +1148,8 @@
                 ITranche(tranche).lock();
                 // Hook to the new most junior Tranche to inform that auctions are ongoing.
                 // forge-lint: disable-next-item(reentrancy-no-eth)
-                if (i != 0) ITranche(tranches[i - 1]).setAuctionInProgress(true);
+                /// DeleteExpressionMutation(`ITranche(tranches[i - 1]).setAuctionInProgress(true)` |==> `assert(true)`) of: `if (i != 0) ITranche(tranches[i - 1]).setAuctionInProgress(true);`
+                if (i != 0) assert(true);
             }
         }
     }
```

**Recommended test**

Add a test that simulates cascading bad debt across two tranches within a single `_processDefault` call (i.e. total badDebt greater than the most junior tranche's realised liquidity but not exceeding the second tranche's), then assert that `ITranche(tranches[i-1]).setAuctionInProgress(true)` was actually invoked — e.g. via a mock Tranche that records calls to `setAuctionInProgress` and asserting `auctionInProgress == true` on the new last tranche after `_processDefault`/`settleLiquidationUnhappyFlow` returns.

---

### 28. [MEDIUM] Missing minReward floor on terminationReward in _calculateRewards

**Location**: `src/LendingPool.sol:1219` in `_calculateRewards()`  
**Type**: Logic Error / Incorrect Fee Calculation  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 520)

**What the code is supposed to enforce**

The original code enforces that the terminationReward (paid to whoever finalizes/terminates a liquidation auction) can never fall below minReward, which is derived from minimumMargin * minRewardWeight. This floor exists specifically to guarantee that the reward covers the gas cost of calling settleLiquidationHappyFlow/settleLiquidationUnhappyFlow, as stated explicitly in the NatSpec: 'The rewards for the initiator and terminator should at least cover the gas costs.'

**What would go wrong if it stopped doing so**

By replacing the floor-clamping ternary with a no-op assert(true), terminationReward is left solely as debt.mulDivDown(terminationWeight, ONE_4), without ever being raised to minReward. For low-debt positions (or when terminationWeight is small relative to debt), the computed terminationReward can be far below the gas cost required to call the terminate functions. Since terminationReward still gets capped by maxReward_ afterward (the second line is untouched), the reward range is now [0, maxReward] instead of [minReward, maxReward]. This breaks the invariant that terminator compensation is bounded below by minReward, undermining the incentive design meant to guarantee liquidations get finalized promptly. It does not directly allow theft of funds from the pool (no unauthorized state change occurs elsewhere), but it can leave debt positions in a state where nobody has sufficient economic incentive to call settleLiquidationHappyFlow/settleLiquidationUnhappyFlow, stalling the resolution of an auction and potentially allowing bad debt to accumulate for longer or requiring an altruistic/subsidized caller.

**Attack scenario (hypothetical — requires the change above)**

1. An attacker (or just organic market conditions) opens a position with just above minimumMargin worth of collateral and a small debt amount such that debt.mulDivDown(terminationWeight, ONE_4) computes to a value much smaller than minReward (e.g., terminationWeight is set low, e.g. 10 (0.1%), and debt is modest, producing a reward of a few wei while minReward, computed from minimumMargin*minRewardWeight, would normally be e.g. 1e18 wei).
2. The position becomes liquidatable and startLiquidation() is called, minting initiationReward+penalty+terminationReward as debt, with initiationReward properly floored but terminationReward not.
3. After the auction concludes (happy or unhappy flow), the Liquidator contract calls settleLiquidationHappyFlow/UnhappyFlow with a terminator address.
4. The terminator receives a terminationReward far below the gas cost of calling terminate, functionally to zero, making it economically irrational for anyone to call the termination function.
5. The auction remains unfinished (auctionsInProgress stays elevated), locking the most junior Tranche (setAuctionInProgress(true) remains active) and delaying release of funds/bad debt handling, potentially exposing further Junior Tranche depositors to prolonged risk or requiring the protocol to manually intervene/subsidize termination calls.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1216,7 +1216,8 @@
 
         // Termination reward must be between minReward and maxReward.
         terminationReward = debt.mulDivDown(terminationWeight, ONE_4);
-        terminationReward = terminationReward > minReward ? terminationReward : minReward;
+        /// DeleteExpressionMutation(`terminationReward = terminationReward > minReward ? terminationReward : minReward` |==> `assert(true)`) of: `terminationReward = terminationReward > minReward ? terminationReward : minReward;`
+        assert(true);
         terminationReward = terminationReward > maxReward_ ? maxReward_ : terminationReward;
 
         liquidationPenalty = debt.mulDivUp(penaltyWeight, ONE_4);
```

**Recommended test**

Add a unit test for _calculateRewards (or an integration test around settleLiquidationHappyFlow/UnhappyFlow) with a small startDebt/minimumMargin_ combination that yields debt.mulDivDown(terminationWeight, ONE_4) < minReward, and assert that the returned terminationReward equals minReward (not the smaller raw calculation). This exercises the removed floor branch and would fail against the mutant since terminationReward would remain unclamped.

---

### 29. [LOW] Stale interestWeight mapping entry after tranche removal in _popTranche

**Location**: `src/LendingPool.sol:281` in `_popTranche()`  
**Type**: State Inconsistency / Improper Cleanup  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 31)

**What the code is supposed to enforce**

When a Tranche is completely wiped out by bad debt and removed via _popTranche, the code is meant to fully reset all state associated with that tranche: unregister it as a tranche (isTranche=false), remove it from the interestWeightTranches array and tranches array, and zero its entry in the interestWeight mapping so that no residual accounting data remains associated with the removed tranche address.

**What would go wrong if it stopped doing so**

The mutation deletes the `interestWeight[tranche] = 0` assignment, replacing it with a no-op `assert(true)`. As a result, the interestWeight mapping entry for the removed tranche retains its old (non-zero) value even though the tranche has been unregistered, its realisedLiquidityOf balance zeroed, and totalInterestWeight already decremented by its share. Because totalInterestWeight was already correctly reduced in the same function (top of _popTranche), the stale mapping value does not corrupt the interest distribution among the remaining active tranches. Its only observable effect is that the view function `liquidityOf(trancheAddress)` will compute a spurious non-zero `interest` term (`calcUnrealisedDebt() * interestWeight[tranche] / totalInterestWeight`) for the removed tranche address as long as calcUnrealisedDebt() and totalInterestWeight remain positive, even though the tranche is locked and holds zero realised liquidity. If the same tranche address is later re-added via addTranche(), the mapping entry is explicitly overwritten, eliminating the stale value. No function that moves funds (withdrawFromLendingPool, repay, borrow, liquidation settlement) relies on this mapping entry for the removed address — those rely on realisedLiquidityOf, which is correctly zeroed. Thus the bug is a real state-hygiene defect but has no direct path to fund loss or unauthorized minting under the current contract interfaces.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -278,7 +278,8 @@
         isTranche[tranche] = false;
         interestWeightTranches.pop();
         tranches.pop();
-        interestWeight[tranche] = 0;
+        /// DeleteExpressionMutation(`interestWeight[tranche] = 0` |==> `assert(true)`) of: `interestWeight[tranche] = 0;`
+        assert(true);
 
         emit TranchePopped(tranche);
     }
```

**Recommended test**

Add a unit test that triggers _popTranche (e.g., via _processDefault with badDebt >= realisedLiquidityOf of the most junior tranche), then asserts that interestWeight[poppedTranche] == 0 afterward — e.g., by calling liquidityOf(poppedTrancheAddress) with a nonzero calcUnrealisedDebt() and totalInterestWeight and asserting the returned value equals realisedLiquidityOf[poppedTrancheAddress] (i.e., no phantom interest term is added). This would fail against the mutant since the stale non-zero interestWeight entry would produce a non-zero interest contribution.

---

### 30. [LOW] Stale reward values emitted in AuctionFinished during bad-debt liquidation

**Location**: `src/LendingPool.sol:1061` in `settleLiquidationUnhappyFlow()`  
**Type**: Logic Error / Incorrect Event Data  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 455)

**What the code is supposed to enforce**

When openDebt exceeds the combined terminationReward and liquidationPenalty (i.e., the debt is fully absorbed as bad debt), the original code resets terminationReward and liquidationPenalty to 0 to correctly reflect that none of these incentives were actually paid out — all available value went to covering bad debt instead.

**What would go wrong if it stopped doing so**

The mutation replaces the reset `(terminationReward, liquidationPenalty) = (0, 0)` with a no-op `assert(true)`, leaving the two local variables at their pre-calculated (non-zero) values from `_calculateRewards`. Tracing forward, these variables are used exclusively in the final `emit AuctionFinished(...)` call — no `realisedLiquidityOf` updates, transfers, or further state mutations depend on them in this branch. Therefore no funds are misallocated and core accounting invariants (totalRealisedLiquidity, realisedDebt, tranche balances) remain correct. The only observable effect is that the emitted `AuctionFinished` event will report non-zero `terminationReward`/`liquidationPenalty` values even though these were not actually paid to anyone, corrupting off-chain analytics, dashboards, or any downstream system/contract that relies on event data to reconstruct protocol state or accounting.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -1058,7 +1058,8 @@
             // forge-lint: disable-next-line(unsafe-typecast)
             totalRealisedLiquidity = uint128(totalRealisedLiquidity - badDebt);
             _processDefault(badDebt);
-            (terminationReward, liquidationPenalty) = (0, 0);
+            /// DeleteExpressionMutation(`(terminationReward, liquidationPenalty) = (0, 0)` |==> `assert(true)`) of: `(terminationReward, liquidationPenalty) = (0, 0);`
+            assert(true);
         } else {
             uint256 remainder = liquidationPenalty + terminationReward - openDebt;
             if (openDebt >= liquidationPenalty) {
```

**Recommended test**

Add a unit test for `settleLiquidationUnhappyFlow` that forces the bad-debt branch (openDebt > terminationReward + liquidationPenalty), then asserts that the emitted `AuctionFinished` event carries `terminationReward == 0` and `liquidationPenalty == 0`, and separately verifies that `realisedLiquidityOf[terminator]` and treasury/tranche balances were NOT incremented by the pre-reset reward amounts.

---

## Test gaps with no security impact

Mutations the suite does not catch, assessed as not exploitable.

| Location | Function | Assessment |
|---|---|---|
| `src/LendingPool.sol:456` | `borrow` | Since `amount > 0` is enforced by the ZeroAmount check earlier in `borrow()`, `amountWithFee + amount` is always strictly positive, making… |
| `src/LendingPool.sol:456` | `borrow` | The mutated condition is always true whenever this code path is reached (since amount>0 is guaranteed by the earlier ZeroAmount check and a… |
| `src/LendingPool.sol:456` | `borrow` | The mutated condition changes truth value only in the fee==0 case, but the guarded operations both reduce to adding zero to state variables… |
| `src/LendingPool.sol:456` | `borrow` | Because the subtraction happens inside `unchecked { ... }`, Solidity does not revert on underflow but wraps around modulo 2^256. Given `amo… |
| `src/LendingPool.sol:474` | `borrow` | This mutation only affects the value emitted in the Borrow event, which is a logging mechanism with no bearing on contract state, token bal… |
| `src/LendingPool.sol:474` | `borrow` | The mutation only corrupts an emitted event parameter used for off-chain indexing/monitoring; it does not alter any on-chain state, fund tr… |
| `src/LendingPool.sol:610` | `_flashActionCallback` | Because origination fee (amountBorrowedWithFee - amountBorrowed) is always non-negative, the mutated condition (sum > 0) and the original c… |
| `src/LendingPool.sol:610` | `_flashActionCallback` | For all practically achievable input magnitudes, the mutated condition either matches the original's true/false outcome, or (in the sole di… |
| `src/LendingPool.sol:610` | `_flashActionCallback` | The condition change is provably equivalent: for fee=0, both original and mutant conditions evaluate to false (0 is not >0). For fee>0, the… |

## Methodology

Each mutant is a single deliberate change to the source — an inverted comparison, a deleted statement, swapped arguments. The mutant is compiled and the full test suite runs against it.

- A mutant that makes a test fail is **killed**: the suite covers that behaviour.
- A mutant that leaves every test passing **survived**: nothing in the suite checks that behaviour.
- A mutant that **does not compile** is excluded from the score. It says nothing about test quality, and counting it as killed would inflate the result.

Surviving mutants are then assessed individually against the full contract source to separate real test gaps from changes with no reachable impact.
