# Proposal: allow a Vulkan-swapchain backend on a device with no DRM nodes

**Status: experimental candidate — NOT compiled, NOT runtime-tested, must not be shipped as a fix.**
This repository exists to share a small candidate change with gamescope maintainers for review and
guidance. It is not a pull request and it does not claim that gamescope works on the affected device.

## Target

- Upstream base: `ValveSoftware/gamescope` **tag `3.16.29`**
  (commit `8f212644c46460549035971ab914421180f61a7c`).
- Only source file changed: **`src/rendervulkan.cpp`** (`CVulkanDevice::createDevice()`).
- The candidate diff is also kept verbatim in `docs/p10-panvk-no-drm-swapchain.patch` so it can be
  inspected independently of the source tree.

## Background

Some Vulkan drivers advertise `VK_EXT_physical_device_drm` but expose **no DRM node at all**. The
case at hand is **Mesa PanVK** running over the ARM **kbase/CSF** kernel interface on an Android
device: the host Linux environment ships no `/dev/dri`, yet Vulkan works and presentation through the
Wayland WSI is fine for a plain client.

`CVulkanDevice::createDevice()` currently requires:

```cpp
if ( !GetBackend()->UsesVulkanSwapchain() && !drmProps.hasPrimary ) { … "physical device has no primary node" … return false; }
if ( !drmProps.hasRender )                                          { … "physical device has no render node"  … return false; }
```

With `hasPrimary == false` and `hasRender == false`, gamescope therefore refuses the device for every
backend:
- the nested **Wayland** backend (`UsesVulkanSwapchain() == false`) fails on the primary node;
- the **SDL** backend (`UsesVulkanSwapchain() == true`) skips the primary check but still fails on the
  render node.

## What the candidate does

`docs/p10-panvk-no-drm-swapchain.patch` changes **only** the no-node case and **only** for a
Vulkan-swapchain backend:

```cpp
const bool bSwapchainBackend = GetBackend()->UsesVulkanSwapchain();
const bool bHasAnyDrmNode    = drmProps.hasPrimary || drmProps.hasRender;

if ( !bHasAnyDrmNode && bSwapchainBackend ) {
    vk_log.warnf( "physical device reports no DRM nodes; swapchain backend will run without DRM node access" );
} else {
    if ( !bSwapchainBackend && !drmProps.hasPrimary ) { … unchanged failure … }
    if ( !drmProps.hasRender )                        { … unchanged failure … }
}
if ( bHasAnyDrmNode ) { … unchanged render-node open / primary dev id … }
```

Invariant: *the DRM-node requirements change only when the device exposes no node and only for a
Vulkan-swapchain backend.*

### Existing checks that remain intact
- Any device that exposes a primary or render node takes the **original** path unchanged (checks,
  render-node open, primary device id) — this includes every existing DRM/KMS-backed GPU.
- Every **non-swapchain** backend (Wayland, DRM, Headless, OpenVR) on a node-less device still fails
  with the original `has no primary node` / `has no render node` messages.
- No fake DRM nodes are created; DRM/KMS paths are not weakened; PanVK is not modified; the Android
  compositor is not touched.

## Known unresolved issue (why this is only a candidate)

On the swapchain (SDL) path, the only remaining consumer of a DRM node is the renderer's
`wlr_renderer_impl.get_drm_fd`, which this change would leave returning **`-1`**. wlroots consumes it
when creating the linux-dmabuf feedback global (`wlr_renderer_init_wl_display()`) and the `wlr_drm`
global (`wlr_drm_create()`). Whether the wlroots revision used by gamescope `3.16.29` (submodule
`d783533489e1f75d6886c2ab5c5960090ef268f8`) safely tolerates `-1` there is **not yet verified**.

The other DRM-node consumers in gamescope are unaffected:
- `vulkan_primary_dev_id()` is used only by the DRM/KMS backend (`Backends/DRMBackend.cpp`,
  `init_drm()`), which this change does not enable on a node-less device.
- DMA-BUF import/export and explicit sync use client-provided file descriptors and Vulkan external
  memory; they do not need gamescope's own node.

## Validation performed

- The candidate applies cleanly to a pristine checkout of tag `3.16.29` (`git apply --check`).
- The resulting change is 39 insertions / 24 deletions in `src/rendervulkan.cpp`; no other file is
  modified; `git diff --check` is clean.

Not performed: compilation, linking, or any runtime/device test. No gamescope build environment was
available, and the change has not been shown to produce a working session.

## Ask

- Is a Vulkan-swapchain backend on a device that advertises `VK_EXT_physical_device_drm` with no
  nodes an intended use case?
- Would a narrowly-scoped change of this shape (relaxing the node requirement only for a swapchain
  backend when the device exposes no node) be acceptable, or is a different/architectural change
  (gamescope and/or wlroots) required?
- What is the expected behaviour of `wlr_renderer_init_wl_display()` / `wlr_drm_create()` when the
  renderer's DRM fd is `-1`, and is the linux-dmabuf feedback global required at all for a nested
  session whose clients are Xwayland/X11?
