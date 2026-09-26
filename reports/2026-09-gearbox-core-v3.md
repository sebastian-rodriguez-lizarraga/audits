# Mutation Audit — Gearbox Protocol (core-v3)

**Repository**: [Gearbox-protocol/core-v3](https://github.com/Gearbox-protocol/core-v3)  
**Commit**: `510fc65`  
**Scope**: `contracts/pool/PoolV3.sol` · 343 tests in the suite  
**Date**: 2026-09-25  
**Method**: Gambit (mutant generation) + Foundry (compile & test) + Claude (survivor assessment)

> **This is a strong suite.** 220 mutations, 97.7% killed — the second highest
> of the four protocols measured so far. What follows is not a criticism of the
> test suite; it is the small residue that survived one of the better ones.

> **Test-run note.** One integration test,
> `test_I_CC_31_addCollateralToken_works_with_state_changing_fallback`, reverts
> on a clean checkout with the current Foundry toolchain and was excluded from
> every run, baseline included. The remaining 343 tests pass. A red baseline
> would make every mutant exit non-zero and report a meaningless ~100% score,
> so it has to be excluded rather than tolerated.

> **Scope note.** `PoolV3.sol` is 832 of roughly 7,900 lines of non-interface
> contract code in this repository — about 11%. The eight remaining contracts,
> including `CreditManagerV3.sol` (1,398 lines) and `CreditFacadeV3.sol`
> (1,147), were not measured.

## The headline: an assertion that checks the wrong mapping entry

`PoolV3.sol:371` carries the annotation `// U:[LP-8,9]`, naming the two tests
that cover it:

```solidity
if (msg.sender != owner) _spendAllowance({owner: owner, spender: msg.sender, amount: shares}); // U:[LP-8,9]
```

Both tests do assert on the allowance
(`PoolV3.unit.t.sol:643` and `:738`):

```solidity
vm.prank(user);
pool.withdraw({assets: cases[i].assets, receiver: owner, owner: owner});

assertEq(pool.allowance(user, owner), 0, "Incorrect shares allowance");
```

ERC20's signature is `allowance(owner, spender)`. The withdrawal runs with
`msg.sender == user` and the `owner` parameter set to `owner`, so the contract
decrements `allowance[owner][user]`. The assertion reads `allowance[user][owner]`
— the opposite entry of the mapping, which is never written and is zero whether
or not `_spendAllowance` runs.

So the line is annotated as covered, the test exists, the assertion exists, and
deleting `_spendAllowance` entirely leaves all 343 tests passing. Without it,
any caller can withdraw another account's shares without an allowance.

This is the shape of finding that line coverage cannot reach: the line runs,
and nothing checks what it did.

---

## Summary

**Mutation score: 97.7%** — the test suite detected 215 of 220 behaviour-changing mutations.

| | Count |
|---|---:|
| Mutants generated | 220 |
| Killed by the suite | 215 |
| **Survived (undetected)** | **5** |

### Undefended invariants by severity

> Severity is the impact **if this behaviour broke and shipped unnoticed** — not a bug in the code as written. The code is correct today. The finding is that nothing in the test suite would catch it if it stopped being correct.

| Severity | Count |
|---|---:|
| CRITICAL | 2 |
| HIGH | 1 |
| MEDIUM | 1 |
| INFORMATIONAL | 1 |

**Least defended**: CRITICAL — Missing allowance check allows unauthorized withdrawal of shares (`contracts/pool/PoolV3.sol:371`)

---

## Undefended invariants

Each entry is a change to your contracts that the suite does not detect. The contract is correct as written; what follows is the test that is missing.

### 1. [CRITICAL] Missing allowance check allows unauthorized withdrawal of shares

**Location**: `contracts/pool/PoolV3.sol:371` in `_withdraw()`  
**Type**: Access Control  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 29)

**What the code is supposed to enforce**

When msg.sender is not the owner of the shares being withdrawn/redeemed, the code must spend (decrement) the ERC20 allowance granted by owner to msg.sender via _spendAllowance, reverting if insufficient allowance exists. This enforces that only accounts explicitly approved by the owner can withdraw/redeem on their behalf.

**What would go wrong if it stopped doing so**

The mutation replaces the _spendAllowance call with a no-op assert(true), completely removing the allowance check and decrement when msg.sender != owner. This means any external account can call withdraw() or redeem() specifying an arbitrary owner address and receiver address, burn the owner's pool shares, and receive the underlying assets, without ever having been granted an allowance. This breaks the core ERC-4626/ERC20 access control invariant and enables complete theft of all depositors' funds from the pool.

