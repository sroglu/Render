# Render.Utilities — Phase 1

Small, focused render-adjacent helpers that don't fit inside a single render pass and don't require URP. Five surfaces: procedural texture generation and readable copies (`TextureFactory`), a `Graphics.Blit`-based texture downscaler with a disposable handle (`TextureResizer` + `TextureResizeHandle`), material / camera chores (`RenderingTools`), a size-tracking render target (`AutoSizedRenderTexture`), and a set of `[Conditional]`-gated debug-draw helpers that strip in release builds (`RenderDebugTools`).

> **Status:** v0.1.0 — shipped 2026-05-15. This sub-module keeps no changelog of its own.

---

## Scope

**Utilities is for:**
- Building placeholder / procedural textures at runtime (solid fills, linear gradients, per-pixel tints) and taking a CPU-readable copy of a texture that may be compressed or GPU-only, optionally at a different size.
- Downscaling a `Texture2D` so neither dimension exceeds a clamp, with a uniform disposable-handle calling convention (`using var h = TextureResizer.Resize(src, max);`) regardless of whether the path produced a fresh texture or a pass-through.
- One-off material and camera chores that every project re-invents: switching a Standard-shader material into an alpha-blended mode, tinting it, restricting a camera to a sub-rect of the viewport, assigning shared materials.
- Holding a `RenderTexture` that follows a changing target size without reallocating when the size is unchanged.
- Drawing world-space debug primitives (lines, rays, boxes, arrows) from gameplay code that **cannot ship in release players** — call sites are stripped at compile time outside `UNITY_EDITOR` / `DEVELOPMENT_BUILD`.

**Utilities is NOT for:**
- URP / RenderGraph authoring — that's `PFound.Render.Core`'s job. Utilities is **independent of Core** by design (FR-003 in the Phase 1 spec; verified by the asmdef having only `Unity.Mathematics` as a reference — no URP, no Core).
- Replacement for Unity's native debug-draw — `RenderDebugTools` is a strip-safe wrapper, not a faster / fancier draw system. It composes on top of `UnityEngine.Debug.DrawLine` / `DrawRay`.
- Allocation-free downscaling — the blit path **must** allocate one `Texture2D` to hold the readback. Only the pass-through path avoids GPU work and allocation entirely.
- Cropping, padding, or upscaling — `TextureResizer` is downscale-only (max-dimension clamp; aspect ratio preserved). `TextureFactory.CreateReadableCopy` is the path for an explicit target size.
- GameSpecific assets — Phase 1 produces no runtime assets (Constitution III).

---

## Public Surface

