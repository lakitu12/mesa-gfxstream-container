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
apt build-dep, ccache). Mesa is cloned from upstream gitlab at build time;
`patches/` are applied after clone. ICD-only meson config
(`-Dvulkan-drivers=gfxstream -Dvirtgpu_kumquat=true -Dplatforms=x11,wayland`,
everything else off) + rustup/bindgen-cli for the kumquat rust/bindgen requirements.

## Patch

`patches/0001-gfxstream-guest-add-xlib-and-headless-surface-exts.patch` — adds
`VK_KHR_xlib_surface` + `VK_EXT_headless_surface` to the guest ICD instance-extension
whitelist (`kGuestEmulatedInstanceExtensions[]` in
`src/gfxstream/guest/vulkan/gfxstream_vk_device.cpp`). Upstream whitelist only carries
xcb/wayland; MobileGL DirectVulkan needs xlib or headless. Not upstreamed — re-check on
tag bumps (`git apply --check` in CI guards this).

## Platform matrix / caveats

- `x11,wayland` (current): the ICD links wayland WSI and needs `wl_fixes_interface`
  (libwayland ≥ forky). On trixie containers the loader rejects it
  (`undefined symbol: wl_fixes_interface`). Kept for anland.
- `x11` only: works on trixie — that build validated the full MobileGL loopback
  (`GL_RENDERER = Magma ... (Virtio-GPU GFXStream ...)`).
- Swapchain export (`vkGetMemoryFdKHR`) requires an exportable host memory backend;
  lavapipe loopback can't provide it (expected). On the Mali/Android host side
  gfxstream auto-enables `VulkanAllocateHostVisibleAsUdmabuf` on 6.6 kernels.
