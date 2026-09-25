# Mutation Audit — CoW Protocol

**Repository**: [cowprotocol/contracts](https://github.com/cowprotocol/contracts)  
**Commit**: `c07a93e`  
**Scope**: `src/contracts/GPv2Settlement.sol` · 225 tests in the Foundry suite  
**Date**: 2026-09-25  
**Method**: Gambit (mutant generation) + Foundry (compile & test) + Claude (survivor assessment)

> **What this measures.** Every finding below is scoped to the **Foundry** suite
> as it stands at this commit. The repository is mid-migration from Hardhat, and
> three TypeScript test files remain (`balancer.ts`, `decoding.test.ts`,
> `encoding.ts`) — none of which touch the settlement flow. Whether equivalent
> coverage existed in the Hardhat suite before the migration was not verified.
> Read this as "the current Foundry suite does not cover this", not as "this was
> never tested".

> **The pattern.** Nine of the ten survivors are on the transfer side of
> settlement: the two transfer calls in `settle()`, and the `account`, `token`
> and `balance` fields of the transfer structs built in `computeTradeExecution`.
>
> Both halves are tested in isolation — `GPv2Transfer/*` exercises the transfer
> library with hand-built structs, and `ComputeTradeExecutions.*` exercises
> amount, fee and fill computation. `Settle.t.sol` covers the solver allowlist,
> the settlement event and interaction ordering, but never settles a trade that
> actually moves tokens. Nothing tests the seam between the two.

> Every finding below was reviewed by hand against the source. One severity was
> downgraded from the automated assessment; it says so in its entry.

---

## Summary

**Mutation score: 89.6%** — the test suite detected 86 of 96 behaviour-changing mutations.

| | Count |
|---|---:|
| Mutants generated | 96 |
| Killed by the suite | 86 |
| **Survived (undetected)** | **10** |

### Undefended invariants by severity

> Severity is the impact **if this behaviour broke and shipped unnoticed** — not a bug in the code as written. The code is correct today. The finding is that nothing in the test suite would catch it if it stopped being correct.

| Severity | Count |
|---|---:|
| CRITICAL | 3 |
| HIGH | 6 |
| MEDIUM | 1 |

**Least defended**: CRITICAL — Removal of vaultRelayer.transferFromAccounts drains protocol funds (`src/contracts/GPv2Settlement.sol:134`)

---

## Undefended invariants

Each entry is a change to your contracts that the suite does not detect. The contract is correct as written; what follows is the test that is missing.

### 1. [CRITICAL] Removal of vaultRelayer.transferFromAccounts drains protocol funds

**Location**: `src/contracts/GPv2Settlement.sol:134` in `settle()`  
**Type**: Logic Error / Missing Fund Transfer (Insolvency)  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 11)

**What the code is supposed to enforce**

The settle() function must pull the computed sell-token amounts from user accounts (via the vault relayer) before paying out the corresponding buy-token amounts to counterparties. This enforces that trades are fully collateralized and the protocol never pays out funds it hasn't collected.

**What would go wrong if it stopped doing so**

By replacing `vaultRelayer.transferFromAccounts(inTransfers)` with a no-op `assert(true)`, the settlement contract no longer collects any sell tokens from trading users, yet it still executes `vault.transferToAccounts(outTransfers)` to pay out buy tokens. This breaks the fundamental balance invariant of the settlement: for every trade, buy-side payouts occur without any matching sell-side inflow. The contract (and Balancer vault) will pay out funds sourced entirely from its own existing token balances (accumulated fees, leftover dust, or other users' unswept balances) rather than the current trade's sell tokens, quickly leading to insolvency and total fund loss across all subsequent trades in the batch and future settlements sharing the same balance pool.

**Attack scenario (hypothetical — requires the change above)**

1. Deploy contract with mutation active. 2. An authorized solver (or any solver session, since exploit does not require special solver collusion beyond the pre-existing onlySolver requirement) submits a settle() call with valid signed orders and trades. 3. computeTradeExecutions computes correct inTransfers/outTransfers, but inTransfers are silently never executed because the call is replaced by assert(true). 4. executeInteractions(interactions[1]) runs, then vault.transferToAccounts(outTransfers) pays out buyAmount tokens to order owners/receivers, funded by the settlement contract's or vault's existing balance instead of freshly pulled sell tokens. 5. The order's filledAmount is still updated as filled, but the seller's tokens were never taken. 6. Repeating this multiple times enables solvers (or complicit users) to receive buy tokens for free, fully draining any funds held by the vault/relayer meant for other users, resulting in total protocol insolvency.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -131,7 +131,8 @@
             GPv2Transfer.Data[] memory outTransfers
         ) = computeTradeExecutions(tokens, clearingPrices, trades);
 
