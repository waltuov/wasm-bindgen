# Emscripten Target

`wasm-bindgen` supports the `wasm32-unknown-emscripten` target in addition to
`wasm32-unknown-unknown`. Where `wasm32-unknown-unknown` is a bare target whose
`std` modules like `std::fs` and `std::net` return errors, Emscripten links a
libc, an in-memory file system, POSIX-style APIs and its own JavaScript
runtime around the Wasm, so a larger portion of `std` works out of the box.

> **Experimental.** Emscripten support is newer than the rest of `wasm-bindgen`
> and is still being smoothed out. The build flow below is the supported one;
> flags and output shape may still change.

You generally want Emscripten when:

- You need `std::time::{Instant, SystemTime}`, `std::env`, or `std::fs`
  (backed by MEMFS).
- You need `HashMap` with its default random state, or `getentropy`-backed
  randomness.
- You are linking Rust against C/C++ sources.
- You need a Tokio runtime inside an exported async function (see
  [Tokio](#tokio) below).

Otherwise `wasm32-unknown-unknown` stays the right default: it has the
smallest runtime and the fastest cold start.

## How it works

Instead of running the `wasm-bindgen` CLI yourself after linking, Emscripten
runs it for you. rustc drives `emcc` as the linker for the Emscripten target,
and when `-sWASM_BINDGEN` is passed, `emcc` detects the marker section the
`wasm-bindgen` crate embeds for Emscripten builds, runs the `wasm-bindgen`
CLI over the linked Wasm as a post-link step, and integrates the generated
bindings (`library_bindgen.js`) into its own JavaScript output. The clean
`#[wasm_bindgen]` API — free functions, classes, enums, namespaces — is then
surfaced by Emscripten's module wrapper, while the raw Wasm exports and
`wasm-bindgen`'s internal glue are kept off the public surface.

`-sWASM_BINDGEN` is a no-op for inputs without the marker section, so it can
be passed unconditionally.

## Prerequisites

- The Rust target: `rustup target add wasm32-unknown-emscripten`.
- [Emscripten](https://emscripten.org/docs/getting_started/downloads.html)
  **6.0.10 or newer**, with `emcc` on `PATH` (rustc uses it as the linker).
- The `wasm-bindgen` CLI on `PATH`, at **exactly** the version of the
  `wasm-bindgen` crate in your `Cargo.lock`, e.g.
  `cargo install wasm-bindgen-cli --version 0.2.128`. `emcc` invokes it by
  name.

## Crate shape

Emscripten packages are **binary crates**. rustc links Emscripten `bin`
targets through `emcc` as self-contained main modules; a `cdylib` is instead
linked as an Emscripten *side module* (`-sSIDE_MODULE=2`), a relocatable
object for Emscripten's dynamic linking rather than a usable package.

`main()` runs automatically when the module initializes (the Emscripten
idiom) and may be empty. The package API is the `#[wasm_bindgen]` exports.

```rust
use wasm_bindgen::prelude::*;

#[wasm_bindgen]
pub struct Greeter {
    greeting: String,
}

#[wasm_bindgen]
impl Greeter {
    #[wasm_bindgen(constructor)]
    pub fn new(greeting: String) -> Greeter {
        Greeter { greeting }
    }

    pub fn greet(&self, name: &str) -> String {
        format!("{}, {}!", self.greeting, name)
    }
}

fn main() {}
```

## Building

Select the target and pass the Emscripten settings as link arguments in
`.cargo/config.toml`:

```toml
[build]
target = "wasm32-unknown-emscripten"

[target.wasm32-unknown-emscripten]
rustflags = [
  "-Cpanic=abort",
  "-Cllvm-args=-enable-emscripten-cxx-exceptions=0",
  "-Crelocation-model=static",
  "-Clink-arg=-sWASM_BINDGEN",
  "-Clink-arg=-Wno-experimental",
  "-Clink-arg=-sMODULARIZE",
  "-Clink-arg=-sEXPORT_ES6",
]
```

Then a plain `cargo build` produces `<name>.js` and `<name>.wasm` under
`target/wasm32-unknown-emscripten/<profile>/`. There is no separate
`wasm-bindgen` or `wasm-opt` invocation: `emcc` runs its own optimization
pipeline at link time, driven by the rustc opt-level.

The codegen flags:

- `-Cpanic=abort` — `panic=unwind` is not yet supported across the
  `wasm-bindgen` boundary on Emscripten
  ([#5165](https://github.com/wasm-bindgen/wasm-bindgen/issues/5165)).
- `-Cllvm-args=-enable-emscripten-cxx-exceptions=0` — avoids pulling in a
  C++ exception runtime that `panic=abort` never uses.
- `-Crelocation-model=static` — PIC is not needed for a statically linked
  main module.

`-Wno-experimental` silences `emcc`'s warning that `-sWASM_BINDGEN` is an
experimental setting. Any other `emcc` setting is passed the same way, e.g.
`"-Clink-arg=-sSTACK_SIZE=8MB"` or `"-Clink-arg=-sALLOW_MEMORY_GROWTH"`.

### Output modes

Emscripten's usual output settings control how the API is exposed:

| Settings | Consumption |
|---|---|
| `-sMODULARIZE -sEXPORT_ES6` | A factory: `import Module from './name.js'; const mod = await Module(); new mod.Greeter(..)` |
| `-sMODULARIZE=instance -sEXPORT_ES6` | Named ESM exports plus an `init` default export: `import init, { Greeter } from './name.js'; await init();` |
| `-sWASM_ESM_INTEGRATION` | As above, with the Wasm itself imported as an ES module |

With the named-export modes, adding `-sAUTO_INIT` makes the module
self-initialize on import so no `init()` call is needed.

When the crate imports JavaScript modules (`#[wasm_bindgen(module = "...")]`
or [JS snippets](js-snippets.md)), the imports are emitted to a sidecar
`library_bindgen.extern-pre.js` that `emcc` prepends to its output, and the
`snippets/` directory is copied next to the output. Cargo only copies the
primary `.js` and `.wasm` out of `deps/`, so consume from — or copy from —
`target/wasm32-unknown-emscripten/<profile>/deps/` in that case.

### Linking as a static library

The other direction also works: build a `staticlib` with `cargo`, then let
`emcc` drive the link together with C/C++ sources:

```sh
emcc main.c target/wasm32-unknown-emscripten/release/libmylib.a -sWASM_BINDGEN -o out.js
```

`EMSCRIPTEN_KEEPALIVE` native exports and the `#[wasm_bindgen]` API compose
in the same module.

## wasm-pack

[`wasm-pack`](https://github.com/wasm-bindgen/wasm-pack) can drive the whole
flow — installing a matching `wasm-bindgen` CLI, injecting the `emcc`
settings, and laying out `pkg/` — via `wasm-pack new --emscripten` and
`wasm-pack build`. See its documentation for details.

## Tokio

> **Experimental.** This depends on Tokio's Emscripten event-loop support,
> which has not yet shipped in a Tokio release, and is subject to change.

An exported `async fn` marked `#[wasm_bindgen(experimental_tokio)]` is driven
as a root on a Tokio event-loop runtime instead of the `wasm-bindgen-futures`
executor, with its outcome bridged to the returned `Promise`. `tokio::spawn`,
timers and Tokio I/O then work inside the export without JSPI. All such
exports share the thread's ambient runtime; `experimental_tokio = "isolated"`
gives each invocation its own runtime, torn down once the root future
settles, for multiplexed hosts where one invocation's I/O must not cross
into another's context.

```rust
#[wasm_bindgen(experimental_tokio)]
pub async fn fetch(req: Request) -> Response {
    // tokio::net, tokio::time, tokio::spawn are all usable here
}
```

The support is unstable and gated behind `--cfg wasm_bindgen_unstable_tokio`.
Until the Tokio side lands upstream ([tokio#8484], with [mio#1969] for the
reactor), it also requires patching both crates to the PR branches and
Tokio's own `--cfg tokio_unstable`:

```toml
# Cargo.toml
[patch.crates-io]
mio = { git = "https://github.com/guybedford/mio", branch = "emscripten" }
tokio = { git = "https://github.com/guybedford/tokio", branch = "emscripten-event-loop-host" }
```

```toml
# .cargo/config.toml
[target.wasm32-unknown-emscripten]
rustflags = ["--cfg=wasm_bindgen_unstable_tokio", "--cfg=tokio_unstable", ...]
```

The cfg makes `wasm-bindgen-futures` depend on Tokio and expose the runtime
glue; without it nothing Tokio-related is compiled or linked, and the
attribute is a compile error (as it is on any other target).

[tokio#8484]: https://github.com/tokio-rs/tokio/pull/8484
[mio#1969]: https://github.com/tokio-rs/mio/pull/1969

## Limitations

- `-Cpanic=unwind` is not yet supported
  ([#5165](https://github.com/wasm-bindgen/wasm-bindgen/issues/5165)).
- `std::fs` requires Emscripten's real file system to be linked in; rustc's
  standalone link currently gets the stub implementations
  ([#5084](https://github.com/wasm-bindgen/wasm-bindgen/issues/5084)).
- `wasm-bindgen-test` supports the target via
  `wasm_bindgen_test_configure!(run_in_emscripten)`, but the harness
  currently only verifies the generated bindings; test bodies are compiled,
  not executed.
