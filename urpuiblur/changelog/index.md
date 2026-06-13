# Changelog

All notable changes to URP UI Blur will be documented in this file.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).

## [1.0.0] - 2026-05-01

### Added
- Six blur algorithms: Dual Kawase, Gaussian Separable, Tent/Hex, Poisson Disk,
  Custom Kawase, Radial — all implemented with URP 17+ native RenderGraph API
- `ScriptableRendererFeature` + `ScriptableRenderPass` architecture supporting
  multiple simultaneous presets, each independently configured via a `BlurPreset`
  ScriptableObject asset
- Five canvas space targets: Screen Space Overlay (before/after post-processing),
  Screen Space Camera, World Space, World Objects — each writing to a distinct
  global texture sampled by the corresponding material
- Persistent RTHandle strategy — each preset owns its intermediate textures
  (allocated via `RTHandles.Alloc()`), preventing cross-preset memory aliasing by
  the RenderGraph allocator
- `UI_Blur_Universal` shader for UI Image / Canvas panels — frosted glass tint,
  brightness, desaturation, vignette edge darkening, and normal-map glass distortion;
  three normal mapping modes: Standard Tile, Local Fixed Scale, Screen Space Fix
- `WorldObject` shader for 3D world-space glass meshes — true IOR refraction via
  `refract()` with normal map support; samples `_BlurredBackground_WorldObjects`
- Resolution-independent blur scaling — continuous parameters (`blurOffset`,
  `gaussianRadius`, `radialRadius`) scale with screen width relative to 1920 px
  baseline; discrete parameters (`iterations`, `taps`, `radialTaps`) do not scale
- Auto-culling for world-space presets via `cullResults.visibleRenderers` —
  UI presets intentionally bypass culling to prevent one-frame pop-in
- Duplicate spaceType detection: Inspector warning in `BlurEditor` and runtime
  guard in `AddRenderPasses` (first preset wins, duplicates are skipped)
- `SetPresetsActive(CanvasSpaceType, bool)` static API for script-driven
  enable/disable of preset groups at runtime
- `BlurSetupWizard` Editor window — auto-opens on first import; scans all
  `UniversalRendererData` assets in the project, adds the Blur Renderer Feature
  via `SerializedObject` to keep `m_RendererFeatureMap` in sync, and auto-assigns
  the bundled `UI-Overlay.asset` default preset
- `BlurPreset` ScriptableObject — per-method field visibility, algorithm info box
  with performance cost indicator (dot scale), and renderer user list in Inspector
- Custom Inspectors: `BlurEditor` (Renderer Feature), `BlurScriptable` (BlurPreset)
- Included materials: `UI_Blur_Universal.mat` (UI panels), `Blur_WorldSurface.mat`
  (3D world meshes) — pre-configured, no wizard generation needed
- Interactive Demo Scene (Samples~) comparing all six blur methods with live
  runtime parameter controls

### Fixed
- Horizontal stretch in Gaussian and Tent/Hex passes — replaced
  `_BlitTexture_TexelSize` (source size) with `_TargetTexelSize` (destination size)
  passed from C#; ensures both separable passes use the same texel step regardless
  of source/destination resolution mismatch
- Blur darkening at high iterations in DualKawase, TentHex, Gaussian — caused by
  dividing by theoretical tap count when some taps were skipped by the `abs(i) >
  radius` guard; fixed by dividing by the actual sampled count
- Screen UV precision at 4K — `ComputeScreenPos` + perspective divide in
  `UI_Blur_Universal.shader` replaces the previous `_ScreenParams`-based UV
  calculation that drifted at high resolutions
- Cross-preset texture contamination — graph-owned transient textures with matching
  descriptors were being memory-aliased by the RenderGraph allocator across presets;
  fixed by switching to externally-owned persistent RTHandles which the allocator
  cannot alias
- Stale `_propReferenceHeight` serialized property reference in `BlurScriptable.cs`
- Pass name collisions across presets sharing the same algorithm — all pass names
  now include `_texPrefix` (slot index + output texture name)
