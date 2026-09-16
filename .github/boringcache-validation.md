# DataFusion BoringCache validation

## Issue-bounded result

| Upstream pain | Exact experiment | Measured result | Bounded verdict |
| --- | --- | --- | --- |
| [Issue #25148](https://github.com/apache/datafusion/issues/25148) investigates DataFusion's high GitHub Actions usage and proposes reusing Rust compilation outputs across overlapping feature-check jobs. | Preserve the complete 26-job Rust workflow and compare its existing GitHub cache path with one shared BoringCache workspace-check producer and compatible Linux consumers. Run a cold pair followed by five adjacent real upstream commits on unchanged runners. | All pairs passed, but the five warmed BoringCache runs used 591.2 job minutes versus 565.8 for GitHub: 25.4 more minutes, or 4.5%. Four of six pairs used more BoringCache runner time. | This configuration does not address the issue's goal of reducing total Rust CI minutes. It proves correct cross-job rolling reuse, but restoring roughly 1.1 GB into 15 consumers costs more than the reused producer output saves. |

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

1. Seed the current upstream GitHub cache path, then run both providers at the
   same source revision. This first BoringCache run is its cold seed, so it is
   reported separately from the warmed rolling sequence.
2. Advance through five adjacent real upstream commits. Run both providers
   once at each commit without overlapping BoringCache publishers that use the
   same rolling tag.
3. Compare successful complete runs using total job minutes, workflow critical
   path, individual job and build-command time, and provider cache evidence.
4. Treat cache misses, transport failures, cancelled runs, and failed workload
   checks as invalid performance samples.

All BoringCache calls use the released Cargo product path from
`boringcache/one` v1.30.4 pinned to
`1039999c65011be670f5655e0e48ad556188ab12`. The workflow uses GitHub OIDC
and contains no static BoringCache token. The cache identity includes platform,
and publication is limited to the trusted validation branch. Cargo target reuse
is limited to compatible Linux stable-toolchain jobs; macOS, wasm, MSRV,
formatting, and other non-compatible jobs keep their upstream behavior.

This is a full-workflow measurement of one shared workspace-check producer,
not a claim that every target directory produced by every job is persisted.
Cargo mode owns the producer's restore, command, and publication lifecycle;
the workflow does not add a custom cache transport or finalizer.

## Result

All six paired revisions completed the full 26-job workflow successfully. The
five rolling revisions followed the cold BoringCache seed in source order.

| Source | Change | BoringCache run | GitHub run | BoringCache job minutes | GitHub job minutes | Difference | BoringCache longest job | GitHub longest job | BoringCache wall time | GitHub wall time |
| --- | --- | --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `463939a21` | validation base; BoringCache cold seed | [35024680298](https://github.com/boringcache/datafusion/actions/runs/35024680298) | [35024660553](https://github.com/boringcache/datafusion/actions/runs/35024660553) | 126.0 min | 130.8 min | -4.8 min | 17.6 min | 16.8 min | 22.9 min | 23.2 min |
| `5470eb472` | `rstest` 0.26.1 to 0.27.0 | [35026704982](https://github.com/boringcache/datafusion/actions/runs/35026704982) | [35026704805](https://github.com/boringcache/datafusion/actions/runs/35026704805) | 118.5 min | 114.2 min | +4.4 min | 15.0 min | 14.9 min | 17.1 min | 19.7 min |
| `44252b8eb` | `dirs` 6.0.0 to 7.0.0 | [35028553727](https://github.com/boringcache/datafusion/actions/runs/35028553727) | [35028551379](https://github.com/boringcache/datafusion/actions/runs/35028551379) | 123.8 min | 117.4 min | +6.4 min | 15.1 min | 14.8 min | 16.6 min | 18.9 min |
| `b4ff082b3` | three other Cargo dependency updates | [35030376346](https://github.com/boringcache/datafusion/actions/runs/35030376346) | [35030376517](https://github.com/boringcache/datafusion/actions/runs/35030376517) | 116.3 min | 117.0 min | -0.7 min | 15.0 min | 15.0 min | 18.7 min | 19.5 min |
| `2f24a62fa` | CodeQL action updates | [35032089898](https://github.com/boringcache/datafusion/actions/runs/35032089898) | [35032091009](https://github.com/boringcache/datafusion/actions/runs/35032091009) | 112.7 min | 105.8 min | +6.8 min | 14.3 min | 13.2 min | 15.7 min | 17.2 min |
| `e6cbe4394` | `setup-uv` 10.0.1 to 10.1.0 | [35033499090](https://github.com/boringcache/datafusion/actions/runs/35033499090) | [35033499118](https://github.com/boringcache/datafusion/actions/runs/35033499118) | 120.0 min | 111.5 min | +8.5 min | 15.9 min | 13.3 min | 17.6 min | 17.3 min |

The warmed five-commit sequence used 591.2 total BoringCache job minutes and
565.8 total GitHub job minutes. BoringCache used 25.4 more runner-minutes, or
4.5%. Including the cold seed pair, BoringCache used 717.2 minutes and GitHub
used 696.6 minutes, a 20.6-minute or 3.0% increase. Four of the six paired
revisions used more BoringCache runner time. The measured result therefore does
not support claiming that this BoringCache configuration reduces DataFusion's
current Rust workflow minutes.

The observed workflow wall time was lower for BoringCache in five pairs, but
that result does not establish a shorter critical path. The providers ran
concurrently, GitHub scheduled their jobs independently, and the BoringCache
longest job was equal or slower in five pairs. Queue and job-start variation
can change wall time without reducing compute. Total completed job minutes and
the longest completed job do not show a performance improvement.

## Cache evidence

The cold producer created a 1.07 GB workspace-check entry containing about
7,800 files. Each rolling producer restored the previous entry, ran the real
workspace check, and published changed chunks:

| Source | Restored | Workspace check | Saved | Missing target chunks uploaded |
| --- | ---: | ---: | ---: | ---: |
| `5470eb472` | 1.07 GB in 8.3 s | 46.6 s | 1.09 GB in 6.2 s | 40 |
| `44252b8eb` | 1.09 GB in 5.9 s | 4.4 s | 1.09 GB in 3.2 s | 2 |
| `b4ff082b3` | 1.09 GB in 9.9 s | 2 min 9 s | 1.15 GB in 5.5 s | 64 |
| `2f24a62fa` | 1.15 GB in 6.7 s | 4.8 s | 1.15 GB in 3.7 s | 2 |
| `e6cbe4394` | 1.15 GB in 13.5 s | 3.6 s | 1.15 GB in 3.3 s | 2 |

This proves correct rolling, content-addressed publication through the official
Cargo path. It also identifies the limiting configuration: 15 compatible
Debian consumers each restore roughly 1.1 GB. Only the first direct Cargo
command in each consumer is wrapped by released Cargo mode. Later commands
reuse that restored state locally, but their additional target state is not
published. The host-Ubuntu `datafusion-cli` consumer correctly misses the
Debian entry because the cache identity isolates platforms; it then compiles
normally and remains restore-only. The workflow does not seed a separate Ubuntu
target entry.

Runs [35026727297](https://github.com/boringcache/datafusion/actions/runs/35026727297)
and [35026727077](https://github.com/boringcache/datafusion/actions/runs/35026727077)
were cancelled and excluded because they began before the preceding publisher
finished. Reusing their measurements would invalidate the rolling sequence.
The earlier seven-job workflow is also excluded because it did not measure the
complete issue.
