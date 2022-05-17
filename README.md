This repo is collecting and proposing ideas and designs for a WASI concept tentatively called "Preview2", which would be a successor to "wasi_snapshot_preview1", featuring:

   - wit-based tooling
   - full sockets support
   - an improved preopen mechanism
   - static analyzability of environment variables
   - async "ready" (but not actual async execution yet; see below)
   - a design for dlls

This Preview2 iteration would not yet support:

   - Support for linking multiple components together.
   - Full async runtime support.

In return for this reduced scope, Preview2 is something we could plausibly roll out and start upstreaming into upstream toolchains in a shorter timeframe, and if we line everything up right, it'll help set up full async execution support in the future.

The [Task List] is a running list of the steps to get to Preview2.

The [Toolchain Design](ToolchainDesign.md) discusses the overall design.

[Task List]: https://github.com/sunfishcode/wasi-preview2/issues/1
