# Your test suite has 100% coverage of a line that checks nothing

I ran mutation testing against four DeFi protocols — Morpho, Gearbox, Arcadia
and CoW — 1,360 mutations in total. This is what came out.

## The finding that explains why I bother

Gearbox's `PoolV3.sol` carries an annotation on line 371 naming the two tests
that cover it:

```solidity
if (msg.sender != owner) _spendAllowance({owner: owner, spender: msg.sender, amount: shares}); // U:[LP-8,9]
```

Both of those tests do assert on the allowance:

```solidity
vm.prank(user);
pool.withdraw({assets: cases[i].assets, receiver: owner, owner: owner});

assertEq(pool.allowance(user, owner), 0, "Incorrect shares allowance");
```

ERC20's signature is `allowance(owner, spender)`. The withdrawal runs with
`msg.sender == user` and the `owner` parameter set to `owner`, so the contract
decrements `allowance[owner][user]`. The assertion reads `allowance[user][owner]`
— the opposite entry of the mapping. It is never written. It is zero whether or
not the allowance check runs.

So: the line is annotated as covered. A test exercises it. An assertion fires.
And you can delete the allowance check entirely and all 343 tests still pass.
Without it, anyone can withdraw another account's shares.

Line coverage reports this line at 100%. It is checked by nothing.

That gap — between "this line ran" and "something verified what it did" — is
the entire reason mutation testing exists.

## What mutation testing actually does

Take your contract. Make one small deliberate change: invert a comparison,
delete a state update, swap two arguments. Compile it. Run your whole suite.

If a test fails, the mutant is **killed** — your suite defends that behaviour.
If every test still passes, the mutant **survived** — nothing in your suite
checks it.

Coverage tells you which lines executed. Mutation testing tells you which lines
are actually defended. They are not the same number, and the gap between them
is where regressions live.

## Four protocols, four different answers

| Protocol | Scope | Mutations | Score | Undefended |
|---|---|---:|---:|---:|
| Morpho Vaults V2 | `VaultV2.sol` | 495 | **100%** | 0 |
| Gearbox core-v3 | `PoolV3.sol` | 220 | 97.7% | 4 |
| Arcadia Lending | `LendingPool.sol` | 549 | 92.9% | 24 |
| CoW Protocol | `GPv2Settlement.sol` | 96 | 89.6% | 9 |

Morpho killed all 495. That is worth stating plainly, because a tool that
always finds something is not measuring anything. It found nothing there.

## The pattern

Across the three suites that did leave gaps, the survivors were not scattered.

**Arcadia** — 549 mutations, 39 survivors. Every one of the 24 with real impact
sits in the liquidation and bad-debt path: `_processDefault`,
`settleLiquidationUnhappyFlow`, `auctionRepay`, `startLiquidation`. Its `borrow`
and flash-action paths are thoroughly defended — the nine survivors there are
event-payload gaps with no reachable impact.

Three separate mutations turn `_processDefault` into a no-op. One of them
(`i > 0` becomes `0 > i`, never true for a `uint256`) skips the bad-debt
write-off loop entirely. Another routes execution into an `unchecked` block
where the tranche balance wraps around instead of being reduced.

The contract's own comment on that path reads *"Unhappy flow, should never
occur in practice!"*.

**CoW** — nine of ten survivors are on the transfer side of settlement: both
transfer calls in `settle()`, and the `account`, `token` and `balance` fields
of the transfer structs. Both halves are well tested in isolation.
`GPv2Transfer/*` exercises the transfer library with hand-built structs;
`ComputeTradeExecutions.*` covers the amount, fee and fill maths. `Settle.t.sol`
covers the solver allowlist, the settlement event and interaction ordering —
and never settles a trade that moves tokens. Nothing tests the seam.

**Gearbox** — 97.7%, one of the strongest suites I measured, and the residue
is the swapped-argument assertion above.

The shape is consistent:

> Teams test the happy path, and they test components in isolation. What goes
> undefended is the error paths, and the seams between components.

Which makes sense. You write tests for what users do. Liquidation and bad-debt
handling are what happens when things go wrong, and by construction nobody is
exercising them in development.

## Three ways your mutation score can lie

Building this, I hit three failure modes that all produce a confident, wrong
number. Each one is silent in the dangerous direction.

**1. Mutants that don't compile counted as killed.** `forge test` exits non-zero
both when a test fails and when the code doesn't compile. Conflate them and
every uncompilable mutant inflates your score while hiding a real gap. Compile
first, and exclude compile failures from the denominator — they say nothing
about test quality.

**2. A red baseline.** If the suite already fails before any mutation, every
mutant exits non-zero and you get a ~100% kill rate that means nothing. Three
of the five repositories I tried did not pass clean on a current toolchain:
Euler's EVC had three failing tests, Gearbox had one, and CoW's
`deny_warnings = true` combined with newer solc warnings stopped `forge test`
from running at all. Verify the baseline is green before mutating anything.

**3. The wrong solc.** Gambit returned zero mutants and exit code 0 when the
pinned pragma didn't match the installed compiler. No error, no warning. Just
an empty run that looks like a clean one.

All three produce output that looks fine. That is what makes them worth naming.

## Method

Gambit generates the mutants. Foundry compiles and runs the suite against each
one. Surviving mutants are assessed against the full contract source to
separate real gaps from changes with no reachable impact.

Every finding in the reports was then reviewed by hand against the source
before publication. Of 33 actionable findings across Arcadia, CoW and Gearbox,
33 held up mechanically and 5 had their severity downgraded from the automated
assessment. The reports say which ones and why.

Cost, for calibration: between 6 and 47 seconds per mutant depending mostly on
`via_ir` and how long the suite takes, not on contract size. The Arcadia run
was 7 hours; CoW was 10 minutes.

## To be clear

None of these are bugs. All four protocols are correct as written. The finding
is that nothing in their suites would fail if they stopped being correct — and
that this clusters in the code that decides solvency.

Full reports, with every surviving mutant and the reasoning behind each
severity: https://github.com/sebastian-rodriguez-lizarraga/audits

If you want this run against your contracts, I'm at
[@Sebas200000](https://t.me/Sebas200000).