-        vaultRelayer.transferFromAccounts(inTransfers);
+        /// DeleteExpressionMutation(`vaultRelayer.transferFromAccounts(inTransfers)` |==> `assert(true)`) of: `vaultRelayer.transferFromAccounts(inTransfers);`
+        assert(true);
 
         executeInteractions(interactions[1]);
```

**Recommended test**

Add an integration test that asserts ERC20 balances of order owners actually decrease by the exact `executedSellAmount` (or equivalent allowance/transferFrom call is invoked with correct arguments) after calling settle(). E.g., mock/spy on `vaultRelayer.transferFromAccounts` to assert it was called exactly once with the expected `inTransfers` array, and verify token balance deltas for both sell and buy sides in an end-to-end settle() test. This would fail immediately when the call is replaced with a no-op.

---

### 2. [CRITICAL] Removed outTransfers execution allows funds to be stuck/stolen in settle()

**Location**: `src/contracts/GPv2Settlement.sol:138` in `settle()`  
**Type**: Logic Error / Missing State Transfer  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 13)

**What the code is supposed to enforce**

The `settle` function must transfer sell tokens from traders' accounts via the vault relayer, execute intermediate interactions, and then send the corresponding bought tokens back to the trade recipients via `vault.transferToAccounts(outTransfers)`. This is the core mechanism ensuring both sides of a trade are settled atomically.

**What would go wrong if it stopped doing so**

By replacing the actual token transfer call with a no-op `assert(true)`, the mutant completely removes the step that pays out bought tokens to order owners/receivers. Meanwhile, `computeTradeExecutions` has already updated `filledAmount` for each order and pulled sell tokens from traders via `vaultRelayer.transferFromAccounts(inTransfers)`. The net effect: sell tokens are debited from users, `filledAmount` is marked as filled (preventing replay or correction), a `Trade` event is emitted claiming the swap succeeded, yet the buy-side tokens are never delivered to any recipient. Funds effectively vanish from user perspective (remain trapped in the settlement contract or vault), while the protocol logs a false record of successful settlement. This breaks the fundamental invariant that both legs of a trade execute atomically and correctly.

**Attack scenario (hypothetical — requires the change above)**

1. A malicious or compromised solver (already authorized via onlySolver) calls `settle()` with valid orders and interactions such that `computeTradeExecutions` correctly computes inTransfers/outTransfers. 2. `vaultRelayer.transferFromAccounts(inTransfers)` succeeds, pulling sell tokens from the users' accounts into the settlement/vault flow. 3. Because `vault.transferToAccounts(outTransfers)` is now a no-op, the buy-side tokens are never transferred to the recipients. 4. `filledAmount` for each order is already updated to reflect a full fill, so users cannot re-submit or reclaim their trade. 5. The solver (or any beneficiary controlling settlement interactions) can then withdraw the now-unaccounted-for buy tokens sitting in the vault/settlement contract via a subsequent interaction or a separate solver-controlled settle call, effectively stealing the difference. Net result: users lose their sold tokens, receive nothing, and an attacker (solver) can capture the un-transferred funds.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -135,7 +135,8 @@
 
         executeInteractions(interactions[1]);
 
-        vault.transferToAccounts(outTransfers);
+        /// DeleteExpressionMutation(`vault.transferToAccounts(outTransfers)` |==> `assert(true)`) of: `vault.transferToAccounts(outTransfers);`
+        assert(true);
 
         executeInteractions(interactions[2]);
```

**Recommended test**

Add an integration test for `settle()` that, after a successful call, asserts the ERC20 balance of each trade's `receiver`/`owner` increases by exactly `executedBuyAmount` (mirroring the `outTransfers` array), not just that `filledAmount` was updated or that events were emitted. A test such as `test_Settle_TransfersBuyTokensToReceivers` that checks pre/post balances of buy tokens for all recipients would kill this mutant, since removing `vault.transferToAccounts` would cause the balance assertion to fail.

---

### 3. [HIGH] Missing inTransfer.account assignment breaks sell-token attribution

**Location**: `src/contracts/GPv2Settlement.sol:433` in `computeTradeExecution()`  
**Type**: Logic Error / Broken Invariant (Missing State Assignment)  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 77)

**What the code is supposed to enforce**

The line `inTransfer.account = recoveredOrder.owner;` binds the computed sell-token transfer to the actual order owner so that `vaultRelayer.transferFromAccounts` pulls the sell amount from the correct user's balance/allowance during settlement. This is fundamental to the protocol's guarantee that a trader only pays with their own funds, based on their own signed order.

**What would go wrong if it stopped doing so**

