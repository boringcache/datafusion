# DataFusion BoringCache validation

## Scope

This validation measures the complete current `.github/workflows/rust.yml`
workflow for [DataFusion issue #25148](https://github.com/apache/datafusion/issues/25148).
It retains all 26 upstream jobs, runners, dependencies, commands, and checks.

A manual `cache_provider` input selects one of two paths:

- `github` keeps the current upstream combination of
  `Swatinem/rust-cache` and the workspace-check artifact introduced by
  DataFusion PR #25249.
- `boringcache` uses BoringCache Cargo mode for the workspace producer and
  restores that workspace-check target before the first direct Cargo command
  in each compatible Linux consumer. The producer publishes through GitHub
  OIDC. Consumers are restore-only. Later commands in a multi-command consumer
  reuse its local target state but do not publish it. Platform-specific and
  script-owned jobs remain unchanged and are included in total workflow time.

The separate seven-job demonstration workflow was removed. It did not measure
the issue's full workflow and its result does not support a performance claim.

## Protocol

1. Run the GitHub provider twice at one source revision so its normal rolling
   caches have a seed and a measured warm run.
2. Run BoringCache twice at the same revision: one seed and one same-source
   warm run.
3. Advance through adjacent real upstream commits. Run both providers once at
   each commit.
4. Compare successful complete runs using total job minutes, workflow critical
   path, individual job and build-command time, and provider cache evidence.
5. Treat cache misses, transport failures, cancelled runs, and failed workload
   checks as invalid performance samples.

All BoringCache calls use the released Cargo product path from
`boringcache/one` v1.30.4 pinned to
`1039999c65011be670f5655e0e48ad556188ab12`. The workflow uses GitHub OIDC
and contains no static BoringCache token. The cache identity includes platform
and branch. Cargo target reuse is limited to compatible Linux stable-toolchain
jobs; macOS, wasm, MSRV, formatting, and other non-compatible jobs keep their
upstream behavior.

This is a full-workflow measurement of one shared workspace-check producer,
not a claim that every target directory produced by every job is persisted.
Cargo mode owns the producer's restore, command, and publication lifecycle;
the workflow does not add a custom cache transport or finalizer.

## Result

Pending complete workflow runs. Do not use the earlier seven-job timings as a
product comparison.