| Type | Role |
|---|---|
| `TextureFactory` (`Runtime/`, `static`) | Procedural `Texture2D` generation and CPU-readable copies. `CreateSolidTexture(width, height, color)`, `CreateVerticalGradient(...)` / `CreateHorizontalGradient(...)` (both via the nested `GradientAxis` enum), `TintTexture(source, tint)`, and `CreateReadableCopy(Texture)` / `CreateReadableCopy(Texture, targetWidth, targetHeight)` — the latter blit through a temporary `RenderTexture`, so the source need not be readable. Everything it returns is `RGBA32`, no mipmaps. |
| `TextureResizer` (`Runtime/`, `static`) | `Resize(Texture2D source, int maxDimension) → TextureResizeHandle`. Aspect-preserving downscale clamp via `Graphics.Blit` + `Texture2D.ReadPixels`. Also exposes the pure `TryGetDownscaledSize(width, height, maxDimension, out targetWidth, out targetHeight)` used to compute that clamp. No exceptions: a `null` source or a `maxDimension ≤ 0` returns a non-owning pass-through handle, as does a source already within the bound. |
| `TextureResizeHandle` (`Runtime/`, `readonly struct`, `IDisposable`) | Uniform return type — owner-managed via `OwnsTexture` flag. Pass-through (no copy) returns `OwnsTexture = false` → `Dispose()` is a no-op. Downscale path returns `OwnsTexture = true` → `Dispose()` calls `Object.Destroy` (Play mode) or `Object.DestroyImmediate` (EditMode). Idempotent. `default(TextureResizeHandle)` is dispose-safe. |
| `RenderingTools` (`Runtime/`, `static`) | Runtime material and camera chores. `SetMaterialFade(material)` / `SetMaterialTransparent(material)` switch a Standard-shader material to straight-alpha or premultiplied blending (blend state, `_ZWrite`, keywords and `RenderQueue.Transparent` all set together); `TintMaterial(material, tint)` multiplies `_Color`; `SetCameraScissor(camera, viewportRect)` clamps a normalised rect and skews the projection matrix so the camera renders into that sub-region without shrinking the frustum; `SetSharedMaterials(renderer, params Material[])` assigns `sharedMaterials`. |
| `AutoSizedRenderTexture` (`Runtime/`, `sealed class`, `IDisposable`) | Owns one `RenderTexture` and keeps it matched to a requested size. `GetForSize(width, height)` reallocates only when the dimensions actually change (so steady-state calls are allocation-free), `GetForCamera(camera)` sizes to the camera's pixel dimensions, `Current` exposes the backing texture (null before the first request), `Dispose()` releases and destroys it. Depth bits and format are fixed at construction. |
| `RenderDebugTools` (`Runtime/`, `static`) | Strip-gated debug draw. Every public method carries `[Conditional("UNITY_EDITOR"), Conditional("DEVELOPMENT_BUILD")]` — call sites are removed at compile time in release player builds (SC-009). API: `DrawWorldLine`, `DrawWorldRay`, `DrawWorldBox`, `DrawWorldArrow`. |

---

## Architecture

### `TextureResizer.Resize` flow

```text
        ┌─ source.width / source.height ≤ maxDimension? ─┐
        │                                                │
       YES                                              NO
        │                                                │
        ▼                                                ▼
  new TextureResizeHandle             RenderTexture.GetTemporary(dstW, dstH, 0, ARGB32)
    (source, ownsTexture=false)         Graphics.Blit(source, rt)
    [pass-through — no GPU work]        RenderTexture.active = rt
                                        new Texture2D(dstW, dstH, RGBA32, false)
                                        ReadPixels(...) → Apply(no-mipmap, keep-readable)
                                        ReleaseTemporary(rt) + restore prev active
                                        new TextureResizeHandle(result, ownsTexture=true)
```

- **Aspect preserved**. The longer axis is anchored exactly on `maxDimension`; the other is scaled by the same factor, `Mathf.RoundToInt`-rounded and floored at `1` so neither collapses to zero. `TryGetDownscaledSize` is the public, GPU-free form of this computation and reports `false` when no downscale is needed.
- **Pass-through does no work at all** — no blit, no readback, no allocation. `TextureResizerBlitTests.PassThrough_WhenAlreadyWithinBound_DoesNoGpuWork` covers this; there is no Profiler-Recorder allocation assertion in this sub-module.
- **Downscale path allocates one `Texture2D`** — the unavoidable cost of returning a CPU-readable result, and called out under Limitations in `Render/MODULE.md` too.
- **No `RenderTexturePool` dependency.** Uses `RenderTexture.GetTemporary` directly so the asmdef stays independent of Core (FR-003).
- **Active-RT restore is guarded** — `prevActive` is captured before `Graphics.Blit` and restored in a `finally` block so an exception inside `ReadPixels` doesn't leak the active RT state.

### `TextureResizeHandle` disposal contract

The handle exists to give callers a **uniform calling convention**: always `using` (or otherwise `Dispose()`) the handle, regardless of whether the underlying `Texture` is shared with the caller or freshly-allocated.

- `OwnsTexture = false` (pass-through): `Dispose` is a no-op. Caller's original `Texture2D` is preserved untouched.
- `OwnsTexture = true` (downscale): `Dispose` destroys the texture via the Play/EditMode-aware path. Idempotent — calling `Dispose` twice (or on a `default` handle) is safe; the second call's `Texture == null` short-circuits the destroy.

