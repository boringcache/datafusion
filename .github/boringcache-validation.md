# DataFusion BoringCache validation

## Result

This run does not validate the proposed cross-job Cargo reuse improvement. The
compiler cache was warm, but the Cargo target was not published. Each restore
job then ran the full workspace check inside the BoringCache Cargo action before
running its assigned feature checks.

All three runs used upstream source
`7917a9a65a0343a6b5536299de5482a899d77c51`, Cargo.lock SHA-256
`c40522a03c46b513b77909c0e90e1c4198ddccafd1df1c7f856a7800c374471d`,
Rust 1.98.1, and Cargo 1.98.1. The seven feature jobs retained their upstream
runner expressions and ran in parallel.

## Runs

- [Cold baseline](https://github.com/boringcache/datafusion/actions/runs/34979495334): success, workflow commit `f6a09a63793335cd429aaa1527f1ecc88de74cc4`.
- [Trusted seed](https://github.com/boringcache/datafusion/actions/runs/34980483355): success, workflow commit `f6a09a63793335cd429aaa1527f1ecc88de74cc4`.
- [Restore fan-out](https://github.com/boringcache/datafusion/actions/runs/34982773905): success, workflow commit `90ce8e057d767c505d7f9db52b975e43892c9259`.

Times below come from GitHub's job and step timestamps. “Check steps” is the
sum of the unchanged feature-check commands in each job. Parallel job totals
measure runner use, not workflow wall time.

| Feature job | Baseline job | Restore job | Cargo action in restore | Baseline check steps | Restore check steps |
| --- | ---: | ---: | ---: | ---: | ---: |
| datafusion-common | 128 s | 394 s | 273 s | 54 s | 36 s |
| datafusion-substrait | 477 s | 861 s | 326 s | 333 s | 448 s |
| datafusion-proto | 240 s | 499 s | 283 s | 160 s | 126 s |
| datafusion-ffi | 156 s | 355 s | 228 s | 74 s | 35 s |
| datafusion | 435 s | 753 s | 324 s | 362 s | 361 s |
| datafusion-functions | 179 s | 466 s | 303 s | 103 s | 71 s |
| datafusion-spark | 197 s | 522 s | 309 s | 127 s | 97 s |
| **Total** | **1,812 s** | **3,850 s** | **2,046 s** | **1,213 s** | **1,174 s** |

The feature-check steps used 39 fewer seconds in the restore run, a 3.2%
reduction in this single sample. Total job time increased by 2,038 seconds, or
112.5%, because the seven Cargo action steps each ran the workspace check. Run
wall time increased from 481 seconds to 898 seconds.

## Cache evidence

The seed's BoringCache Cargo step took 343 seconds. Its native evidence reports
1,453 compiler requests, 521 executed requests, 3 hits, 518 misses, a 0.6% hit
rate, and no cache, read, or write errors. It published the Cargo registry cache
and index. The log states that the Cargo target save was skipped because the
wrapped command changed the source checkout. `cargo-git-db` had no local state.

Every restore action restored the 60.68 MB Cargo registry cache and the 42.64 MB
registry index. Every action missed `cargo-git-db` and
`datafusion-feature-check-target`, while compiler-cache preflight found
`datafusion-feature-check-compiler`.

Each restore action then executed the configured workspace check. Its native
sccache evidence was identical across the seven jobs: 1,453 requests, 521
executed requests, 520 hits, 1 miss, a 99.8% hit rate, no cache or read errors,
and one write error. The seven later feature-check sections recorded zero
sccache requests, so this run does not show that those commands used the remote
compiler cache.

The workflows used GitHub OIDC, `boringcache/one` v1.30.4 at immutable commit
`1039999c65011be670f5655e0e48ad556188ab12`, and Mozilla's official sccache
setup action at immutable commit `fc920bf0ec8de6ee65d409111f7ec508035751ba`.
Restore jobs used `trust-policy: restore` and did not publish.

The production evidence connector did not expose the new
`boringcache/datafusion` workspace to this session. The cache facts above come
from the BoringCache action evidence artifacts and GitHub logs.

## Qualification

The result is inconclusive for DataFusion's issue #25148. A valid follow-up must
first make the Cargo target publishable and must configure cache reuse without
running the workspace check once per consumer job. It should then compare the
parallel feature jobs against DataFusion's current warmed
`Swatinem/rust-cache` baseline. This run's baseline was cold and is not that
provider comparison.
