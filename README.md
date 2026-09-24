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

| Date | Protocol | Scope | Result |
|---|---|---|---|
| — | — | — | *first report in progress* |

## Contact

Telegram: [@Sebas200000](https://t.me/Sebas200000)