### `RenderDebugTools` strip discipline

Every public method uses **two** `[Conditional]` attributes (`"UNITY_EDITOR"` + `"DEVELOPMENT_BUILD"`) — both gating the call from being emitted at the caller's site in release player builds. Inside the methods, calls forward to `UnityEngine.Debug.DrawLine` / `DrawRay`. The composite drawers (`DrawWorldBox`, `DrawWorldArrow`) construct their primitives entirely with `Vector3` math + `Quaternion` rotation — no GameObject allocation, no Gizmos dependency.

| Method | Signature | Behaviour |
|---|---|---|
| `DrawWorldLine` | `(Vector3 from, Vector3 to, Color color, float duration = 0f)` | Single `Debug.DrawLine`. |
| `DrawWorldRay` | `(Vector3 origin, Vector3 direction, Color color, float duration = 0f)` | Single `Debug.DrawRay`. |
| `DrawWorldBox` | `(Vector3 center, Vector3 size, Quaternion rotation, Color color, float duration = 0f)` | 12-edge wireframe using 8 corners precomputed from `center + rotation * (±h.x, ±h.y, ±h.z)` where `h = size * 0.5f`. 12 `DrawLine` calls. |
| `DrawWorldArrow` | `(Vector3 from, Vector3 to, Color color, float headSize = 0.25f, float duration = 0f)` | Body line + two head segments via `Quaternion.LookRotation(dir) * Quaternion.Euler(0, ±150f, 0) * Vector3.forward * headSize`. Early-return when `dir.magnitude < 1e-4f`. |

**Critical detail:** `[Conditional]` strips the **call site**, not the body. So a release build still contains the (now-unreachable) method bodies. The strip works only if call sites consume no side effects beyond the conditional methods — passing a pre-computed `Vector3` that would otherwise be unused is the standard pattern.

**Duration semantics** follow Unity's `Debug.DrawLine` contract — `0f` means "one frame", positive values persist on the Scene-view gizmo overlay for the given duration.

### Asmdef independence (FR-003)

The Phase 1 spec requires `PFound.Render.Utilities` to be consumable **without** pulling in `PFound.Render.Core`, URP, or RenderGraph dependencies. Verified by the asmdef:

```json
{
  "name": "PFound.Render.Utilities",
  "rootNamespace": "PFound.Render.Utilities",
  "references": [ "Unity.Mathematics" ],
  ...
}
```

Zero URP references, zero Core references. A consumer project can take only this asmdef (without `PFound.Render.Core`) and use every type in it standalone.

---

## Owner-Managed Conventions

`TextureResizer` returns an `IDisposable` handle — callers own it and **must** `Dispose` it. The owner-managed contract is the same shape as Phase 11 `BatchRendering` and Phase 10 `RenderContext`:

- `using var h = TextureResizer.Resize(src, 256);` is the canonical call.
- The library does **not** track handles internally — no leak detection, no auto-disposal.
- A leaked downscale handle leaks one `Texture2D` until the next domain reload.

`RenderDebugTools` has no allocations to manage; the strip-gated contract is the entire lifecycle.

---

## Tests

Assembly `PFound.Render.Utilities.Tests` (EditMode, `Utilities/Tests/EditMode/`):

