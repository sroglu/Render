# Render.RenderContext — Changelog

All notable changes to the RenderContext submodule.

## [1.0.0] — 2026-09-11

First complete release. The submodule now ships the whole render-to-texture context stack;
the previous entry only covered the empty folder scaffolding.

### Added

- **Service** — `IRenderContextService` / `RenderContextService`. `Acquire(descriptor, anchor)`
  returns an `IRenderContextHandle` carrying the leased `Texture`, the dedicated `Camera`, a
  `ContentRoot` `Transform`, `IsAlive` and `Refresh()`. The service owns a hidden
  `HideAndDontSave` GameObject (`DontDestroyOnLoad` in PlayMode only) parked far from the origin
  so context content stays out of the consumer's main camera.
- **Descriptor** — `RenderContextDescriptor`, an `IEquatable` value struct describing the render
  target and camera essentials (size, format, depth bits, MSAA, clear flags, background colour,
  field of view, culling mask, orthographic settings).
- **Pool** — `RenderContextPool` + `RenderContextPoolKey` + `PooledEntry`. Six descriptor fields
  key an entry; the render-texture lifecycle itself is delegated to `PFound.Render.Core`'s
  `RenderTexturePool`, so idle eviction and leak detection come from Core rather than being
  duplicated here. Camera and `ContentRoot` are kept as a local sidecar that survives an RT
  eviction. Per-lease camera state is deliberately excluded from the key and re-applied on
  every lease.
- **Scene factory** — `RenderContextSceneFactory` builds the per-acquisition hierarchy
  (`BuildHierarchy`), re-applies camera state (`ResetCamera`) and tears content down
  (`DestroyChildren`, which detaches before destroying so `childCount` drops immediately).
- **Anchors** — `IRenderContextAnchor`, `IExplicitSizeAnchor`, and three built-ins:
  `RawImageAnchor` (uGUI), `VisualElementAnchor` (UI Toolkit), `MeshRendererAnchor` (world-space).
- **Sinks** — `IRenderContextSink` and the matching `RawImageSink`, `VisualElementSink` and
  `MeshRendererSink`. Each captures the target's pre-bind state on `Bind` and restores it on
  `Unbind`. `VisualElementSink` binds through a stretch-to-fill `Image` child rather than
  `style.backgroundImage`, which Unity 6 normalises away for `RenderTexture` values.
- **Resize watcher** — `AnchorResizeWatcher`, ticked on `PFound.LoopScheduler`'s BeforeRender
  loop, calls `Refresh()` on a handle when its anchor's preferred size changes; the stable,
  changed, zero-size and destroyed-target cases are all covered.
- **Resolution strategies** — `RenderContextResolver` (`Use`, `Clear`, `IsConfigured`, plus an
  `internal Resolve()` used by the sink component) over `IRenderContextServiceProvider`, with
  `SingletonRenderContextServiceProvider`, `ContainerRenderContextServiceProvider` and
  `DelegateRenderContextServiceProvider`. `RenderContextRegistration.Register(container)` is the
  `PFound.DependencyContainer` bootstrap convenience; the container is never required.
- **Inspector authoring path** — `RenderContextSinkBehaviour`, an opt-in MonoBehaviour that
  acquires on `OnEnable` and disposes on `OnDisable`, resolving its service through the resolver.
- **Editor companion** — `PFound.Render.RenderContext.Editor` with
  `RenderContextSinkBehaviourEditor`.
- **Diagnostics** — one-shot-per-service warnings for `Msaa > 2`, a `CullingMask` of "Everything",
  and camera shadows left enabled on a context camera.
- **Tests** — `PFound.Render.RenderContext.Tests` (EditMode) covers the descriptor, pool key,
  pool, service lifecycle, all three anchor/sink pairs, the resize watcher, headless rendering,
  idempotent handle disposal, steady-state zero allocation and the diagnostics.
  `PFound.Render.RenderContext.Tests.PlayMode` covers the uGUI, UI Toolkit and world-space
  smoke paths, the MonoBehaviour wrapper parity path, and cross-backend render orthogonality.

### Notes

- Render-texture only: URP's `Camera.cameraStack` screen-blit stacking is out of scope, as are
  session-lifetime persistent render targets (those stay client-managed).
- Owner-managed lifecycle throughout — whoever acquires a handle disposes it.

## Phase 1 — Setup

- Folder layout, runtime + editor asmdefs, MODULE.md skeleton, this changelog. No release date
  was recorded for this entry.
