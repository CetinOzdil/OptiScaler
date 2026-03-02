# DLSS-RR → NRD Integration Analysis (OptiScaler)

## 1) DLSS-RR inputs available in current FSR-RR path

`FSRDFeatureDx12::PrepareDenoiseConvInput` already reads these DLSS-RR resources and camera transforms:

- Core temporal buffers
  - `NVSDK_NGX_Parameter_Color`
  - `NVSDK_NGX_Parameter_MotionVectors`
  - `NVSDK_NGX_Parameter_Depth`
- Material / gbuffer
  - `NVSDK_NGX_Parameter_GBuffer_Normals`
  - `NVSDK_NGX_Parameter_GBuffer_Roughness` (optional when roughness is packed)
  - `NVSDK_NGX_Parameter_DiffuseAlbedo`
  - `NVSDK_NGX_Parameter_SpecularAlbedo`
- Auxiliary
  - `NVSDK_NGX_Parameter_DLSS_Input_Bias_Current_Color_Mask` (reactive/bias mask)
  - `NVSDK_NGX_Parameter_DLSSD_SpecularHitDistance` (optional)
- Camera / matrices
  - `NVSDK_NGX_Parameter_DLSS_WORLD_TO_VIEW_MATRIX`
  - `NVSDK_NGX_Parameter_DLSS_VIEW_TO_CLIP_MATRIX`
  - plus jitter and MV scales from standard NGX keys in `PrepareDenoiserInput`.

## 2) Cross-check with NRD working modes

Practical mode fit from available inputs:

- **REBLUR**
  - Works with radiance + depth + normal/roughness + motion vectors + albedo.
  - Best default when specular hit distance is absent.
- **RELAX**
  - Benefits from richer specular tracking (e.g. spec hit distance).
  - Good fit when `DLSSD_SpecularHitDistance` is available.

Resulting auto-selection policy implemented:

- If `NrdWorkingMode=Auto` and spec hit distance exists ⇒ choose **RELAX**.
- Else choose **REBLUR**.

## 3) Implementation plan

1. Keep existing DLSS-RR input acquisition/conversion path as the canonical source.
2. Add a denoiser backend selector in config:
   - `FfxDenoiserBackend` (`0=FFX`, `1=NRD`).
3. Add NRD mode selector:
   - `NrdWorkingMode` (`0=auto`, `1=REBLUR`, `2=RELAX`).
4. Build an NRD dispatch plan from already-available DLSS-RR/FSR-RR-ready data.
5. Wire a backend switch in `Evaluate()`.
6. Keep FFX dispatch unchanged as default and stable path.

## 4) What is implemented in this patch

- Added backend and mode config keys in `Config.h`.
- Added NRD planning structures/methods in `FSRDFeatureDx12`:
  - `NrdDispatchPlan`
  - `BuildNrdDispatchPlan(...)`
  - `DispatchNrdDenoiser(...)`
- Added runtime backend branching in `Evaluate()`:
  - FFX path unchanged.
  - NRD path now performs input compatibility/mode planning and logs a full dispatch summary.

## 5) Current limitation

The repository does not currently vendor/link NVIDIA NRD binaries/headers, so this patch implements the NRD integration surface and planning path, but not NRD runtime execution yet.

To complete full NRD execution, next step is adding NRD SDK dependency and replacing `DispatchNrdDenoiser` placeholder body with actual NRD context creation, permanent pool management, and per-frame dispatch.