With the assignment deleted, `inTransfer.account` retains its default zero-initialized value (address(0)) for every trade processed in `computeTradeExecutions`. Consequently, every in-transfer generated by `settle()` will attempt to move sell-token amounts from the zero address rather than from `recoveredOrder.owner`. Depending on the downstream transfer implementation in `GPv2Transfer`/`GPv2VaultRelayer` (typically using `safeTransferFrom(account, ...)` or the Vault's internal balance mechanics), this will either (a) unconditionally revert because address(0) has no allowance/balance, causing every single `settle()` call to fail — a complete denial-of-service of the core settlement functionality — or (b) in a worst-case implementation that does not strictly validate the `from` address, allow buy-token transfers to proceed to the receiver while no corresponding sell-token debit ever occurs from the real owner, resulting in free extraction of funds from the settlement contract's or vault's pooled balances. Either outcome is a critical failure: complete unavailability of the protocol's central function, or fund theft.

**Attack scenario (hypothetical — requires the change above)**

1. An authorized solver calls `settle()` with a normal batch containing at least one order with a nonzero sell amount.
2. Inside `computeTradeExecution`, `inTransfer.account` is never set to `recoveredOrder.owner` and remains address(0).
3. `vaultRelayer.transferFromAccounts(inTransfers)` is invoked with an in-transfer whose `account` field is the zero address.
4a. If the transfer implementation uses `safeTransferFrom(address(0), ...)`, the ERC20 call reverts (insufficient allowance/balance for zero address), causing the entire `settle()` transaction — and therefore all trades in the batch — to revert. Repeating this for any settlement makes the protocol permanently unusable.
4b. Alternatively, if the transfer mechanism does not strictly validate the source account (e.g., some non-standard token or Vault internal-balance path treats a zero/unset account permissively), the sell-side debit silently fails or transfers 0 value while the out-transfer to `recoveredOrder.receiver` still executes from the settlement contract's own or vault's pooled token reserves, letting a colluding solver/trader receive buy tokens without ever surrendering the corresponding sell tokens — directly draining protocol/vault funds.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -430,7 +430,8 @@
             orderUid
         );
 
-        inTransfer.account = recoveredOrder.owner;
+        /// DeleteExpressionMutation(`inTransfer.account = recoveredOrder.owner` |==> `assert(true)`) of: `inTransfer.account = recoveredOrder.owner;`
+        assert(true);
         inTransfer.token = order.sellToken;
         inTransfer.amount = executedSellAmount;
         inTransfer.balance = order.sellTokenBalance;
```

**Recommended test**

Add a unit test for `computeTradeExecution` (or an integration test on `settle()`) that asserts the resulting `GPv2Transfer.Data.account` field for the in-transfer equals `recoveredOrder.owner`, and an integration test that verifies actual ERC20 balance changes after `settle()` show the sell token debited from the order owner's account (not address(0)) and credited to the settlement/vault. A mock ERC20 that records the `from` argument of `transferFrom` calls would directly catch this mutant, since it would observe `from == address(0)` instead of the expected owner.

> **Reviewer note**: Downgraded from CRITICAL on hand review. With `inTransfer.account` left at `address(0)`, `transferFromAccounts` attempts an ERC20 `transferFrom` from the zero address and reverts. The effect is a denial of service, not silent misattribution.

---

### 4. [CRITICAL] Missing outTransfer.account assignment sends bought tokens to address(0)

**Location**: `src/contracts/GPv2Settlement.sol:438` in `computeTradeExecution()`  
**Type**: Logic Error / Incorrect State Assignment  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 81)

**What the code is supposed to enforce**

computeTradeExecution populates a GPv2Transfer.Data struct (outTransfer) that is later used by vault.transferToAccounts() to route the bought tokens to the order's designated receiver. The line `outTransfer.account = recoveredOrder.receiver;` sets the recipient address for this out-transfer, which is essential for correctly delivering funds to the trader (or a designated receiver).

**What would go wrong if it stopped doing so**

With the assignment removed, `outTransfer.account` retains its default zero-initialized value (address(0)) since `outTransfer` is a freshly allocated memory struct passed by reference from `computeTradeExecutions`. As a result every settled trade's buy-side transfer is instructed to be delivered to the zero address instead of the actual order receiver. Depending on the Vault/ERC20 implementation, this either (a) reverts the transfer to address(0) causing all settlements to fail, or (b) succeeds and permanently burns/loses the bought tokens that were meant for the trader. Either outcome breaks a core protocol invariant: user orders must be settled to the correct receiver, and constitutes a direct, unconditional loss of user funds or complete denial of service for every trade routed through `settle()`.

**Attack scenario (hypothetical — requires the change above)**

1. A solver submits a valid settlement via `settle()` containing one or more trades. 2. `computeTradeExecutions` builds `outTransfers` array; for each trade, `outTransfer.account` is never set to `recoveredOrder.receiver` and remains address(0). 3. `vault.transferToAccounts(outTransfers)` is called, attempting to send the bought token amount to address(0) for every trade. 4a. If the underlying token/vault reverts transfers to the zero address, all settlements permanently fail (protocol DoS). 4b. If the transfer to address(0) succeeds, the trader's purchased tokens are irrecoverably lost/burned while the trader's sell tokens have already been debited via the in-transfer, resulting in a full loss of the trade's value for every user whose order settles through this path. No special attacker privileges are needed—this triggers on every settlement.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -435,7 +435,8 @@
         inTransfer.amount = executedSellAmount;
         inTransfer.balance = order.sellTokenBalance;
 
-        outTransfer.account = recoveredOrder.receiver;
+        /// DeleteExpressionMutation(`outTransfer.account = recoveredOrder.receiver` |==> `assert(true)`) of: `outTransfer.account = recoveredOrder.receiver;`
+        assert(true);
         outTransfer.token = order.buyToken;
         outTransfer.amount = executedBuyAmount;
         outTransfer.balance = order.buyTokenBalance;
```

