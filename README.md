# primase-naga

[![crates.io: primase-naga](https://img.shields.io/crates/v/primase-naga?label=primase-naga)](https://crates.io/crates/primase-naga) [![docs.rs](https://img.shields.io/docsrs/primase-naga)](https://docs.rs/primase-naga) [![CI](https://img.shields.io/github/actions/workflow/status/Kalmarv/primase-naga/ci.yml?branch=main&label=CI)](https://github.com/Kalmarv/primase-naga/actions/workflows/ci.yml)

A fork of [naga](https://github.com/gfx-rs/wgpu/tree/trunk/naga) 30.0.1, the shader translator and validator from the wgpu project, with three additions. It is used by [iridia](https://github.com/Kalmarv/iridia), which lifts compiled Direct3D shaders into naga modules that need these additions.

## Features

- Writes ray queries in the WGSL backend (`rayQueryInitialize`, `rayQueryProceed`, the intersection and hit-vertex-position getters) and enables `wgpu_ray_query` and `wgpu_ray_query_vertex_return` as needed
- Adds an optional `result` to `Statement::ImageAtomic` that holds the texel's previous value, written by the SPIR-V, HLSL, MSL and GLSL backends
- Accepts `AtomicFunction::Exchange` in `ImageAtomic` on 32-bit integer images, written by the SPIR-V, HLSL and GLSL backends
- Otherwise identical to naga 30.0.1 as released in gfx-rs/wgpu v30.0.1
- Reads WGSL, SPIR-V and GLSL (Vulkan 440 to 460)
- Writes WGSL, SPIR-V, HLSL, MSL, GLSL (core 330+ and ES 300+) and Graphviz DOT
- Validates modules against a configurable set of capabilities
- Keeps the library name `naga`, so existing code needs no changes
- Returns an unsupported-feature error from backends that cannot express an addition (an image atomic result in WGSL, image exchange in WGSL and MSL)
- Builds as `no_std` with `alloc` and forbids unsafe code

## Getting Started

### Requirements

Rust 1.87 or newer.

### Installation

primase-naga is published on crates.io. Rename it to `naga` in `Cargo.toml` and enable the front and back ends you need. No features are enabled by default.

```toml
[dependencies]
naga = { package = "primase-naga", version = "30.0.1", features = ["wgsl-in", "wgsl-out"] }
```

The front-end features are `wgsl-in`, `spv-in` and `glsl-in`. The back-end features are `wgsl-out`, `spv-out`, `hlsl-out`, `msl-out`, `glsl-out` and `dot-out`. `hlsl-out-if-target-windows` and `msl-out-if-target-apple` enable those back ends only on the matching platform.

### Usage

This parses a WGSL compute shader that uses a ray query, validates it and writes it back out as WGSL. Stock naga 30.0.1 panics in its WGSL backend on this module.

```rust
use naga::back::wgsl::WriterFlags;
use naga::valid::{Capabilities, ValidationFlags, Validator};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let source = r#"
enable wgpu_ray_query;

@group(0) @binding(0) var acc: acceleration_structure;

@compute @workgroup_size(1)
fn main() {
    var rq: ray_query;
    rayQueryInitialize(&rq, acc, RayDesc(0u, 255u, 0.1, 10.0, vec3<f32>(0.0), vec3<f32>(0.0, 1.0, 0.0)));
}
"#;

    let module = naga::front::wgsl::parse_str(source)?;
    let info = Validator::new(ValidationFlags::all(), Capabilities::RAY_QUERY).validate(&module)?;
    let wgsl = naga::back::wgsl::write_string(&module, &info, WriterFlags::empty())?;

    // Starts with `enable wgpu_ray_query;` and calls `rayQueryInitialize((&rq), acc, RayDesc(...))`.
    println!("{wgsl}");
    Ok(())
}
```

Everything else follows naga 30.0.1's API, which is documented at [docs.rs/naga/30.0.1](https://docs.rs/naga/30.0.1).

Run the library's unit tests with `cargo test --lib --all-features`. Upstream's snapshot and integration tests depend on the wgpu workspace and are not included. Changes from naga are listed in [CHANGELOG.md](CHANGELOG.md). Licensed under MIT or Apache-2.0, as naga is ([LICENSE.MIT](LICENSE.MIT), [LICENSE.APACHE](LICENSE.APACHE)).

## Releasing

A release is a commit that changes a version. Once every CI job has passed on main, CI's release job
(`.github/release.py`) publishes each crate whose version crates.io does not have yet, in dependency order (trusted
publishing: no token is stored anywhere), tags the commit `v<version>` (`<package>-v<version>` for a package with a
version of its own) and creates the GitHub Release with that version's CHANGELOG section as its notes.
A commit that changes no version only runs CI.

1. Bump the version when a change is worth releasing: a patch (0.x.Y) for fixes and data or manifest additions, a minor
   (0.X.0) for new API or any breaking change before 1.0. Run `cargo check` so `Cargo.lock` follows.
2. In the same commit, move the change's CHANGELOG lines under a `## <version> (<date>)` heading.
3. Push to main. Never publish by hand; a new crate's first release is the one exception (the token holder makes it,
   then the crate gets its trusted-publishing config).
