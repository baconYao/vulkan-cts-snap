# Vulkan CTS Snap

This snap provides an easy way to install and run the tests found in
[Khronos's Vulkan Conformance Test Suite](https://github.com/KhronosGroup/VK-GL-CTS)

## Snap bases

The snap is maintained for multiple bases, each in its own self-contained
snapcraft project directory:

| Directory | Base   | GPU content       | Arches       | Notes                                     |
|-----------|--------|-------------------|--------------|--------------------------------------------|
| `core22/` | core22 | `graphics-core22` | amd64, arm64 | Toolchain and drivers from the 22.04 archive |
| `core24/` | core24 | `gpu-2404`        | amd64, arm64 | Toolchain and drivers from the 24.04 archive |

Each project cross-compiles for `arm64` from an `amd64` build host (as well as
building natively on an `arm64` host), so a single `amd64` builder can produce
both architecture's snaps.

In the Snap Store the variants are published on separate tracks
(`latest`/default for core24, `core22` for core22).

## Build

Each directory is a directly-buildable snapcraft project. `cd` into the base
you want and run snapcraft:

```
cd core22 && snapcraft pack
cd core24 && snapcraft pack
```

Each project declares `platforms: amd64, arm64`; pass `--build-for=arm64` (or
build on/for an arm64 host) to produce the arm64 snap.

## Install

```
snap install --dangerous vulkan-cts_<version>_<your_arch>.snap
```

Or from the store, choosing the channel that matches your base:

```
snap install vulkan-cts                       # default (core24) track
snap install vulkan-cts --channel=core22/edge # core22 track
```

The GPU content interface auto-connects for store installs. For a sideloaded
(`--dangerous`) install, connect it manually to match the base:

```
snap connect vulkan-cts:graphics-core22 mesa-core22:graphics-core22   # core22
snap connect vulkan-cts:gpu-2404 mesa-2404:gpu-2404                   # core24
```

## Run

### List available test caselists
```
vulkan-cts.list-tests
```

### Run tests

Run a specific test case:
```
vulkan-cts.test dEQP-VK.info.build
```

Or run tests from a mustpass caselist:
```
vulkan-cts.test --caselist=mustpass/main/vk-default/api.txt
```

Use `vulkan-cts.vulkaninfo` to inspect the Vulkan driver stack that will be
used, and `vulkan-cts.test --no-confinement` to bypass the GPU content
interface and use the host's Vulkan loader/drivers instead.