**Recommended test**

Add a unit test for `computeTradeExecution`/`computeTradeExecutions` that asserts the returned `outTransfer.account` equals `recoveredOrder.receiver` (and differs from the sell-side account when receiver != owner). Additionally, add an integration test on `settle()` using a mock `IVault` that records and asserts the `account` field of each `GPv2Transfer.Data` passed to `transferToAccounts`, failing if any entry has `account == address(0)` while a valid receiver was set in the order.

---

### 5. [HIGH] Missing buyToken limit removes price protection in swap()

**Location**: `src/contracts/GPv2Settlement.sol:182` in `swap()`  
**Type**: Missing Input Validation / Slippage Protection Bypass  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 29)

**What the code is supposed to enforce**

In the swap() function, for KIND_SELL orders the code sets limits[trade.buyTokenIndex] = -limitAmount.toInt256() so that the Balancer Vault enforces a minimum output amount (order.buyAmount or better) when executing the batch swap. This encodes the order's limit price directly into the vault's `limits` array, which the vault uses to revert the swap if the actual received amount is below the negotiated minimum.

**What would go wrong if it stopped doing so**

By deleting the assignment (replaced with assert(true)), limits[trade.buyTokenIndex] remains at its default value of 0 (from `new int256[](tokens.length)`). Balancer's Vault enforces `delta <= limits[i]` for each asset; for an output token, delta is negative (tokens received), so a limit of 0 means the constraint `delta <= 0` is trivially satisfied for ANY positive receive amount, no matter how small. This completely nullifies the price/slippage protection for KIND_SELL orders executed via swap(). The only post-swap check that remains is `require(executedSellAmount == order.sellAmount)`, which validates the sell side was fully consumed but says nothing about how much the user actually received on the buy side. As a result, an order can be executed at an arbitrarily bad exchange rate, and the user (order owner) can receive far less than order.buyAmount while still having their full sellAmount taken.

**Attack scenario (hypothetical — requires the change above)**

