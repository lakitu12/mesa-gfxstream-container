# mesa-gfxstream-container

CI for building **Mesa's gfxstream Vulkan ICD with the kumquat transport** (glibc arm64),
for use inside DroidSpaces debian containers on Mali/MTK Android hosts.

## Why

Container gfxstream guest ICDs shipped by Debian (`libvulkan_gfxstream.so`) are built
**without** `virtgpu_kumquat`, so the kumquat platform is a stub — the ICD can only talk
virtio-gpu DRM, which does not exist inside containers. With `VIRTGPU_KUMQUAT=1` and a
real kumquat-enabled ICD, the driver connects to a `kumquat_virtio` server over a unix
socket (`/tmp/kumquat-gpu-0`) instead of DRM.

## Stack

```
container (glibc arm64)                          host (Android / same container for PoC)
MobileGL DirectVulkan
  -> libvulkan_gfxstream.so (this CI, kumquat)
    -> unix socket /tmp/kumquat-gpu-0
      -> kumquat_virtio (rutabaga_gfx kumquat/server, rust)
        -> libgfxstream_backend.so (gfxstream host)
          -> VK_ICD_FILENAMES backend ICD  (PoC: lavapipe | hardware: vendor Mali bionic)
```

- kumquat server pinned the same as gfxstream:
  https://github.com/magma-gpu/rutabaga_gfx `13cde31787f65f1b5521b2a7ab54f56099cdc09b`
  (+ gfxstream's `third_party/rutabaga/PATCH.rutabaga.patch`)
- build the server: `cargo build --release -p kumquat_virtio --features gfxstream`
  with `GFXSTREAM_PATH=<dir containing libgfxstream_backend.so>`

## Usage (container)

```bash
tar -xzf gfxstream_icd_*.tar.gz -C /            # /usr/lib/<triplet>/libvulkan_gfxstream.so + icd.d json
VIRTGPU_KUMQUAT=1 VK_ICD_FILENAMES=/usr/share/vulkan/icd.d/gfxstream_vk_icd.json vulkaninfo --summary
# deviceName flips llvmpipe -> gfxstream
```

## Workflow

`.github/workflows/build.yml` — manual dispatch (`tag`, default `mesa-26.1.5`, `draft_release`).
Based on lakitu12/mesa build.yml (ubuntu-24.04-arm native arm64, debian:testing-slim,
apt build-dep, ccache). Mesa is cloned from upstream gitlab at build time; this repo
carries only the workflow. ICD-only meson config (`-Dvulkan-drivers=gfxstream
-Dvirtgpu_kumquat=true`, everything else off) + rustup/bindgen-cli for the kumquat
rust/bindgen requirements.
