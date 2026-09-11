# PFound.Render

Rendering building blocks for Unity URP. Ten independent sub-modules, each its own assembly — take
only what you need.

## Sub-modules

| Sub-module | Assembly | What it is |
|---|---|---|
| [Core](Core/MODULE.md) | `PFound.Render.Core` | Render-texture pool, global shader parameter manager, URP RenderGraph feature/pass base classes, shared HLSL includes. |
| [BatchRendering](BatchRendering/MODULE.md) | `PFound.Render.BatchRendering` | Burst frustum/distance-culled GPU instancing service (classic / indirect / procedural backends). |
| [RenderContext](RenderContext/MODULE.md) | `PFound.Render.RenderContext` | Off-screen camera → `RenderTexture` bound to a `RawImage` / `MeshRenderer` / UI Toolkit element. |
| [Utilities](Utilities/MODULE.md) | `PFound.Render.Utilities` | Texture creation, GPU resize/blit, readable copies, material/camera helpers, strip-gated render debug helpers. |
| [Effects.Blur](Effects.Blur/MODULE.md) | `PFound.Render.Effects.Blur` | Volume-driven full-screen Gaussian blur + priority-queue request service. |
| [Effects.Outline](Effects.Outline/MODULE.md) | `PFound.Render.Effects.Outline` | Volume-driven full-screen depth-edge outline + priority-queue request service. |
| [Effects.Overdraw](Effects.Overdraw/MODULE.md) | `PFound.Render.Effects.Overdraw` | Developer-only overdraw heatmap debug view (strips from release builds). |
| [PostProcess](PostProcess/MODULE.md) | `PFound.Render.PostProcess` | Typed post-process request/adapter stack (built-in Blur + Outline adapters). |
| [ShaderWarmup](ShaderWarmup/MODULE.md) | `PFound.Render.ShaderWarmup` | Time-sliced boot shader variant pre-warm controller. |
| [UIShapes](UIShapes/MODULE.md) | `PFound.Render.UIShapes` | SDF UI shape shader + size-sync component + editor bake tooling. |

Core, RenderContext and UIShapes each ship an Editor companion assembly alongside their runtime one,
so the module is 13 non-test assemblies in total. A `Shaders/` folder at the module root carries the
authored SoftToony URP shader set; it has no assembly of its own.

## Docs

Deep reference: **[MODULE.md](MODULE.md)** — subsystem scope, sub-module map, asmdef dependency model,
GameSpecific hook, per-sub-module setup/wiring, and verification. Per-sub-module depth lives in that
sub-module's own `MODULE.md`, linked from the table above.

## Dependencies

Utilities is fully independent (no URP, no Core); Core needs only URP + Collections/Burst/Mathematics.
Every other sub-module builds on Core. BatchRendering, RenderContext, PostProcess and ShaderWarmup
also take `PFound.LoopScheduler` for their per-frame tick; RenderContext, PostProcess and ShaderWarmup
take `PFound.DependencyContainer` for their optional registration helpers — see
[MODULE.md](MODULE.md) for the full per-assembly table.
