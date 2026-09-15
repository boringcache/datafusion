# DataFusion BoringCache validation

## Result

BoringCache One v1.30.4 correctly published one Cargo target snapshot and
restored it into all seven feature-check jobs. This single sample did not
reduce CI time. Total job time increased by 108 seconds, or 6.0%, and the sum
of measured check steps increased by 175 seconds, or 14.4%.

The result validates cross-job Cargo target correctness for DataFusion issue
#25148. It does not support a performance claim for this matrix or this 1.07 GB
workspace target.

All compared runs used upstream source
`7917a9a65a0343a6b5536299de5482a899d77c51`, Cargo.lock SHA-256
`c40522a03c46b513b77909c0e90e1c4198ddccafd1df1c7f856a7800c374471d`,
Rust 1.98.1, and Cargo 1.98.1. The seven feature jobs retained their upstream
runner expressions and ran in parallel. The corrected seed and restore used
validation workflow commit `ec8cf31feabde18f36f993dfbe8af3d1b43204d2`.

## Compared runs

- [Cold baseline](https://github.com/boringcache/datafusion/actions/runs/34979495334): success, workflow commit `f6a09a63793335cd429aaa1527f1ecc88de74cc4`.
- [Corrected trusted seed](https://github.com/boringcache/datafusion/actions/runs/35001645961): success, workflow commit `ec8cf31feabde18f36f993dfbe8af3d1b43204d2`.
- [Corrected restore fan-out](https://github.com/boringcache/datafusion/actions/runs/35002214116): success, workflow commit `ec8cf31feabde18f36f993dfbe8af3d1b43204d2`.

Times come from GitHub's job and step timestamps. “Check steps” includes the
Cargo Action that runs the first check in each restore job and the remaining
unchanged check commands. Parallel job totals measure runner use, not workflow
wall time.

| Feature job | Baseline job | Restore job | Change | Baseline check steps | Restore check steps | Cargo Action | Target restore |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| datafusion-common | 128 s | 149 s | +21 s | 54 s | 63 s | 39.4 s | 18.8 s |
| datafusion-substrait | 477 s | 583 s | +106 s | 333 s | 495 s | 63.6 s | 7.6 s |
| datafusion-proto | 240 s | 252 s | +12 s | 160 s | 176 s | 50.0 s | 6.7 s |
| datafusion-ffi | 156 s | 127 s | -29 s | 74 s | 50 s | 45.9 s | 10.0 s |
| datafusion | 435 s | 430 s | -5 s | 362 s | 367 s | 82.1 s | 12.4 s |
| datafusion-functions | 179 s | 179 s | 0 s | 103 s | 105 s | 41.3 s | 8.6 s |
| datafusion-spark | 197 s | 200 s | +3 s | 127 s | 132 s | 49.9 s | 6.2 s |
| **Total** | **1,812 s** | **1,920 s** | **+108 s** | **1,213 s** | **1,388 s** | **372.2 s** | — |

The Cargo Action values are the v1.30.4 evidence durations, so their sum can
differ from the rounded GitHub step total. Substrait's `protoc` check took 289
seconds in the restore run and accounts for most of the aggregate regression;
the baseline step took 196 seconds. One sample cannot separate that variance
from cache effects.

## Cache evidence

The corrected seed restored the existing 60.68 MB Cargo registry cache and
42.69 MB registry index, then ran the workspace check in 204.9 seconds for the
complete Cargo Action. Cargo reported 3m13s for the check itself. The seed
published a fresh `datafusion-issue-25148-cargo-v4-target-debian-13-x86_64`
entry containing 1.07 GB and 7.8K files in 8.0 seconds. It also republished the
two dependency entries. `cargo-git-db` had no local state.

Every restore job reported `cache_result=hit`, `cache_hit=true`,
`target_cache_hit=true`, and `cache_hit_restore_only`. Each restored three of
four planned entries: the target, registry cache, and registry index. Target
transfer took 6.2 to 18.8 seconds per job. After each restore, the Cargo
freshness check reported 4,011 unchanged sources and zero changed or new
sources, plus 564 unchanged directories and zero changed or new directories.

The workflow used GitHub OIDC without static BoringCache tokens. The seed used
`trust-policy: publish`; all seven consumers used `trust-policy: restore` and
did not publish. `boringcache/one` was pinned to immutable commit
`1039999c65011be670f5655e0e48ad556188ab12`, which installed CLI v1.30.4.
Compiler-cache support was disabled, so the test measured Cargo dependency and
target archives only.

DataFusion's setup exports `-C incremental=false`, which rustc interprets as a
relative incremental-output directory named `false`, not a Boolean setting.
The corrected Cargo Action steps override that malformed flag with
`-C debuginfo=line-tables-only`; DataFusion's `profile.ci` continues to set
`incremental = false`. This keeps the seed checkout clean through target
publication without changing the check command or upstream source.

## Excluded evidence

These runs remain useful for diagnosing the integration but are excluded from
the performance comparison:

- [Initial seed](https://github.com/boringcache/datafusion/actions/runs/34980483355) and [invalid restore](https://github.com/boringcache/datafusion/actions/runs/34982773905): the restore Action repeated the full workspace check in every consumer before the assigned feature checks.
- [Cancelled restore](https://github.com/boringcache/datafusion/actions/runs/34981534170): cancelled before it could provide a complete comparison.
- [Child-plan seed](https://github.com/boringcache/datafusion/actions/runs/34998531410) and [dependency-only restore](https://github.com/boringcache/datafusion/actions/runs/34999307820): the command shape was correct, but the seed refused to publish the target after the command created the untracked `false/` tree. Consumers restored only the two dependency entries.
- [Diagnostic seed](https://github.com/boringcache/datafusion/actions/runs/35000545066): recorded 1,338 untracked paths, all under `false/`, and no tracked changes or other untracked roots. The target was not published.

## Qualification

The released Cargo adapter satisfies the correctness and trust requirements:
one trusted job publishes, seven restore-only jobs reuse the same target, each
upstream feature-check command runs once, and the matrix remains parallel.
The measured target is too large to claim a speed improvement from this run.
Any follow-up should reduce the target boundary or compare another native cache
mode, then repeat the same cold-seed-restore measurement. Outreach should state
the correctness result and the measured regression rather than claim faster CI.
