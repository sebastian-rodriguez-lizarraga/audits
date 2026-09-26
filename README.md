# Security & Test Quality Audits

Mutation testing and test-suite hardening for Solidity protocols.

## What I do

I find the changes to your contracts that your test suite would not notice —
and write the tests that catch them.

Coverage tells you which lines ran. It does not tell you whether anything was
asserted. A suite can have 95% line coverage and still not fail when you invert
a comparison or delete a state update.

Mutation testing answers the question coverage cannot: **if this code stopped
being correct, would any test fail?**

## How it works

1. Generate thousands of small, deliberate changes to your contracts
   (inverted comparisons, deleted statements, swapped arguments).
2. Compile and run your full suite against each one.
3. A change that makes a test fail is *caught*. A change that leaves every test
   passing is an **undefended invariant** — nothing in the suite checks it.
4. Assess each survivor against the full contract to separate real gaps from
   changes with no reachable impact.
5. Write the missing tests and prove they catch it.

Mutants that fail to compile are excluded from the score. They say nothing
about test quality, and counting them inflates the result.

## Reports

| Date | Protocol | Scope | Mutations | Score | Undefended |
|---|---|---|---:|---:|---:|
| 2026-09 | [Arcadia Finance](reports/2026-09-arcadia-lending-pool.md) | `LendingPool.sol` | 549 | 92.9% | **24** |
| 2026-09 | [Morpho Vaults V2](reports/2026-09-morpho-vault-v2.md) | `VaultV2.sol` | 495 | **100%** | 0 |
| 2026-09 | [CoW Protocol](reports/2026-09-cow-protocol.md) | `GPv2Settlement.sol` | 96 | 89.6% | **9** |
| 2026-09 | [Gearbox core-v3](reports/2026-09-gearbox-core-v3.md) | `PoolV3.sol` | 220 | 97.7% | 4 |

Four protocols, 1,360 mutations, four different answers.

**Morpho** caught all 495 of its mutations.

**Gearbox** caught 97.7% — and the residue includes a line its own source
annotates as covered by two named tests. Both tests assert on the allowance,
with the arguments to `allowance(owner, spender)` the wrong way round, so they
read a mapping entry that is zero either way. Delete the allowance check
entirely and all 343 tests still pass.

**Arcadia** defends its happy path thoroughly; every undefended invariant with
real impact is in its liquidation and bad-debt code.

**CoW** shows a third shape: nine of ten survivors are on the transfer side of
settlement. The transfer library and the execution maths are each well tested
in isolation — nothing tests the seam where `settle()` joins them.

The pattern across all four: teams test the happy path and they test components
in isolation. What goes undefended is the error paths, and the seams between
components.

A 100% score is as much a result as a 92.9% one. A tool that always finds
something is not measuring anything.

Every finding is reviewed by hand against the source before publication.
Of the 24 on Arcadia, 24 held up mechanically and 4 had their severity
downgraded; the reports say which.

## Contact

Telegram: [@Sebas200000](https://t.me/Sebas200000)
