# jsonrpc-bench: project plan

This repository is maintained by the author of JSON-RPC.NET, one of the measured libraries. Everything below is written so that the maintainers of the other libraries can check it, object to it and reproduce it.

jsonrpc-bench compares JSON-RPC 2.0 server implementations across languages. It gives every implementation the same requests through one load generator, admits a result only after the server passes a shared conformance corpus, and publishes the raw data of every run. This file records what the suite measures, how it measures it, how the repository is laid out, the rules that keep it fair, the risks that are known but not solved, and the milestones. It was decided in a design review on 2026-09-28 that worked from downloaded prior art (the sources are listed at the end); the rejected alternatives are recorded with their reasons.

Status: design decided, nothing implemented yet. The first milestone is described in [Milestones](#milestones).

## Contents

1. [Scope](#scope)
2. [Targets](#targets)
3. [Transports](#transports)
4. [Workloads](#workloads)
5. [Measurement](#measurement)
6. [Conformance gate](#conformance-gate)
7. [Repository layout and contracts](#repository-layout-and-contracts)
8. [Results, CI and publishing](#results-ci-and-publishing)
9. [Fairness and governance](#fairness-and-governance)
10. [Claims policy](#claims-policy)
11. [Cost](#cost)
12. [Known risks and open questions](#known-risks-and-open-questions)
13. [Milestones](#milestones)
14. [Relationship to the JSON-RPC.NET repository](#relationship-to-the-json-rpcnet-repository)
15. [Rejected alternatives](#rejected-alternatives)
16. [Sources](#sources)

## Scope

- **First-class targets are JSON-RPC 2.0 servers.** The suite answers "how do JSON-RPC server libraries compare when they serve the same calls over the same transport".
- **Other RPC stacks are reference rows, not contestants.** gRPC, FlatBuffers over gRPC, Cap'n Proto, Thrift, MessagePack-RPC or Connect enter only through a scope proposal that names the whole-stack question the row answers, its native transport, its client driver, its conformance gate and its budget. The published reference panel holds at most one row per serialization format, so it never becomes a second leaderboard. Every reference chart carries the caption "an end-to-end stack comparison, not a parser or serializer ranking".
- **Axes are recorded separately on every run:** protocol and envelope, serialization format, transport and framing, language, library and version, tier, workload, and client driver. A combination is measured only where the stack supports it natively. No chart plots "Protocol Buffers versus JSON"; it plots two libraries with their axes in the label.
- **gRPC across languages is already measured** by [grpc_bench](https://github.com/LesnyRumcajs/grpc_bench) (42 active and 6 on-hold implementations, including Connect). This suite cites and links it as external context and does not fold its numbers into our charts. It differs from grpc_bench in three ways: a conformance gate, open-loop latency, and a results schema with machine and cost capture.
- **No cross-language JSON-RPC server benchmark or JSON-RPC conformance suite was found** during the review. The conformance corpus here is written to be useful on its own, and may later be split out.
- **Name.** The repository stays `jsonrpc-bench`. A general name is reconsidered only after a second protocol family has a repeatable, independently reviewed full cohort; GitHub keeps redirects, so a later rename stays cheap.

## Targets

**Milestone 1 measures four servers, each in its default tier, each pinned by its native lockfile with the artifact hash recorded:**

| Folder | Library | Language | Version | Native transports used in milestone 1 |
| --- | --- | --- | --- | --- |
| `targets/csharp-jsonrpc-net` | JSON-RPC.NET (`AustinHarris.JsonRpc.AspNetCore`) | C# | 2.0.1 | http1, tcp-raw |
| `targets/csharp-streamjsonrpc` | StreamJsonRpc | C# | 2.25.29 | tcp-nl, tcp-cl |
| `targets/rust-jsonrpsee` | jsonrpsee | Rust | 0.26.0 | http1 |
| `targets/go-jrpc2` | jrpc2 | Go | v1.3.5 | http1, tcp-nl, tcp-cl |

Plus one diagnostic row outside the target count: `targets/ceiling-echo`, a newline echo server written in the load generator's language that does no JSON parsing (`kind = "ceiling"`). It is a canary for host drift and a sanity check. It is never by itself the proof that the load generator has headroom.

**Tiers.** The default tier is the serving path in the library's own README or quick start, with per-request logging off for everyone. For StreamJsonRpc that means its default Json.NET formatter; the System.Text.Json formatter is a tuned variant. Tiers are variants inside the target's folder (`tiers/default`, `tiers/tuned`) so a tuned variant keeps the target's identity. A tuned variant lists its exact diff from the default and gets its own row. Tuned tiers are allowed from milestone 2.

**Versions.** A version bump is a pull request that creates a new target-version cohort. Nothing floats to "latest". Results from different cohorts are never joined in one comparison.

**Admission rule** (anyone may propose a target by pull request):

1. It is a server-side JSON-RPC 2.0 implementation with a distributable release.
2. It has a documented server example that the target folder follows.
3. It has one measurable adoption or maintenance signal, named in `target.toml`: registry downloads, a named production deployment, or a dependent project. There is no star threshold.
4. It passes the conformance corpus on every transport it declares.
5. It names a maintainer contact, and an upstream review request is filed before its first published run (see [governance](#fairness-and-governance)).

**Next in line:** vscode-jsonrpc, jayson and sourcegraph/jsonrpc2, then Python and Java candidates that pass the rule. Java and C++ are open slots: the survey found no candidate that currently meets the rule.

## Transports

Each transport is a cohort with its own tables. **There is no rank across cohorts.** A cohort is published once it has two conformant native targets.

| Transport | Definition | Milestone 1 targets |
| --- | --- | --- |
| `http1` | One JSON-RPC document per HTTP/1.1 POST, keep-alive, one outstanding POST per connection | JSON-RPC.NET (`MapJsonRpc`), jsonrpsee, jrpc2 (`jhttp`) |
| `tcp-nl` | TCP, one single-line JSON document per line in both directions. Requests end with LF. A reply may end with LF or CR LF: StreamJsonRpc's handler defaults to CR LF, the terminator is declared in `target.toml`, and the generator accepts either. A raw line break inside a document is a framing failure. `TCP_NODELAY` on both sides | StreamJsonRpc (`NewLineDelimitedMessageHandler`), jrpc2 (`channel.Line`) |
| `tcp-cl` | TCP with LSP `Content-Length` headers, `TCP_NODELAY` on both sides | StreamJsonRpc (`HeaderDelimitedMessageHandler`), jrpc2 (`channel.LSP`) |
| `tcp-raw` | TCP, self-delimiting JSON documents with no delimiter on replies | JSON-RPC.NET only: a single-target diagnostic lane, not a comparison |

JSON-RPC.NET's raw connection handler accepts newline-separated input but writes replies back to back with no delimiter, so it is not a native member of `tcp-nl` or `tcp-cl`. It joins those cohorts when the library ships native framers, not through a suite adapter.

**Adapters.** When a library lacks a transport natively the cell is `n/a`. An adapter may appear only outside the headline tables, with its source, its CPU cost and any changed semantics displayed.

**Later:** WebSocket, HTTP/2 and stdio in milestone 2, with MCP as a separate application-protocol lane (MCP's Streamable HTTP carries one message per POST and forbids batches). Unix sockets and TLS variants come later.

## Workloads

Every request carries `"jsonrpc":"2.0"`, ids mix integers and strings, and operands and strings are drawn from a published seed. There is no fixed ring of identical requests: a server must not be able to assume that requests repeat.

**Milestone 1:**

| Workload | Content | Counted as |
| --- | --- | --- |
| `calc5` | The five historical calls (`add`, `addInt`, `NullableFloatToNullableFloat`, `Test2`, `StringMe`) with seeded operands | valid RPC/s |
| `echo_utf8` | A seeded 1,024-byte UTF-8 string with escapes and non-ASCII characters, echoed back | valid RPC/s |
| `errors` | `calc5` with 10 % method-not-found (`-32601`) and 10 % invalid-params (`-32602`) mixed in | an error reply counts only when its code matches; the denominator is every request sent |
| `batch100` | Arrays of 100 `calc5` members | valid RPC/s and documents/s |

Sweeps: `calc5` at 1, 8 and 32 connections, and on stream transports at pipeline depth 1 and 256. The other workloads run at 32 connections and the lane's default depth.

Notifications are a conformance case in milestone 1. They become a timed workload in milestone 2, once the target contract has a server-side completion counter: a notification has no reply, so counting sends measures the client, not the server.

**Milestone 2 and later:** 64 KiB and 1 MiB payloads, nested objects with escaped strings, a table of 256 registered methods, a method that awaits a 1 ms timer, mixed batches, timed notifications.

**Rules against biased workloads.** A workload proposed by a contestant's maintainer, this repository's owner included, needs approval from an outside reviewer. No workload may exercise a path that only one library has. `calc5` is kept for continuity with published figures, not because it is representative; it is never the only headline workload.

## Measurement

**One load generator.** `loadgen/` is written in Rust, with codecs for `http1`, `tcp-nl`, `tcp-cl` and `tcp-raw` over one scheduler, one id matcher and one validator. Every reply is parsed and matched by id, and batch members are counted individually. A JSON-RPC error inside an HTTP 200 is a failure unless the workload expects that code. That jsonrpsee is also written in Rust is disclosed; the generator never links a contestant's code.

**Two separate experiments:**

- **Saturation (closed loop).** It reports throughput only. No percentile is ever reported from a closed-loop run, because closed-loop generators hide queueing delay (coordinated omission).
- **Latency (open loop).** Requests are sent at a fixed rate, and latency is recorded from each request's intended send time into an HdrHistogram. There is no `recordValueWithExpectedInterval` correction on open-loop data, which would correct twice. Milestone 1 measures `calc5` at 32 connections at 50, 80 and 95 % of that cell's measured saturation, 20 s per step, and reports p50, p90, p99, p99.9 and max, with missed schedules and timeouts counted.

**Windows.** Warm-up ends when three consecutive one-second throughput samples are within 5 % of each other, with a minimum of 3 s and a cap of 10 s. Then 5 s are timed, and the per-second timeline is kept in every result.

A run that has not settled by the cap is `nonsteady`. A cell with any nonsteady run is ineligible for headline tables and charts. It stays in the raw data and is shown as nonsteady in the explorer. If a target is nonsteady in most of its cells, the next approved run raises the warm-up cap for every target alike, never for one target alone. Window lengths are revisited once the first milestone's measured cost is known.

**Headroom gate, on every cell.** The generator's CPU use must stay under 75 % of its CPU set, and doubling the generator's workers must change completions by less than 3 %. A cell that fails is marked generator-limited and not published. The ceiling row runs first and last in every job, and more than 10 % drift between the two invalidates the job.

**Topology.** Milestone 1 publishes one Hugging Face Jobs `cpu-performance` job (32 vCPU) with two disjoint CPU sets. The runner reads the job's actual `Cpus_allowed_list` and `cpu.max` and splits the allowed CPUs into two equal sets, one for the server and one for the generator. It records both lists and never assumes that CPUs 0 to 31 exist. If the allowed list is smaller than the quota implies, the run records that and the plan uses the smaller figure. The same approval covers one experiment with two jobs in a network group (a `cpu-performance` server and a `cpu-xl` client, with link RTT and bandwidth measured), because the documentation promises neither placement nor link quality. That experiment decides the milestone 2 default. Topology is an axis in every result file.

**Statistics.** Five interleaved runs per cell (A B C A B C ...). Reports show the five raw values, the median and the range. No confidence interval is claimed from five runs. A claim of a significant difference needs at least ten independent runs and a 95 % interval.

**Metrics:** valid RPC/s, documents/s, wire bytes, server and generator CPU-seconds per million valid calls (from cgroup `cpu.stat`), peak RSS of the target process, errors and timeouts, startup-to-ready, time-to-steady, offered rate, and runtime and GC settings.

## Conformance gate

- `corpus/conformance/` holds request and expected-reply cases. Replies are compared after parsing, matched by id, with whitespace and batch order ignored.
- **Required cases:** valid calls, integer and string ids, `-32700`, `-32600`, `-32601` and `-32602`, the null id, batches, and a notification-only batch that produces no reply. A target that fails a required case has no row for that transport.
- **Optional spec-edge cases** that a target fails are listed in `corpus/expected-failures/<target>.yaml` and shown next to its row.
- Semantic reply checks apply to reference rows too, even when their ecosystem's own conformance suite (gRPC interop, Connect conformance) passes.

## Repository layout and contracts

```
targets/<lang>-<library>/    target.toml, Dockerfile, tiers/, server source, README.md
corpus/                      VERSION, catalog/v1.yaml, workloads/ (generators and seeds), conformance/, expected-failures/
loadgen/                     the Rust load generator and the ceiling echo server
runner/                      run.py (cells, interleaving, captures), hf/launch.sh, hf/fetch.py
results/raw/<job-id>/        <run-id>.json per repetition (immutable), job.log
results/index.json           references every immutable run
charts/                      ingest, render, render --check
site/                        GitHub Pages explorer
PROJECT.md, CONTRIBUTING.md, THIRD_PARTY_NOTICES.md
```

**Per-target contract.**

- The Dockerfile is the only build recipe; nothing depends on toolchains installed on the host.
- The Dockerfile has an `artifact` stage that leaves a self-contained server in `/out/<target>/` with an executable `run`.
- `run` starts one foreground process configured by `BENCH_HOST`, `BENCH_PORT`, `BENCH_TRANSPORT` and `BENCH_TIER`, and exits nonzero for an unsupported transport or tier.
- Readiness is a TCP connect followed by a `calc5` probe.

CI builds and tests each target's image. A Hugging Face job runs one image, and Docker inside the job is not established by the documentation. So the runner image for a job is assembled from every target's `artifact` stage (multi-stage `COPY`) plus the load generator, which lets one job start targets one after another and interleave runs.

**`target.toml`** fields: slug, kind (`jsonrpc`, `reference`, `ceiling`), protocol, format, language and runtime version, library, package URL, exact version, lockfile hash, transports, tiers with their settings, GC mode, adoption signal, maintainer contact, upstream review status, and known conformance deviations.

**Catalogue.** `corpus/catalog/v1.yaml` defines operations independently of any IDL, with exact value semantics: a bounded integer sum, a varied UTF-8 echo, an object echo and a specified error. Per-stack schemas (`.proto`, `.fbs`) are added only when a reference row exists. The historical decimal call (`Test2`) is a diagnostic only: Protocol Buffers has no decimal, and the existing `calculator.proto` maps it to units and nanos.

**Pinning.** Native lockfiles are committed (`packages.lock.json`, `Cargo.lock`, `go.sum`, `package-lock.json`) and base images are pinned by digest. Monthly bump pull requests must pass conformance. A paid rerun still needs the owner's approval.

## Results, CI and publishing

**One immutable JSON file per repetition** holds: schema version, suite version and commit, SHA-256 of the corpus and catalogue, the generator build, target version, lock and image digests, both machine inventories, topology, commands and environment, offered, completed and failed calls, the per-second timeline, the histogram for open-loop runs, CPU and RSS, readiness and warm-up, and `cost_usd`. Charts never join results with unlike hashes.

**CI (GitHub Actions), on every pull request:** build and conformance for every affected target, and for all targets when the corpus, the generator, a toolchain image or the contract changes. It also runs a 5 s smoke run per target, printed as "not comparable", and `render --check`. No published number comes from a shared CI runner.

**Full runs** happen on Hugging Face Jobs, only after the owner approves them. Whether they start from the local launch script or from a `workflow_dispatch` with a Hugging Face token secret is an [open question](#known-risks-and-open-questions).

**Reuse from the JSON-RPC.NET repository.** The log framing (`== BEGIN` and `== END` markers), `fetch.py` and the `render --check` invariant are ported. Ingestion and chart grouping are rewritten, because the library's ingester hard-codes its own comparison rows.

**Publishing:** generated README tables, a GitHub Pages explorer, and `results/index.json` as a feed. Claims cite stable job and run permalinks, never a "latest" file.

**Suite versioning:** semantic versioning in `corpus/VERSION`. A change to workload semantics, the transport contract or the measurement algorithm is a major version. A target version bump creates a new cohort under the same suite version.

## Fairness and governance

- **Disclosure.** The first paragraph of README.md and PROJECT.md, and every chart footer, say: "This repository is maintained by the author of JSON-RPC.NET, one of the measured libraries."
- **Rules for targets.** The default tier is the library's documented serving path. There are no pre-serialized replies, no special cases for method names or corpus contents, and no assumption that requests repeat. Protocol work stays in the measured process. Special-casing found after publication removes the row until it is fixed.
- **Upstream review.** Before a target's first published run, a `review-request` issue names the folder, the configuration and the draft numbers, and the upstream project is pinged once. The review window is 14 days, and its status is shown next to the row. Lack of a reply is displayed, not treated as a veto. A maintainer's objection with a concrete reason pauses that row while it is reviewed.
- **Suite review.** Every change needs one suite maintainer plus either an outside approving reviewer or a seven-day open window. Changes to the JSON-RPC.NET target need an outside reviewer, or failing that a seven-day open window with the change listed in the results changelog.
- **Disputes.** A public issue labelled `dispute` marks the row as disputed in the explorer within 48 hours. Raw artifacts are kept, and the fix arrives as a correction pull request.
- **Losing rows get the same prominence as winning ones.** Nothing is omitted because it is unflattering.
- **Licensing.** Suite code is MIT. Result data is CC0, with contributors agreeing to that in CONTRIBUTING.md. Dependencies are consumed as packages and nothing is vendored. Anything copied, including the chart pipeline ported from the JSON-RPC.NET repository, is listed in THIRD_PARTY_NOTICES.md with its license.

## Claims policy

- A README may cite a number from this suite only with: the suite version, a run permalink, the date, the topology, the targets and tier, the workload, the transport, the metric and its spread. JSON-RPC.NET's README is the first to follow this.
- A ratio may compare only rows from the same run, transport and workload.
- There are no unqualified superlatives.

**Consequence.** The JSON-RPC.NET README currently says it answered the same calls at "about 15×" StreamJsonRpc's rate. That compares raw-document TCP with Content-Length TCP from a same-process harness, so it cannot be repeated under this policy. The first thing this suite changes is its own maintainer's claim.

## Cost

- An approved full run is capped at **$4**. The runner computes the plan's cost from the flavor rates, records `cost_usd` in every result, and refuses a plan over the cap without an explicit budget flag.
- **Rates** (Hugging Face Jobs, 2026-09-28): `cpu-performance` 32 vCPU at about $1.90 per hour, `cpu-xl` 16 vCPU at about $1.00 per hour, `cpu-upgrade` 8 vCPU at about $0.03 per hour.
- **Milestone 1 estimate.** The saturation grid has 63 cells (18 on `http1`, 45 on the stream transports), five runs each, at about 13 s per run including warm-up and restarts: about 70 minutes. Latency adds 8 lanes × 3 rates × 20 s × 5 runs, about 40 minutes. That is about $3.50 on one `cpu-performance` job. The two-job experiment (about 15 minutes at $2.90 per hour) would push the total over the cap, so it needs its own approval unless the measured plan comes in lower.

## Known risks and open questions

| # | Risk or gap | Handling |
| --- | --- | --- |
| 1 | JIT, tiered compilation and warm-up differ by runtime | Per-second timeline, steady-state rule, time-to-steady reported as a metric |
| 2 | Garbage-collector modes | Container default in the default tier, GC mode recorded, tuned tier may change it |
| 3 | A CPU quota is not CPU affinity | CPU sets applied; `cpu.max`, `Cpus_allowed_list` and `nproc` recorded. **Open:** whether the quota maps to physical CPUs |
| 4 | Kernel and socket settings | `TCP_NODELAY` on both sides is part of every TCP transport's definition; `somaxconn` and `tcp_rmem`/`tcp_wmem` recorded, not changed |
| 5 | Physical placement of two jobs, and link quality | RTT and bandwidth measured, placement recorded as unknown |
| 6 | TLS | Milestone 4, as separate rows |
| 7 | ARM | **Open:** Hugging Face Jobs lists no ARM flavor |
| 8 | Noisy neighbours on shared hosts | Interleaving, five runs, the ceiling canary first and last; over 10 % drift invalidates the job |
| 9 | The load generator's language biasing results | Headroom gate on every cell; from milestone 2, a second generator cross-checks one cell per job |
| 10 | Upstream releases changing results | Lockfiles, digests, cohorts, a quarterly re-run |
| 11 | Only tiny messages | `echo_utf8` in milestone 1; 64 KiB, 1 MiB and nested objects in milestone 2 |
| 12 | A one-sided comparison | Disclosure, upstream review, the claims policy, equal prominence for losing rows |
| 13 | Protocol edges (batches forbidden on MCP HTTP, stdio has one connection) | `n/a` rules in the runner, not footnotes |
| 14 | Cancellation, backpressure, long-lived streams, server-to-client calls | **Open:** follow-ups, not yet designed |
| 15 | A target doing work outside its measured process | Per-process CPU of the target and complete-reply validation recorded in every run |
| 16 | How full runs are launched | **Open:** the local launch script or a `workflow_dispatch` holding a Hugging Face token; the owner decides |
| 17 | Kalibera and Jones on rigorous benchmarking in reasonable time | **Open:** the paper could not be downloaded during the review; revisit before claiming significance |

## Milestones

### Milestone 1: four servers, four lanes, published

- **Targets:** `csharp-jsonrpc-net`, `csharp-streamjsonrpc`, `rust-jsonrpsee` and `go-jrpc2`, plus the `ceiling-echo` diagnostic.
- **Cohorts:** `http1` (three targets), `tcp-nl` (two), `tcp-cl` (two), and the `tcp-raw` diagnostic for JSON-RPC.NET.
- **Workloads:** `calc5`, `echo_utf8`, `errors` and `batch100`. Notifications are tested for conformance only. Open-loop latency is measured for `calc5`.
- **Historical cell.** The JSON-RPC.NET README's comparison shape is rerun exactly, as a labelled diagnostic: shared cores, 32 connections, 256 in flight, `calc5` with the `"2.0"` member, 2 s warm-up, 3 s timed. A gap of more than 25 % from the published JSON-RPC.NET and StreamJsonRpc figures triggers a written investigation of the cause, not a milestone failure. The client, process layout and topology all differ from the old harness.

**Acceptance:**

1. Conformance is green, with zero timed mismatches, for every published lane.
2. CI builds every image and runs conformance and smoke in under 20 minutes.
3. The headroom gate passes for every published cell.
4. Every cell has five interleaved runs from one owner-approved `cpu-performance` job under the $4 cap.
5. Raw per-run JSON, machine and cost capture, README tables, the Pages explorer and `results/index.json` are generated and checked.
6. A review-request issue is filed for each external target, with its status shown.

### Milestone 2: breadth

- **Targets:** vscode-jsonrpc, jayson, sourcegraph/jsonrpc2 and a Python target.
- **Transports:** WebSocket, HTTP/2, stdio, and the MCP lane.
- **Workloads:** timed notifications, 64 KiB and 1 MiB payloads, nested objects, the 1 ms await, and the 256-method table.
- **Measurement:** tuned tiers, a second generator as a cross-check, and the topology default chosen from the two-job experiment.

### Milestone 3: gRPC reference

- A gRPC reference row on the shared catalogue, with its own conformance and headroom gates.

### Milestone 4: topology and platform

- A topology study, TLS rows, ARM when a flavor exists, and cost and energy per call.

## Relationship to the JSON-RPC.NET repository

- **What leaves the library repository:** the `--compare` and `--sweep` modes, `GrpcCompare.cs`, `Protos/calculator.proto` (this catalogue keeps a copy), the StreamJsonRpc and gRPC package references, the comparison charts, and the README's comparison section.
- **When:** after one green published suite run and the upstream review window, and once a JSON-RPC.NET release whose README links this suite is published. Numerically reproducing the old rows is not the condition.
- **What stays in the library:** its own release gates (`--scale`, `--sync`, `--async`, `--kestrel`), the micro-benchmarks and its own charts.
- **Library-side follow-up:** native newline and Content-Length framers in JSON-RPC.NET's stream host, so it joins the `tcp-nl` and `tcp-cl` cohorts. That work belongs to the library, not to this suite.

## Rejected alternatives

| Alternative | Why it was rejected |
| --- | --- |
| A general rpc-bench grid (protocol × format × transport × language × library) | grpc_bench already covers gRPC across languages. No maintained cross-protocol server benchmark exists to build on. One workload cannot be fair to stacks whose strengths differ: Cap'n Proto's promise pipelining and FlatBuffers' zero-copy reads are invisible to small independent calls. The cell count is unaffordable under the per-run cap. |
| Entering the TechEmpower Framework Benchmarks | The project was archived on 2026-03-24 (last round 23, February 2025). Its tests are fixed HTTP contracts (`GET /json`, plaintext, Fortunes) that a JSON-RPC server can only meet by bypassing its RPC layer. |
| YCSB | It measures data stores, not RPC servers. |
| Keeping the comparison inside the JSON-RPC.NET repository | One process hosts every library, and the load generator shares the server's cores. It cannot include servers in other languages. A comparison owned by one contestant's repository reads as less neutral. |
| An adoption threshold by stars or downloads | It would have excluded jrpc2, a maintained server with a real serving path. One named, measurable signal is required instead. |
| Adding JSON-RPC.NET to `tcp-nl` through a suite adapter | Its replies are not line-delimited. A strict line client could not read them, and an adapter would hide that. |
| Flagging nonsteady runs but still publishing them | A just-in-time runtime that has not settled would be compared at a disadvantage. Nonsteady cells are kept in the raw data but excluded from headline tables. |
| Fixed CPU numbers (server 0 to 15, generator 16 to 31) | A job's allowed CPU list is not guaranteed to be 0 to 31. The sets are derived from `Cpus_allowed_list` at run time. |
| A bootstrap confidence interval from five runs | It would overstate what five runs show. The median and range are reported instead. |
| Percentiles from closed-loop runs | Coordinated omission hides queueing delay. Latency is measured open loop only. |
| A host-side `build.sh`/`run.sh` as the build contract | It reintroduces toolchain drift between CI and the benchmark host. Dockerfiles are the only recipe, and one job's runner image is assembled from their artifact stages. |
| An upstream veto over a target's configuration | An unanswered veto would block publication indefinitely. A time-boxed review window with visible status and an objection-pauses-the-row rule are used instead. |
| Folding grpc_bench's numbers into this suite's charts | Its generator (ghz) and workload differ. It is linked as external context. |

## Sources

Prior art and specifications consulted during the design review (all retrieved 2026-09-28):

- **Cross-implementation benchmarks.**
  - [TechEmpower FrameworkBenchmarks](https://github.com/TechEmpower/FrameworkBenchmarks) (archived) and its wiki pages on [framework test rules](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Project-Information-Framework-Tests-Overview) and [framework files](https://github.com/TechEmpower/FrameworkBenchmarks/wiki/Codebase-Framework-Files).
  - [LesnyRumcajs/grpc_bench](https://github.com/LesnyRumcajs/grpc_bench).
  - [dotnet/crank](https://github.com/dotnet/crank).
  - [nelsam/jsonrpc2 benchmarks](https://github.com/nelsam/jsonrpc2).
  - The [gRPC benchmarking guide](https://grpc.io/docs/guides/benchmarking/).
- **Load generation and latency.**
  - [wrk2](https://github.com/giltene/wrk2) (coordinated omission).
  - [HdrHistogram](https://github.com/HdrHistogram/HdrHistogram).
  - [vegeta](https://github.com/tsenart/vegeta), [ghz](https://github.com/bojand/ghz), [h2load](https://nghttp2.org/documentation/h2load-howto.html) and [flood](https://github.com/paradigmxyz/flood).
- **Specifications.**
  - [JSON-RPC 2.0](https://www.jsonrpc.org/specification).
  - [LSP 3.17 base protocol](https://microsoft.github.io/language-server-protocol/specifications/lsp/3.17/specification/).
  - [MCP transports](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports) and [MCP conformance](https://github.com/modelcontextprotocol/conformance).
  - [Ethereum JSON-RPC](https://ethereum.org/en/developers/docs/apis/json-rpc/) and [geth RPC](https://geth.ethereum.org/docs/interacting-with-geth/rpc).
  - [Connect protocol](https://connectrpc.com/docs/protocol/) and [Connect conformance](https://github.com/connectrpc/conformance).
  - [gRPC interop tests](https://github.com/grpc/grpc/blob/master/doc/interop-test-descriptions.md).
- **Other RPC stacks.**
  - [FlatBuffers gRPC](https://github.com/google/flatbuffers/tree/master/grpc).
  - [Cap'n Proto RPC](https://capnproto.org/rpc.html).
  - [Apache Thrift](https://github.com/apache/thrift).
  - [MessagePack-RPC](https://github.com/msgpack-rpc/msgpack-rpc).
- **Methodology papers.**
  - Laaber, Scheuner and Leitner, "Software microbenchmarking in the cloud. How bad is it really?", Empirical Software Engineering, 2019.
  - Papadopoulos et al., "Methodological Principles for Reproducible Performance Evaluation in Cloud Computing", IEEE TSE.
  - Bulej et al., "Duet Benchmarking", [arXiv:2001.05811](https://arxiv.org/abs/2001.05811).
  - Barrett et al., "Virtual Machine Warmup Blows Hot and Cold", [arXiv:1602.00602](https://arxiv.org/abs/1602.00602).
  - Chen and Revels, "Robust benchmarking in noisy environments", [arXiv:1608.04295](https://arxiv.org/abs/1608.04295).
  - Viotti and Kinderkhedia on JSON-compatible binary serialization, [arXiv:2201.03051](https://arxiv.org/abs/2201.03051) and [arXiv:2201.02089](https://arxiv.org/abs/2201.02089).
  - An empirical study of streaming serialization, [arXiv:2407.13494](https://arxiv.org/abs/2407.13494).
- **Hosting.** [Hugging Face Jobs configuration](https://huggingface.co/docs/hub/jobs-configuration) (flavors, network groups).