1. A user signs a KIND_SELL order specifying sellAmount=1000 USDC, buyAmount=500 DAI (limit price 2:1).
2. A solver (authorized via onlySolver, but potentially malicious, compromised, or colluding with a searcher) calls swap() with crafted swap steps that route through a manipulated or thin pool where the effective price is far worse (e.g., only 100 DAI out for 1000 USDC).
3. Because limits[trade.buyTokenIndex] is now 0 instead of -500e18, the Vault's batchSwap does not revert even though only 100 DAI was received instead of the required 500 DAI minimum.
4. Post-swap checks only verify executedSellAmount == order.sellAmount (1000 USDC), which passes; there is no check that executedBuyAmount respects order.buyAmount.
5. filledAmount[orderUid] is set to order.sellAmount, marking the order as fully executed, and the Trade/Settlement events fire normally, hiding the value extraction from naive observers.
6. The user's 1000 USDC is spent but they only receive 100 DAI worth of value (or whatever the solver/attacker chooses), with the difference captured by the solver or a colluding pool/arbitrageur (e.g., via a sandwich attack), resulting in direct fund loss for the order owner.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -179,7 +179,8 @@
         if (order.kind == GPv2Order.KIND_SELL) {
             require(limitAmount >= order.buyAmount, "GPv2: limit too low");
             limits[trade.sellTokenIndex] = order.sellAmount.toInt256();
-            limits[trade.buyTokenIndex] = -limitAmount.toInt256();
+            /// DeleteExpressionMutation(`limits[trade.buyTokenIndex] = -limitAmount.toInt256()` |==> `assert(true)`) of: `limits[trade.buyTokenIndex] = -limitAmount.toInt256();`
+            assert(true);
         } else {
             require(limitAmount <= order.sellAmount, "GPv2: limit too high");
             limits[trade.sellTokenIndex] = limitAmount.toInt256();
```

**Recommended test**

Add a unit test for `swap()` with a KIND_SELL order where the underlying Balancer swap route returns an executedBuyAmount below order.buyAmount (e.g., mock IVault.batchSwap to return a smaller negative delta for the buy token) and assert that the transaction reverts. This would kill the mutant because with limits[trade.buyTokenIndex] correctly set to -limitAmount, the mocked Vault call would revert on the limit check, whereas with the mutation applied it would succeed. Additionally test that limits[trade.buyTokenIndex] is set to a nonzero value for KIND_SELL orders by inspecting the arguments passed to vaultRelayer.batchSwapWithFee via a mock/spy.

---

### 6. [HIGH] Missing sellToken assignment causes zero-address token transfers

**Location**: `src/contracts/GPv2Settlement.sol:434` in `computeTradeExecution()`  
**Type**: Logic Error / Data Corruption  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 78)

**What the code is supposed to enforce**

The code populates the `inTransfer` struct with the sell token address so that `vaultRelayer.transferFromAccounts` correctly identifies which ERC20 token to pull from the order owner's account.

**What would go wrong if it stopped doing so**

By deleting `inTransfer.token = order.sellToken;`, the `token` field of the freshly-allocated `GPv2Transfer.Data` struct remains at its default value of `address(0)` (Solidity zero-initializes memory structs). This means every settlement trade's inbound transfer will reference the null address instead of the actual sell token. When `vaultRelayer.transferFromAccounts` attempts to call `IERC20(address(0)).transferFrom(...)`, the call will revert because there is no contract code at address(0) (a low-level call to a non-contract address returns success with empty data unless `extcodesize` checks are used, in which case behavior is undefined but almost certainly not the intended token transfer). This breaks the core `settle()` functionality for every trade, causing a complete denial of service on the settlement mechanism, and if any downstream logic does not validate token addresses it could also cause mismatched accounting between inTransfers and outTransfers.

**Attack scenario (hypothetical — requires the change above)**

1. A solver calls `settle()` with valid orders, tokens, and prices.
2. Internally, `computeTradeExecution` builds `inTransfer` for each trade but never sets `inTransfer.token`, leaving it as `address(0)`.
3. `vaultRelayer.transferFromAccounts(inTransfers)` is invoked with transfer records pointing to token `address(0)`.
4. The call to transfer sell tokens from the order owner either reverts (bricking every settlement) or, in an environment where zero-address calls silently succeed (e.g., low-level call without a revert check), no actual token transfer occurs even though the trade is recorded as filled and buy-side funds are paid out from `outTransfers`. This would let a malicious solver settle orders where the sell-side transfer never actually debits the user, resulting in the contract paying out buyAmount tokens without receiving the corresponding sellAmount — a direct fund-drain vector if the call doesn't revert.
5. Either outcome (total DoS or fund-loss via unbacked buy transfers) is a critical failure of core settlement invariants.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -431,7 +431,8 @@
         );
 
         inTransfer.account = recoveredOrder.owner;
-        inTransfer.token = order.sellToken;
+        /// DeleteExpressionMutation(`inTransfer.token = order.sellToken` |==> `assert(true)`) of: `inTransfer.token = order.sellToken;`
+        assert(true);
         inTransfer.amount = executedSellAmount;
         inTransfer.balance = order.sellTokenBalance;
```

**Recommended test**

Add a unit/integration test for `computeTradeExecution`/`settle()` that asserts the `token` field of each generated `GPv2Transfer.Data` (both inTransfer and outTransfer) equals `order.sellToken`/`order.buyToken` respectively, e.g. by inspecting the arguments passed to a mocked `vaultRelayer.transferFromAccounts` and `vault.transferToAccounts`. A test that decodes the calldata used in `transferFromAccounts` mock and checks `token != address(0)` and equals the expected sell token would kill this mutant.

---

### 7. [HIGH] Missing propagation of sellTokenBalance breaks fund-source invariant in trade transfer

**Location**: `src/contracts/GPv2Settlement.sol:436` in `computeTradeExecution()`  
**Type**: Logic Error / Broken Invariant (Incorrect balance-location routing)  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 80)

**What the code is supposed to enforce**

The `inTransfer.balance = order.sellTokenBalance;` assignment propagates the signed order's chosen balance location (ERC20 wallet balance, Balancer external balance, or Balancer internal balance) into the GPv2Transfer.Data struct so that the vault relayer withdraws the seller's funds from the exact balance source the trader authorized when signing the order. This field is part of the EIP-712 signed order and is a core trust boundary: a trader may deliberately keep e.g. an ERC20 allowance untouched and only authorize use of their Balancer internal balance for a given order, or vice versa.

**What would go wrong if it stopped doing so**

