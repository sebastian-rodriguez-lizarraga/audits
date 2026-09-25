# Mutation Audit — Morpho — Vaults V2

**Repository**: [morpho-org/vault-v2](https://github.com/morpho-org/vault-v2)  
**Commit**: `1ae84f3`  
**Scope**: `src/VaultV2.sol` · 373 tests in the suite  
**Date**: 2026-09-25  
**Method**: Gambit (mutant generation) + Foundry (compile & test) + Claude (survivor assessment)

---

## Summary

**Mutation score: 100.0%** — the test suite detected 495 of 495 behaviour-changing mutations.

| | Count |
|---|---:|
| Mutants generated | 495 |
| Killed by the suite | 495 |
| **Survived (undetected)** | **0** |

No mutation with security impact survived the suite.

---

## Methodology

Each mutant is a single deliberate change to the source — an inverted comparison, a deleted statement, swapped arguments. The mutant is compiled and the full test suite runs against it.

- A mutant that makes a test fail is **killed**: the suite covers that behaviour.
- A mutant that leaves every test passing **survived**: nothing in the suite checks that behaviour.
- A mutant that **does not compile** is excluded from the score. It says nothing about test quality, and counting it as killed would inflate the result.

Surviving mutants are then assessed individually against the full contract source to separate real test gaps from changes with no reachable impact.