**Attack scenario (hypothetical — requires the change above)**

1. Victim deposits assets into PoolV3 and holds pool shares (owner = victim, no allowance granted to anyone).
2. Attacker calls pool.withdraw(assets, attacker_address, victim_address) or pool.redeem(shares, attacker_address, victim_address).
3. Inside _withdraw, msg.sender (attacker) != owner (victim), so the code attempts _spendAllowance but it has been replaced with assert(true) — no revert occurs even though allowance is zero.
4. _burn(victim, shares) executes, burning the victim's shares.
5. Underlying assets are transferred to attacker_address (receiver).
6. Attacker repeats this for every liquidity provider in the pool, draining all pool funds.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -368,7 +368,8 @@
         uint256 shares
     ) internal {
         if (assetsReceived == 0 || shares == 0) revert AmountCantBeZeroException(); // U:[LP-5B]
-        if (msg.sender != owner) _spendAllowance({owner: owner, spender: msg.sender, amount: shares}); // U:[LP-8,9]
+        /// DeleteExpressionMutation(`_spendAllowance({owner: owner, spender: msg.sender, amount: shares})` |==> `assert(true)`) of: `if (msg.sender != owner) _spendAllowance({owner: owner, spender: msg.sender, amount: shares}); // U:[LP-8,9]`
+        if (msg.sender != owner) assert(true); // U:[LP-8,9]
         _burn(owner, shares); // U:[LP-8,9]
 
         _updateBaseInterest({
```

**Recommended test**

Add a unit test that: (1) has account A deposit and receive shares, (2) has account B (with zero allowance from A) call withdraw(assets, B, A) or redeem(shares, B, A), and (3) asserts the transaction reverts with ERC20InsufficientAllowance or similar. Also add a positive test verifying that after an approved allowance, spending decrements allowance by the exact number of shares withdrawn, and that withdrawing more than the allowance reverts.

---

### 2. [CRITICAL] _transfer override deletes actual token transfer logic

**Location**: `contracts/pool/PoolV3.sol:794` in `_transfer()`  
**Type**: Logic Error / Broken Core Functionality  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 186)

**What the code is supposed to enforce**

The _transfer override is meant to enforce the whenNotPaused modifier while still performing the standard ERC20 balance transfer via super._transfer(from, to, amount), which updates the sender's and receiver's balances.

**What would go wrong if it stopped doing so**

With the mutation, the call to super._transfer is replaced with a no-op assert(true). This means that ERC20 transfer(), transferFrom(), and any internal _mint/_burn/_transfer-driven balance updates (including those used by deposit, withdraw, mint, redeem via _mint/_burn which are separate, but explicit transfer() and transferFrom() calls by users) will no longer move any balance between accounts. Effectively, pool shares (LP tokens) can never be transferred via the standard ERC20 interface: calling transfer or transferFrom will succeed (emitting Transfer event via the base _transfer's internal emit, but since super._transfer body never executes, no event either) without changing any balances. This breaks a core ERC20/ERC4626 invariant: token transfers must actually move balances. This creates several severe issues: (1) users can call transfer(to, amount) repeatedly without ever decreasing their own balance while still appearing to succeed to callers who don't check balances directly (though since no Transfer event is emitted, most integrators checking balanceOf would notice discrepancy, but on-chain composability e.g. other contracts trusting transfer() return value could be exploited); (2) more critically, this breaks the ERC4626 vault's share semantics used throughout DeFi integrations, letting an attacker who calls transfer to move shares to another address without reducing their own balance, effectively duplicating share balance and being able to redeem/withdraw the same underlying value twice from two different addresses (double-spend of shares). This is a direct path to draining pool funds.

**Attack scenario (hypothetical — requires the change above)**

1. Attacker deposits underlying tokens into the pool and receives pool shares (LP tokens) recorded in their balance.
2. Attacker calls transfer(shares, attacker2) to send all (or a portion) of their shares to a second address they control.
3. Because super._transfer is replaced with assert(true), the internal balance mapping is never updated: attacker's balance remains unchanged and attacker2's balance is never credited... however with the mutation, ERC20 _transfer's require checks and balance updates are skipped entirely, meaning the underlying OpenZeppelin implementation's balance decrement/increment logic never executes. Depending on how the mutation manifests exactly (whether _beforeTokenTransfer hooks or the Transfer event still fire), the practical outcome is that share balances are frozen relative to that call.
4. If a caller composes with other contracts that rely on the return value of transfer (typically true) without verifying balanceOf change, an attacker can fake having transferred shares while retaining full ownership, then redeem/withdraw the same shares via redeem()/withdraw() from their original address while also being credited by the counterparty for the 'transfer' that never actually moved value — enabling a double-spend / fraudulent settlement with any protocol or user that trusts the transfer to have succeeded.
5. Even without composability abuse, this vulnerability breaks the core promise of the ERC-20/ERC-4626 token, halting all normal transfers, redemptions via aggregators, and secondary market trading of pool shares, effectively freezing user funds tied to transfer functionality and enabling protocol-breaking inconsistencies between total supply and actual holder balances.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -791,7 +791,8 @@
         override
         whenNotPaused // U:[LP-2A]
     {
-        super._transfer(from, to, amount);
+        /// DeleteExpressionMutation(`super._transfer(from, to, amount)` |==> `assert(true)`) of: `super._transfer(from, to, amount);`
+        assert(true);
     }
 
     /// @dev Returns amount of token that should be transferred to receive `amount`
```

**Recommended test**

Add a dedicated unit test (e.g. in PoolV3Test) that: (1) has account A deposit and obtain shares, (2) call pool.transfer(B, amount), (3) assert that balanceOf(A) decreased by amount and balanceOf(B) increased by amount, and (4) assert a Transfer event was emitted with correct from/to/amount. This test would fail against the mutant since balances would remain unchanged and no Transfer event would be emitted, killing the mutant.

---

### 3. [HIGH] unpause() no longer unpauses the contract

**Location**: `contracts/pool/PoolV3.sol:772` in `unpause()`  
**Type**: Logic Error / Denial of Service  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 184)

**What the code is supposed to enforce**

The unpause function is meant to call Pausable._unpause() to clear the paused state, restoring deposit, withdrawal, and transfer functionality after an emergency pause.

**What would go wrong if it stopped doing so**

With `_unpause()` replaced by `assert(true)` (a no-op), the contract's paused state is never actually cleared. Once the pool is paused (via `pause()`), it can never be unpaused again through this function, permanently blocking deposits, withdrawals, and transfers (all gated by `whenNotPaused`). This is a denial-of-service on core pool functionality: user funds already deposited become effectively locked (withdraw/redeem revert), and no new deposits/mints can occur. Since pausing can happen in emergency scenarios, this mutation turns a temporary pause into a permanent, irreversible freeze of the pool, which is a severe operational and availability failure even though it does not directly enable theft of funds.

**Attack scenario (hypothetical — requires the change above)**

1. A pausable admin calls `pause()` during a routine or emergency pause (e.g., for a security incident or oracle issue), setting `_paused = true`.
2. Later, an authorized unpausable admin calls `unpause()` intending to restore normal operation.
3. Due to the mutation, `unpause()` executes `assert(true)` instead of `_unpause()`, so `Pausable._paused` remains `true`.
4. All calls to `deposit`, `mint`, `withdraw`, `redeem`, and ERC20 `transfer` (which are gated by `whenNotPaused`) continue to revert.
5. Liquidity providers cannot withdraw their funds, and the pool becomes permanently frozen, effectively locking all deposited capital until a contract upgrade or emergency governance action outside the contract is taken.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -769,7 +769,8 @@
     /// @notice Unpauses contract, can only be called by an account with unpausable admin role
     /// @dev Reverts if contract is already unpaused
     function unpause() external override unpausableAdminsOnly {
-        _unpause();
+        /// DeleteExpressionMutation(`_unpause()` |==> `assert(true)`) of: `_unpause();`
+        assert(true);
     }
 
     /// @dev Sets new total debt limit
```

**Recommended test**

Add a unit test that calls `pause()` then `unpause()` and asserts `paused() == false` afterward, and additionally verifies that `deposit`/`withdraw` succeed post-unpause (currently only pause-state transitions for `pause()` seem tested, not the full pause/unpause round trip with a post-condition check on `paused()`).

---

### 4. [MEDIUM] Withdrawal fee amount computed via division instead of subtraction

**Location**: `contracts/pool/PoolV3.sol:383` in `_withdraw()`  
**Type**: Logic Error / Incorrect Arithmetic Operator  
**Confidence**: HIGH  
**Mutation**: `BinaryOpMutation` (id 36)

**What the code is supposed to enforce**

When withdrawFee > 0, the difference between the total amount debited from the pool (assetsSent) and the amount actually paid to the user (amountToUser) represents the withdrawal fee, which must be forwarded in full to the treasury (`assetsSent - amountToUser`).

**What would go wrong if it stopped doing so**

The mutant replaces subtraction with division, so the treasury receives `assetsSent / amountToUser` instead of `assetsSent - amountToUser`. Since assetsSent is only slightly larger than amountToUser (a small percentage fee), integer division truncates to 1 in virtually all realistic fee configurations, meaning the treasury receives a token unit of ~1 instead of the correct (much larger) fee amount. Meanwhile `_updateBaseInterest` already reduced expected/available liquidity by the full `assetsSent`, so the pool's internal accounting assumes the fee was collected even though almost none of it actually left the contract's balance for the receiver+treasury transfers combined. This creates a permanent, growing discrepancy between actual token balance and `expectedLiquidity`/`totalAssets`, silently mispricing pool shares over time and causing the protocol to systematically under-collect withdrawal fee revenue.

**Attack scenario (hypothetical — requires the change above)**

1) Configurator calls `setWithdrawFee(newWithdrawFee)` with a nonzero fee (e.g., 100 bps). 2) Any liquidity provider calls `withdraw()` or `redeem()`. 3) `_withdraw` computes `assetsSent` (debited amount) and `amountToUser` (net amount, ~1% less). 4) Instead of transferring `assetsSent - amountToUser` (the fee) to treasury, the mutated code transfers `assetsSent / amountToUser`, which truncates to 1 wei for any withdrawal size (since ratio is ~1.01). 5) Repeated over many withdrawals, the treasury collects essentially nothing while depositors' effective fee-adjusted share price calculation still assumes the fee was extracted, leaving surplus underlying tokens stuck in the pool contract that are unaccounted for in `totalAssets()`/`expectedLiquidity()`. Over time this both starves protocol revenue and creates an unaccounted asset surplus that could be exploited or mismanaged by future contract upgrades or sweep functions.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -380,7 +380,8 @@
         IERC20(asset()).safeTransfer({to: receiver, value: amountToUser}); // U:[LP-8,9]
         if (assetsSent > amountToUser) {
             unchecked {
-                IERC20(asset()).safeTransfer({to: treasury, value: assetsSent - amountToUser}); // U:[LP-8,9]
+                /// BinaryOpMutation(`-` |==> `/`) of: `IERC20(asset()).safeTransfer({to: treasury, value: assetsSent - amountToUser}); // U:[LP-8,9]`
+                IERC20(asset()).safeTransfer({to: treasury, value: assetsSent/amountToUser}); // U:[LP-8,9]
             }
         }
         emit Withdraw(msg.sender, receiver, owner, assetsReceived, shares); // U:[LP-8,9]