- `TextureResizerMathTests` — `TryGetDownscaledSize` clamp math (landscape / portrait / square / extreme aspect, short axis never below one pixel, aspect preserved within a pixel); `null` source and non-positive `maxDimension` both returning a non-owning pass-through; within-bound returning the same instance; non-owning `Dispose` leaving the texture alive; owning `Dispose` destroying it and being idempotent.
- `TextureFactoryTests` — solid fill size + per-pixel colour, vertical gradient running bottom-to-top, horizontal gradient running left-to-right, `TintTexture` multiplying pixels.
- `RenderingToolsTests` — `SetMaterialFade` blend state + transparent queue, `SetMaterialTransparent` premultiplied blend, `TintMaterial` multiplying the main colour, `SetCameraScissor` rect clamping + projection skew, `SetSharedMaterials` assigning the array.
- `RenderDebugToolsSmokeTests` — `DrawWorldLine` / `DrawWorldBox` / `DrawWorldRay` / `DrawWorldArrow` smoke (no-throw). Visual rendering verification is documented as manual per SC-009 — the automated suite covers strip-safety + no-throw only.

Assembly `PFound.Render.Utilities.Tests.PlayMode` (PlayMode, `Utilities/Tests/PlayMode/`) — these need a live graphics device:

- `TextureResizerBlitTests` — downscale produces an owned texture at the clamped size, preserves a solid colour through the blit, restores the previously active render target, destroys the owned texture on `Dispose`, and does no GPU work on the pass-through path.
- `ReadableCopyTests` — `TextureFactory.CreateReadableCopy` preserving size + content and rescaling to a target size; `AutoSizedRenderTexture` reusing its texture when the size is unchanged, reallocating when it changes, and releasing on `Dispose`.

The EditMode suites use the namespaces `PFound.Render.Tests` and `PFound.Render.Utilities.Tests`; both compile into the same assembly.

---

## Known Gaps / Deferred

- **Downscale path allocates one `Texture2D` per call** — documented as unavoidable. Callers needing per-frame downscaling should pool the result themselves or switch to a `RenderTexture`-only flow.
- **No upscale path.** Pass-through is the only behaviour for `max(W, H) ≤ maxDimension` sources. Upscaling intentionally out of scope (better handled by sampler `filterMode` than CPU readback).
- **Downscale result is always `RGBA32`.** `TextureResizer` reads back into a fresh `RGBA32`, no-mipmap texture regardless of the source format. If you need a different format, convert separately.
- **No cropping / padding / aspect-correction.** Aspect is always preserved.
- **No `RenderTexture` overload.** Only `Texture2D → Texture2D`. A `RenderTexture` overload could land if a concrete consumer needs it.
- **`RenderDebugTools` is line-art only.** No filled-volume primitives (no shaded sphere, no filled box). Gizmos / custom mesh remain the path for filled debug visualizations.
- **`RenderDebugTools` body code is NOT stripped, only call sites.** Release builds contain unreachable bodies (acceptable cost — bodies are short and unreferenced). If body-strip ever becomes important, wrap the entire class in `#if UNITY_EDITOR || DEVELOPMENT_BUILD`.
- **Main-thread only.** Every surface here assumes the Unity main thread. No thread-safety guarantees.
- **No editor companion.** This sub-module ships no Editor assembly — Phase 1 didn't need one. Future helpers requiring editor-only surface should follow the established layout: an `Editor/` subfolder inside the sub-module with the Editor asmdef at its root (see `Core` and `RenderContext`).

---

## Dependencies

- `Unity.Mathematics` (asmdef reference) — pulled in as a low-cost helper but **not currently used** in the public surface (`Vector3` / `Mathf` from `UnityEngine` cover the Phase 1 needs). Reserved for future Burst-friendly variants.
- **Intentionally NOT** `PFound.Render.Core` — independence is a hard contract (FR-003).
- **Intentionally NOT** URP packages — the asmdef has zero URP refs; verified by `Utilities/PFound.Render.Utilities.asmdef`.

---

## Related

- `Render/MODULE.md` — top-level Render submodule index + phase roadmap. Limitations section calls out the `TextureResizer` blit-path allocation explicitly.
- `Render/Core/MODULE.md` — sibling Core asmdef (`RenderFeatureBase`, `RenderTexturePool`, `GlobalShaderParameterManager`, shared HLSL).
- `Render/Effects.Overdraw/MODULE.md` — the other strip-gated developer-only surface in Render.
