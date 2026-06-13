# URP UI Blur Pro

Real-time frosted glass and background blur for Unity 6 (URP 17+), built natively on the RenderGraph API.
Create modern blurred UI panels, world-space interfaces, and dynamic glass surfaces without camera stacking, GrabPass, or custom render textures.
Supports six blur algorithms, multiple simultaneous presets, and runtime control.
Each of the six blur algorithms writes a global _BlurredBackground_* texture that is sampled directly by the included frosted-glass shaders.

---

## Feature Highlights

- 6 real-time blur algorithms
- Multiple simultaneous presets
- Screen Space Overlay / Camera / World Space support
- Dynamic 3D glass refraction
- Native RenderGraph implementation
- Setup Wizard for one-click installation
- Runtime API support
- Resolution-independent blur scaling

---

## Requirements

- Unity 6000.0 or later
- Universal Render Pipeline 17.0.0 or later
- RenderGraph enabled (Project Settings → Graphics → URP Global Settings)

---

## Quick Start

1. Open the **Setup Wizard** via `Tools → URPUIBlur → Setup Wizard`
   (it also opens automatically the first time the package is imported)
2. Tick the URP Renderer assets you want to add the feature to and click **Add Blur Feature**
3. A default **Screen Space Overlay** preset is assigned automatically
4. Add a UI Image and assign the included `UI_Blur_Universal` material
5. Set the material's **Canvas Mode** to match your Canvas render mode

---

## Canvas Space Types

| Space Type | Render Event | Use With |
|---|---|---|
| Overlay Before Post-Processing | BeforeRenderingPostProcessing | Canvas — Screen Space Overlay (blur captured before bloom/tonemapping) |
| Overlay After Post-Processing | AfterRenderingPostProcessing | Canvas — Screen Space Overlay (blur includes post-processing output) |
| Screen Space Camera | BeforeRenderingTransparents | Canvas — Screen Space Camera |
| World Space | AfterRenderingSkybox | Canvas — World Space |
| World Objects | AfterRenderingSkybox | 3D meshes using the WorldObject shader |

> **Duplicate rule:** If two presets share the same Space Type, only the first active one
> executes. A warning is shown in the Inspector and a runtime guard skips duplicates.

---

## Blur Methods

| Method | Cost | Best For |
|---|---|---|
| Dual Kawase | ●●○○○ | Frosted-glass UI panels — widest radius at lowest cost |
| Custom Kawase | ●●○○○ | Fast circular blur with tunable tap count |
| Poisson Disk | ●●●○○ | Cinematic bokeh — eliminates grid sampling artifacts |
| Gaussian Separable | ●●●○○ | Mathematically exact Gaussian circle |
| Tent / Hex | ●●●●● | Hexagonal lens bokeh — best for cinematic moments |
| Radial | ●●○○○ | Zoom blur, impact effects, cutscenes |

---

## Parameters

### All methods

| Parameter | Description |
|---|---|
| Is Active | Enable/disable this preset at runtime without removing it |
| Space Type | Which canvas space and render event this preset targets |
| Blur Method | Algorithm used for this preset |

### Dual Kawase

| Parameter | Description |
|---|---|
| Iterations | Depth of the down/up mip pyramid — more = wider blur |
| Blur Offset | Tap spread multiplier (scales with screen resolution) |

### Custom Kawase / Poisson Disk

| Parameter | Description |
|---|---|
| Iterations | Number of ping-pong blur passes |
| Blur Offset | Tap spread per pass (scales with screen resolution) |
| Taps | Number of samples per pass (3–12) |
| Downsample | RT resolution divisor — auto-adjusted for screens wider than 1920 px |
| Use IGN Rotation | (Poisson only) Per-pixel sincos rotation — higher quality, tiny extra cost |

### Gaussian Separable / Tent Hex

| Parameter | Description |
|---|---|
| Gaussian Radius | Sample radius in pixels (scales with screen resolution, capped at 64) |
| Downsample | RT resolution divisor |

### Radial

| Parameter | Description |
|---|---|
| Radial Taps | Number of samples arranged around the circle (4–32) |
| Radial Radius | Ring radius in texels (scales with screen resolution) |
| Downsample | RT resolution divisor |

### Performance tip
 
Each active preset runs its full pass chain every frame, regardless of whether any UI is currently visible.
Only add presets for the space types you actually use — if your project only uses Screen Space Overlay, there is no reason to keep a World Space preset in the list.

---

## UI_Blur_Universal Shader

For use with UI Image components and Canvas panels.

| Property | Description |
|---|---|
| Canvas Mode | Must match the Canvas render mode: Screen Space Overlay / Screen Space Camera / World Space |
| Frost Tint | Color tint blended over the blurred background |
| Frost Strength | Tint blend weight (0 = no tint, 1 = full tint) |
| Brightness | Multiplier applied after tint |
| Desaturate | Desaturation amount (0 = full color, 1 = grayscale) |
| Edge Strength | Vignette darkening toward panel edges |
| Normal Map | Texture driving glass surface distortion |
| Normal Distortion Strength | Distortion intensity |
| Normal Mapping Mode | `Standard Tile` / `Local Fixed Scale` / `Screen Space Fix` |

