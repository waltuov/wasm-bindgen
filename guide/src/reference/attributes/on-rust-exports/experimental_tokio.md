# `experimental_tokio`

> **Experimental.** This attribute is only supported on the
> [`wasm32-unknown-emscripten` target](../../emscripten.md) and depends on
> Tokio's Emscripten event-loop support, which has not yet shipped in a Tokio
> release. Using it on any other target is a compile error.

When attached to an exported `async fn`, the future is driven as a root on a
Tokio event-loop runtime instead of the `wasm-bindgen-futures` executor, with
its outcome bridged to the returned `Promise`. `tokio::spawn`, timers and
Tokio I/O then work inside the export without JSPI: the runtime lowers only
its wait onto the host event loop.

```rust
#[wasm_bindgen(experimental_tokio)]
pub async fn fetch(req: Request) -> Response {
    // tokio::net, tokio::time, tokio::spawn are all usable here
}
```

By default every such export shares the thread's ambient runtime — one timer
wheel, one I/O driver, one scheduler — built with `enable_all` on first use.
A custom builder configuration can be installed ahead of time with
`wasm_bindgen_futures::tokio::try_set_ambient`.

When called from a top-level JS call the export drives synchronously before
returning, so the first poll happens immediately like `future_to_promise`.
When re-entered from inside a task it only queues the root, and the drive in
progress picks it up.

## `experimental_tokio = "isolated"`

Builds a fresh runtime for each invocation, torn down with native `Runtime`
drop semantics once the root future settles: tasks still in flight are
dropped and the reactor is closed. This is for multiplexed hosts where one
invocation's event loop must not perform I/O on behalf of another's context.

```rust
#[wasm_bindgen(experimental_tokio = "isolated")]
pub async fn handle(req: Request) -> Response {
    // ...
}
```

## Requirements

- The `wasm32-unknown-emscripten` target.
- The `tokio` feature of `wasm-bindgen-futures`.
- `--cfg tokio_unstable` in `RUSTFLAGS`, as the event-loop runtime is an
  unstable Tokio API.

Panics inside the future are caught by Tokio's task harness and reject the
`Promise` with an error describing the failed task.