```

**Recommended test**

Add a unit test (e.g., extending U:[LP-8]/[LP-9] withdraw/redeem test cases) that sets a nonzero `withdrawFee` via `setWithdrawFee`, performs a `withdraw`/`redeem`, and asserts that the treasury's underlying token balance increases by exactly `assetsSent - amountToUser` (not merely a nonzero amount), and that `assetsSent == amountToUser + treasuryDelta`. This would catch the divide-instead-of-subtract mutation.

---

## Test gaps with no security impact

Mutations the suite does not catch, assessed as not exploitable.

| Location | Function | Assessment |
|---|---|---|
| `contracts/pool/PoolV3.sol:383` | `_withdraw` | Given the protocol's enforced invariant that `withdrawFee <= MAX_WITHDRAW_FEE` (checked in `setWithdrawFee`) and that `MAX_WITHDRAW_FEE` is… |

## Methodology

Each mutant is a single deliberate change to the source — an inverted comparison, a deleted statement, swapped arguments. The mutant is compiled and the full test suite runs against it.

- A mutant that makes a test fail is **killed**: the suite covers that behaviour.
- A mutant that leaves every test passing **survived**: nothing in the suite checks that behaviour.
- A mutant that **does not compile** is excluded from the score. It says nothing about test quality, and counting it as killed would inflate the result.

Surviving mutants are then assessed individually against the full contract source to separate real test gaps from changes with no reachable impact.