By deleting the assignment and replacing it with `assert(true)`, `inTransfer.balance` is left at its default zero value (bytes32(0)), which does not correspond to any of GPv2Order's BALANCE_ERC20 / BALANCE_EXTERNAL / BALANCE_INTERNAL hash constants. Downstream, GPv2Transfer's dispatch logic branches on this value to decide which balance source to debit from. Regardless of what the trader actually signed, every sell-side transfer now silently falls into whatever branch corresponds to the zero/default case (commonly the ERC20 or internal-balance fallback), instead of the trader-specified source. This breaks a signed, security-relevant invariant of the order: the solver-controlled settlement could pull funds from a balance location the trader never intended to expose for that specific order (e.g., debiting the trader's main ERC20 allowance when they explicitly opted for Balancer-internal-balance-only trading, or vice versa), or cause spurious reverts/DoS for legitimate orders that rely on a balance type not matching the default branch. Either outcome undermines a fund-safety guarantee of the protocol and was not caught because the existing test suite apparently only exercises the default/ERC20 balance path.

**Attack scenario (hypothetical — requires the change above)**

1. A trader signs an order with `sellTokenBalance = GPv2Order.BALANCE_INTERNAL`, intending only their Balancer internal balance to be used for this trade, while keeping their on-chain ERC20 approval to the vault relayer active for other unrelated (external-balance) orders.
2. A solver includes this order in a `settle()` call. `computeTradeExecution` runs with the mutant: `inTransfer.balance` stays at `bytes32(0)` instead of `BALANCE_INTERNAL`.
3. `vaultRelayer.transferFromAccounts(inTransfers)` interprets the zero balance value as a different balance type (e.g., ERC20), and pulls the sell amount directly from the trader's regular ERC20 balance/allowance instead of their internal Balancer balance.
4. The trader's wallet funds—which they never intended to expose to this particular order—are debited, or the transaction reverts unexpectedly, causing a denial of service for legitimately internal-balance-funded orders. In the fund-misdirection case, the trader suffers an unauthorized debit outside the scope they signed for.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -433,7 +433,8 @@
         inTransfer.account = recoveredOrder.owner;
         inTransfer.token = order.sellToken;
         inTransfer.amount = executedSellAmount;
-        inTransfer.balance = order.sellTokenBalance;
+        /// DeleteExpressionMutation(`inTransfer.balance = order.sellTokenBalance` |==> `assert(true)`) of: `inTransfer.balance = order.sellTokenBalance;`
+        assert(true);
 
         outTransfer.account = recoveredOrder.receiver;
         outTransfer.token = order.buyToken;
```

**Recommended test**

Add a unit test on `computeTradeExecution`/`settle` that constructs orders with each of the three `sellTokenBalance` values (BALANCE_ERC20, BALANCE_EXTERNAL, BALANCE_INTERNAL) and asserts that the resulting `GPv2Transfer.Data.balance` field passed to `vaultRelayer.transferFromAccounts` exactly equals the order's `sellTokenBalance` (e.g., via a mock vault relayer capturing call arguments). This would directly kill this mutant since the zeroed/default value would fail the equality assertion for the EXTERNAL and INTERNAL cases.

---

### 8. [HIGH] outTransfer.token field left uninitialized in computeTradeExecution

**Location**: `src/contracts/GPv2Settlement.sol:439` in `computeTradeExecution()`  
**Type**: Logic Error / State Corruption (Fund Transfer Mismatch)  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 82)

**What the code is supposed to enforce**

The code should set outTransfer.token = order.buyToken so that when vault.transferToAccounts(outTransfers) is later invoked, the buyer receives the correct buyToken that the trader intended to purchase. This field is essential for correctly routing settlement out-transfers.

**What would go wrong if it stopped doing so**

By deleting the assignment (replaced with assert(true), a no-op), outTransfer.token retains its default zero-initialized value (address(0)) from the `new GPv2Transfer.Data[](trades.length)` allocation, instead of order.buyToken. Since outTransfer.amount, .account, and .balance are still correctly populated, the resulting transfer struct describes a nonzero-amount transfer of the zero address 'token' to the trader's receiver. Depending on how GPv2Transfer.transferToAccounts/IVault handles a zero-address ERC20 (calling IERC20(address(0)).transfer(...) will typically revert in Solidity >=0.6 due to the automatic extcodesize check on functions with return values), this either (a) causes every settlement containing this trade to revert (a denial-of-service on the settlement contract), or (b) if the transfer path silently no-ops for a zero address / uses low-level calls without decoding return data, the trader's sell tokens are debited (filledAmount updated, Trade event emitted with correct buyToken info) while the trader never actually receives their purchased buyToken — a direct, silent loss of funds for the trader with no compensating revert.

**Attack scenario (hypothetical — requires the change above)**

1. A solver submits a settle() batch containing a trade where the mutated computeTradeExecution runs.
2. inTransfer correctly debits the trader's sellToken via vaultRelayer.transferFromAccounts.
3. The corresponding outTransfer for this trade has token=address(0) instead of order.buyToken, due to the deleted assignment.
4. filledAmount[orderUid] is updated as if the trade succeeded and a Trade event is emitted showing the correct (but never transferred) buyToken and buyAmount.
5. vault.transferToAccounts(outTransfers) is called with the corrupted struct: if the underlying transfer logic does not revert on a token=address(0) transfer (e.g., uses a low-level .call that succeeds trivially against an address with no code), the trader's account never receives the buyToken while their sellToken has already been debited — funds are effectively lost to the trader (retained in the settlement contract or vault instead of forwarded).
6. If the vault instead reverts on such a transfer, then every settlement batch containing this trade path permanently fails, which is a critical denial-of-service on core settlement functionality that could be missed if end-to-end integration tests were not exercising the full settle() -> vault.transferToAccounts() path for this branch.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -436,7 +436,8 @@
         inTransfer.balance = order.sellTokenBalance;
 
         outTransfer.account = recoveredOrder.receiver;
-        outTransfer.token = order.buyToken;
+        /// DeleteExpressionMutation(`outTransfer.token = order.buyToken` |==> `assert(true)`) of: `outTransfer.token = order.buyToken;`
+        assert(true);
         outTransfer.amount = executedBuyAmount;
         outTransfer.balance = order.buyTokenBalance;
     }
```

**Recommended test**

Add a unit test that calls computeTradeExecution (or computeTradeExecutions) directly and asserts outTransfer.token == order.buyToken for both KIND_SELL and KIND_BUY branches. Additionally add/expand an integration test that calls settle() end-to-end with a real or mock Vault and asserts the buyToken balance of the receiver actually increases by executedBuyAmount, which would fail (either by revert or balance mismatch) under this mutation.

---

### 9. [HIGH] outTransfer.balance always defaults to 0, ignoring order.buyTokenBalance

**Location**: `src/contracts/GPv2Settlement.sol:441` in `computeTradeExecution()`  
**Type**: Logic Error / Broken Invariant  
**Confidence**: MEDIUM  
**Mutation**: `DeleteExpressionMutation` (id 84)

**What the code is supposed to enforce**

The `outTransfer.balance` field must be copied from `order.buyTokenBalance` so that downstream transfer logic (GPv2Transfer/Vault interaction) knows whether the buy-side settlement payout should be delivered as a plain ERC20 transfer, or routed through the Balancer Vault's internal/external balance mechanism, exactly as the trader signed in their order.

**What would go wrong if it stopped doing so**

With the assignment deleted, `outTransfer.balance` keeps its default zero value from the `new GPv2Transfer.Data[]` allocation instead of the actual `order.buyTokenBalance` (which is one of the non-zero keccak-derived constants BALANCE_ERC20/BALANCE_INTERNAL/BALANCE_EXTERNAL defined in GPv2Order). Because zero never equals any of these constants, every single buy-side transfer in every settlement will be routed to the 'non-ERC20' branch of the transfer-to-accounts logic (typically a Balancer Vault manageUserBalance/internal-balance deposit) regardless of what the trader actually signed. This silently overrides the balance-location field that is part of the order hash the user cryptographically committed to, breaking the core invariant that settlement execution must respect signed order parameters. Users expecting a direct ERC20 transfer to their wallet will instead have funds deposited into their Balancer Vault internal balance, which they must separately withdraw, and which behaves differently with respect to custody, composability and risk assumptions.

**Attack scenario (hypothetical — requires the change above)**

1. A trader signs a GPv2 order with `buyTokenBalance = BALANCE_ERC20`, expecting the bought tokens to be transferred directly into their EOA/contract wallet. 2. A solver calls `settle()` including this order; `computeTradeExecution` builds the `outTransfer` struct but (due to the mutation) never sets `outTransfer.balance`, leaving it as the zero value. 3. `vault.transferToAccounts(outTransfers)` is invoked; since `outTransfer.balance != BALANCE_ERC20`, the transfer is routed through the Vault's internal-balance mechanism instead of a direct ERC20 transfer. 4. The trader's tokens land in their Balancer Vault internal balance rather than their wallet. If the receiver is a smart contract that does not know to call `vault.withdrawFromInternalBalance`, or if internal balance semantics interact with unexpected permissions/fees, the trader effectively loses immediate access to the purchased tokens and must take extra, non-obvious action to recover them — a systemic, always-triggered misdirection of settlement funds affecting every trade in the batch.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -438,7 +438,8 @@
         outTransfer.account = recoveredOrder.receiver;
         outTransfer.token = order.buyToken;
         outTransfer.amount = executedBuyAmount;
-        outTransfer.balance = order.buyTokenBalance;
+        /// DeleteExpressionMutation(`outTransfer.balance = order.buyTokenBalance` |==> `assert(true)`) of: `outTransfer.balance = order.buyTokenBalance;`
+        assert(true);
     }
 
     /// @dev Execute a list of arbitrary contract calls from this contract.
