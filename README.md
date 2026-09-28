# jsonrpc-bench

This repository is maintained by the author of JSON-RPC.NET, one of the measured libraries.

jsonrpc-bench compares JSON-RPC 2.0 server implementations across languages. Every server gets the same requests from one load generator over the same transport. A server's numbers count only after it passes a shared conformance corpus, and the raw data of every run is published so anyone can check it.

**Status:** the design is decided and nothing is implemented yet. [PROJECT.md](PROJECT.md) records what the suite measures, how it measures it, the repository layout, the fairness and claims rules, the known risks, and the milestones.

## Planned first milestone

| Library | Language | Transports |
| --- | --- | --- |
| JSON-RPC.NET | C# | HTTP/1.1 (the raw TCP lane is a single-target diagnostic) |
| StreamJsonRpc | C# | newline-framed TCP, Content-Length TCP |
| jsonrpsee | Rust | HTTP/1.1 |
| jrpc2 | Go | HTTP/1.1, newline-framed TCP, Content-Length TCP |

Each transport is its own comparison; there is no ranking across transports. Other RPC stacks such as gRPC appear only as labelled reference rows. For gRPC across many languages, see [grpc_bench](https://github.com/LesnyRumcajs/grpc_bench).

## Taking part

If you maintain one of these libraries, or a JSON-RPC server that should be here, open an issue. Before a library's first published run, its maintainers get a review request covering its folder, configuration and draft numbers. The admission rule and the review process are in [PROJECT.md](PROJECT.md#fairness-and-governance).

## License

Code is MIT ([LICENSE](LICENSE)). Published result data will be CC0.