---

## WorldObject Shader

For 3D world-space glass meshes (windows, floors, panels). Requires a preset with
Space Type set to **World Objects**.

| Property | Description |
|---|---|
| Frost Tint / Strength | Same as UI shader |
| Brightness / Desaturate | Same as UI shader |
| Refraction Index | IOR offset — how strongly the surface bends the background sample |
| Normal Map | Surface detail used to perturb the refraction direction |
| Normal Map Strength | Refraction bend intensity |

---

## Runtime API

### Toggle presets

```csharp
// Enable or disable all presets targeting a space type
Blur.SetPresetsActive(CanvasSpaceType.ScreenSpaceCamera, false);
Blur.SetPresetsActive(CanvasSpaceType.OverlayBeforePostProcessing, true);

// Subscribe to the event fired when a preset's active state changes
Blur.OnPresetActiveChanged += preset =>
    Debug.Log($"{preset.name} is now {(preset.isActive ? "active" : "inactive")}");
```

Individual presets can also be toggled directly via `BlurPreset.isActive`.

### Read and modify presets

```csharp
// Get the first preset for a given space type (returns null if not found)
BlurPreset overlay = Blur.GetPreset(CanvasSpaceType.OverlayBeforePostProcessing);
if (overlay != null) overlay.blurOffset = 2.5f;

// Get all presets as a snapshot list
List<BlurPreset> all = Blur.GetAllPresets();
foreach (var p in all)
    Debug.Log($"{p.name}: {p.blurMethod}, active={p.isActive}");
```

### Intensity — smooth fade without pop-in

`SetIntensity` scales all continuous parameters (`blurOffset`, `gaussianRadius`,
`radialRadius`) by a multiplier each frame. Unlike toggling `isActive`, intensity 0
keeps the pass alive so there is no one-frame black flash when the blur returns.

```csharp
// Instant set
Blur.SetIntensity(CanvasSpaceType.OverlayBeforePostProcessing, 0.5f);

// Read current value
float current = Blur.GetIntensity(CanvasSpaceType.OverlayBeforePostProcessing);

// Smooth fade in Update or a coroutine
void Update()
{
    float target = _menuOpen ? 1f : 0f;
    float current = Blur.GetIntensity(CanvasSpaceType.OverlayBeforePostProcessing);
    Blur.SetIntensity(
        CanvasSpaceType.OverlayBeforePostProcessing,
        Mathf.MoveTowards(current, target, Time.deltaTime * 3f));
}
```

> **Note:** Intensity affects only continuous spatial parameters. Discrete values
> (`iterations`, `taps`, `radialTaps`) are never fractional and are not scaled.

---

## Folder Structure

```
Assets/URPUIBlur/
├── Runtime/
│   ├── Scripts/            Blur.cs, BlurPreset.cs, *BlurPass.cs
│   ├── Shaders/            All .shader files
│   └── URPUIBlur.Runtime.asmdef
├── Materials/
│   ├── UI_Blur_Universal.mat
│   └── Blur_WorldSurface.mat
├── Editor/
│   ├── BlurEditor.cs       Custom Inspector for Blur (Renderer Feature)
│   ├── BlurScriptable.cs   Custom Inspector for BlurPreset
│   ├── BlurSetupWizard.cs  Setup Wizard window
│   └── URPUIBlur.Editor.asmdef
├── Presets/
│   └── UI-Overlay.asset    Default preset (auto-assigned by Setup Wizard)
├── Samples~/
│   └── Demo/               Interactive comparison scene
└── Documentation~/
    ├── CHANGELOG.md
    └── README.md
```

---

## Troubleshooting

**UI panel is black / shows no blur**
- Confirm the Blur Renderer Feature is added and enabled on the active URP Renderer asset
- Check that the preset's Space Type matches the Canvas render mode
- Check that the material's Canvas Mode dropdown matches the Canvas render mode

**Two presets on the same Space Type — only one works**
- This is by design. Each Space Type writes to one global texture; duplicates are
  skipped at runtime (first active preset wins). Assign different Space Types if you
  need multiple simultaneous blur effects.

**Blur is stretched horizontally**
- This is fixed in Gaussian and Tent/Hex passes via `_TargetTexelSize`. If you see
  stretching, confirm you are using the included shader files and not an older version.

**Performance is poor on mobile**
- Switch to **Dual Kawase** (lowest cost per quality)
- Increase **Downsample** to 4
- Reduce **Iterations** to 2

**Setup Wizard does not appear**
- Open it manually via `Tools → URPUIBlur → Setup Wizard`
- If the wizard shows no renderers, create a URP Universal Renderer asset first:
  `Assets → Create → Rendering → URP Universal Renderer`

---

## License

See LICENSE included with this package.
