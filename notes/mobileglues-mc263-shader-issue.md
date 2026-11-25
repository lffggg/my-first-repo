# MC 26.3 on Android — Graphics Stack Debug Log

Date: 2026-11-25 | Device: vivo V2410A / Adreno 710 (SM6450) / Android 15 / hardware Vulkan 1.1.0
Launcher: ZPCL 1.5.6 (PCL-style UI) | MC: Java 26.3 vanilla, no mods
Upstream issue: MobileGL-Dev/MobileGlues#66

## Symptom

Vanilla MC 26.3 dies during initial shader load under MobileGlues 2.0.0. All 37
`minecraft:pipeline/*` programs fail to compile, each reporting `_uniform` /
`_instance_xx_xx` at a fixed line number of `0:224`. No mods, no resource packs,
no shaderpack. 100% reproducible on a clean instance.

## Root cause of the MG crash (third-party reproduction, not independently verified)

`process_uniform_declarations()` scans with `compare(scan_pos, 7, "uniform")`
without checking token boundaries, so the `uniform` substring inside the
SPIRV-Cross-generated identifier `_uniform_instance_00_03` is treated as the keyword
and the code is chopped into invalid GLSL.

Secondary: `oit_add_transmittance.glsl` indexes fragment output arrays with a
runtime variable, which mobile drivers reject unconditionally.

## Attempt 1 — MobileGlues settings (all tested, all crash identically)

- "Ignore shader/program error" toggle (setting does not persist, and it cannot
  bypass MC's own program validation)
- Hand-written `mg.conf` with `ignoreError=1` / all multidraw backends forced to unroll
- ANGLE enabled and disabled
- `ARB_compute_shader` / DSA / `timer_query` disabled
- GLSL cache cleared / custom GL version disabled / heap raised to 2560M

## Attempt 2 — self-built patched library: built, but not injectable

Built MobileGlues at `97558a6` carrying **only the two shader fixes** (the multidraw
reordering and framebuffer-bind caching hunks were deliberately dropped).
Toolchain: NDK r27, arm64-v8a, android-21, Release + ThinLTO, stripped.

```
size    6.2 MB
sha256  ffaf9bbd3157b95a3e7c4cf3629002d1f28fb43d4250c4e61172c8e3602ef153
```

Patch markers verified inside the binary: `_mg_out_` and `fail_%u.glsl`.

**Result: cannot be loaded.** ZPCL's renderer list is a fixed built-in set:

```
Krypton Wrapper / GL4ES / Kopper Zink / VirGLRenderer
Freedreno (Adreno) / Panfrost (Mali) / MobileGL
MobileGlues (OpenGL 4.0, 1.17+)
```

There is no local-renderer-import entry. MG ships as a launcher component, so a loose
`.so` has no injection point. A working patched build is useless here.

Only exception would be replacing the library inside the launcher's own package,
which means repacking and re-signing the launcher — out of scope.

## Attempt 3 — bypass MG entirely with Kopper Zink

Configuration: Renderer = Kopper Zink, Graphics API = follow game setting, Driver = Turnip.
Environment as reported by the launcher:

```
POJAV_RENDERER            = opengles3_desktopgl_zink_kopper
MESA_LOADER_DRIVER_OVERRIDE = zink
```

Result:

```
No supported graphics backend was found.
- Tried OpenGL:  glGetError mismatch
- Tried Vulkan:  No Vulkan capable devices
```

Relevant log lines:

```
Vulkan: 1.1.0 (below 1.2, which Minecraft 26.2+ requires for its Vulkan backend)
EGLBridge: LWJGL-side Vulkan loader requested the Vulkan handle
AdrenoSupp: Loaded Turnip, loader address: 0x...
```

## Interpretation

- Every renderer other than MG — Zink, MobileGL, Freedreno, and MC's own native
  Vulkan backend — requires a working Vulkan device.
- Turnip's `.so` **does load**, yet zero Vulkan physical devices are enumerated.
  So the built-in Turnip build carries no Adreno 710 entry (chip_id `0x07010000`).
- The `glGetError mismatch` is a consequence of the missing Vulkan device, not a
  defect in Zink itself. Both errors share one root cause.
- Only MG can start, because it runs on the stock Adreno GLES driver — and MG carries
  the `_uniform` bug. The machine is therefore locked onto the one broken path.
- Hardware Vulkan 1.1.0 is also below the 1.2 threshold MC 26.3 wants for its native
  Vulkan backend, so that escape route is closed as well.

## Next steps

1. Import an Adreno-710-capable Turnip build —
   `Turnip v26.3.0-20260717-r5 (710/720 Test)`, Vulkan 1.4.354, community build,
   package format `meta.json` + `libvulkan_freedreno.so` — *if* the launcher exposes a
   driver-import entry.
   Success criterion is objective: the startup log must stop saying `Vulkan: 1.1.0`
   and report `1.4.x`.
2. Fallback that always works: run a pre-26.3 MC version under MG.

## Open questions

- Does ZPCL 1.5.6 expose driver import at all?
- Does the A710 Turnip test build initialise on SM6450 / Android 15?
- Is the `_uniform` mangling fixed in any released MobileGlues build after 2.0.0?

## References

- Upstream issue: https://github.com/MobileGL-Dev/MobileGlues/issues/66
- Community patch, Mali-validated only: https://github.com/Dap-boss/MobileGlues-MC26.3-patch
- Adreno 710/720 Turnip test build: https://github.com/The412Banner/Banners-Turnip
