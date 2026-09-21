# Vulkan CTS Snap

This snap provides an easy way to install and run the tests found in
[Khronos's Vulkan Conformance Test Suite](https://github.com/KhronosGroup/VK-GL-CTS)

## Snap bases

The snap is maintained for multiple bases, each in its own self-contained
snapcraft project directory, and published as its own separate Snap Store
package (not tracks of one shared package):

| Directory | Base   | Snap Store name          | GPU content       | Arches       | Notes                                        |
|-----------|--------|---------------------------|-------------------|--------------|-----------------------------------------------|
| `core22/` | core22 | `baconyao-vulkan-cts-22` | `graphics-core22` | amd64, arm64 | Toolchain and drivers from the 22.04 archive  |
| `core24/` | core24 | `baconyao-vulkan-cts-24` | `gpu-2404`        | amd64, arm64 | Toolchain and drivers from the 24.04 archive  |
| `core26/` | core26 | `baconyao-vulkan-cts-26` | `gpu-2604`        | amd64, arm64 | Toolchain and drivers from the 26.04 archive  |

Each project cross-compiles for `arm64` from an `amd64` build host (as well as
building natively on an `arm64` host), so a single `amd64` builder can produce
both architectures' snaps.

Newer hardware needs newer userspace drivers. If a test fails at startup with
`VK_ERROR_INCOMPATIBLE_DRIVER` or no devices are found, the base you installed
likely predates your GPU; use a newer base.

Each variant publishes to the `edge` channel of its own Snap Store package.

## Build

Each directory is a directly-buildable snapcraft project. `cd` into the base
you want and run snapcraft:

```
cd core22 && snapcraft pack
cd core24 && snapcraft pack
cd core26 && snapcraft pack
```

Each project declares `platforms: amd64, arm64`; pass `--build-for=arm64` (or
build on/for an arm64 host) to produce the arm64 snap, or use
`snapcraft remote-build` to build all platforms via Launchpad.

## Install

```
snap install --dangerous baconyao-vulkan-cts-<22|24|26>_<version>_<your_arch>.snap
```

Or from the store, choosing the package that matches your base:

```
snap install baconyao-vulkan-cts-22 --channel=edge
snap install baconyao-vulkan-cts-24 --channel=edge
snap install baconyao-vulkan-cts-26 --channel=edge
```

The GPU content interface auto-connects for store installs. For a sideloaded
(`--dangerous`) install, connect it manually to match the base:

```
snap connect baconyao-vulkan-cts-22:graphics-core22 mesa-core22:graphics-core22   # core22
snap connect baconyao-vulkan-cts-24:gpu-2404 mesa-2404:gpu-2404                   # core24
snap connect baconyao-vulkan-cts-26:gpu-2604 mesa-2604:gpu-2604                   # core26
```

## Run

### List available test caselists
```
baconyao-vulkan-cts-<22|24|26>.list-tests
```

### Run tests

Run a specific test case:
```
baconyao-vulkan-cts-<22|24|26>.test dEQP-VK.info.build
```

Or run tests from a mustpass caselist:
```
baconyao-vulkan-cts-<22|24|26>.test --caselist=mustpass/main/vk-default/api.txt
```

Use `baconyao-vulkan-cts-<22|24|26>.vulkaninfo` to inspect the Vulkan driver
stack that will be used, and
`baconyao-vulkan-cts-<22|24|26>.test --no-confinement` to bypass the GPU
content interface and use the host's Vulkan loader/drivers instead.