```

**Recommended test**

Add a unit test for `computeTradeExecution` that constructs orders with each of the three `buyTokenBalance` values (BALANCE_ERC20, BALANCE_INTERNAL, BALANCE_EXTERNAL) and asserts that the returned `outTransfer.balance` (and, ideally in an integration test, the actual downstream call made to the Vault) exactly matches `order.buyTokenBalance`. This directly kills the mutant since the deleted assignment would then produce a mismatched/default value only when `order.buyTokenBalance` is non-zero.

---

### 10. [MEDIUM] Missing sell-token limit assignment breaks KIND_SELL swap() path

**Location**: `src/contracts/GPv2Settlement.sol:181` in `swap()`  
**Type**: Logic Error / Denial of Service  
**Confidence**: HIGH  
**Mutation**: `DeleteExpressionMutation` (id 28)

**What the code is supposed to enforce**

In the swap() function, the `limits` array passed to Balancer's batchSwap must set `limits[trade.sellTokenIndex] = order.sellAmount.toInt256()` for KIND_SELL orders so that the Vault enforces the trader's maximum authorized outflow of the sell token (a positive limit bounding the amount the settlement contract may pay out of the trader's balance).

**What would go wrong if it stopped doing so**

With the assignment deleted, `limits[trade.sellTokenIndex]` remains at its default value of 0 instead of `order.sellAmount`. Balancer's Vault enforces, for each token with a positive net delta (amount owed to the Vault), that `delta <= limit`. Since any real KIND_SELL swap requires paying a nonzero amount of the sell token (delta > 0), the check `delta <= 0` will fail and the batchSwap call will revert. This effectively disables the swap() entry point for all KIND_SELL orders when routed against the real Balancer Vault, even though unit tests using a mocked vaultRelayer/vault do not simulate this limit-check logic and therefore pass. This is a functional correctness/availability defect rather than a fund-draining vulnerability, since the mutated value is more restrictive (0) rather than looser than the original.

**Attack scenario (hypothetical — requires the change above)**

1. A solver constructs a valid KIND_SELL order and calls `swap()` against a live Balancer Vault. 2. The vaultRelayer.batchSwapWithFee call is forwarded to the real Vault with `limits[sellTokenIndex] = 0` instead of `order.sellAmount`. 3. Because the Vault computes a positive delta for the sell token exceeding the 0 limit, the Vault reverts the entire transaction. 4. This causes solver transactions for KIND_SELL swap orders to permanently fail on-chain, effectively taking the swap() feature offline for this order type, wasting solver gas and reducing protocol availability. No direct fund loss occurs, but the settlement mechanism for this trade type becomes unusable.

**The mutation the suite missed**

```diff
--- original
+++ mutant
@@ -178,7 +178,8 @@
         // the order's limit price.
         if (order.kind == GPv2Order.KIND_SELL) {
             require(limitAmount >= order.buyAmount, "GPv2: limit too low");
-            limits[trade.sellTokenIndex] = order.sellAmount.toInt256();
+            /// DeleteExpressionMutation(`limits[trade.sellTokenIndex] = order.sellAmount.toInt256()` |==> `assert(true)`) of: `limits[trade.sellTokenIndex] = order.sellAmount.toInt256();`
+            assert(true);
             limits[trade.buyTokenIndex] = -limitAmount.toInt256();
         } else {
             require(limitAmount <= order.sellAmount, "GPv2: limit too high");
```

**Recommended test**

Add a test that verifies the exact `limits` array constructed and passed to `vaultRelayer.batchSwapWithFee` for a KIND_SELL order (e.g., via a mock vaultRelayer that records call arguments) and asserts `limits[trade.sellTokenIndex] == order.sellAmount.toInt256()`. Additionally, add an integration-style test using a realistic Balancer Vault mock that reverts if the sell-token limit is insufficient, to catch cases where the limit array is incorrectly populated.

---

## Methodology

Each mutant is a single deliberate change to the source — an inverted comparison, a deleted statement, swapped arguments. The mutant is compiled and the full test suite runs against it.

- A mutant that makes a test fail is **killed**: the suite covers that behaviour.
- A mutant that leaves every test passing **survived**: nothing in the suite checks that behaviour.
- A mutant that **does not compile** is excluded from the score. It says nothing about test quality, and counting it as killed would inflate the result.

Surviving mutants are then assessed individually against the full contract source to separate real test gaps from changes with no reachable impact.
