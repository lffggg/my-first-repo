# MC 26.3 on Android — Graphics Stack Debug Log

Date: 2026-11-25, revised 2026-11-26 | Device: vivo V2410A / Adreno 710 (SM6450) /
Android 15 / hardware Vulkan 1.1.0
Launchers: **ZPCL 1.5.6** (crash) → **FCL 1.3.4.8** (works) | MC: Java 26.3 vanilla, no mods
Upstream issue: MobileGL-Dev/MobileGlues#66

## Status — a working path was found

| Launcher | MC | Renderer | Result |
|---|---|---|---|
| ZPCL 1.5.6 | 26.3 | MobileGlues 2.0.0 (bundled) | 37 pipeline programs fail, cannot start |
| FCL 1.3.4.8 | 26.3 | MobileGlues (bundled, no manual `.so`) | loads the world, renders normally |

Same game version, same renderer family, same GPU, same OS. **The crash is therefore
not an unconditional property of MobileGlues on MC 26.3 — it is a property of how a
specific launcher installs and configures it.**

## Symptom (ZPCL only)

Vanilla MC 26.3 dies during initial shader load under MobileGlues. All 37
`minecraft:pipeline/*` programs fail to compile, each reporting `_uniform` /
`_instance_xx_xx` at a fixed line number of `0:224`. No mods, no resource packs,
no shaderpack. 100% reproducible on a clean instance, in ZPCL only.

## Reported root cause (third-party reproduction, not independently verified)

`process_uniform_declarations()` scans with `compare(scan_pos, 7, "uniform")`
without checking token boundaries, so the `uniform` substring inside the
SPIRV-Cross-generated identifier `_uniform_instance_00_03` is treated as the keyword
and the code is chopped into invalid GLSL.

Secondary: `oit_add_transmittance.glsl` indexes fragment output arrays with a
runtime variable, which mobile drivers reject unconditionally.

**Amendment:** if this were the whole story the crash would have to reproduce in FCL
too. It does not. Either the FCL build already carries the boundary fix, or the
mangling path is only reached under specific context/config conditions. Which one is
still unknown — see Attempt 4.

## Attempt 1 — MobileGlues settings (all tested in ZPCL, all crash identically)

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

**Result: cannot be loaded in ZPCL.** Its renderer list is a fixed built-in set:

```
Krypton Wrapper / GL4ES / Kopper Zink / VirGLRenderer
Freedreno (Adreno) / Panfrost (Mali) / MobileGL
MobileGlues (OpenGL 4.0, 1.17+)
```

There is no local-renderer-import entry. MG ships as a launcher component, so a loose
`.so` has no injection point.

Only exception would be replacing the library inside the launcher's own package,
which means repacking and re-signing the launcher — out of scope.

## Attempt 3 — bypass MG entirely with Kopper Zink (ZPCL)

Configuration: Renderer = Kopper Zink, Graphics API = follow game setting, Driver = Turnip.

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

```
Vulkan: 1.1.0 (below 1.2, which Minecraft 26.2+ requires for its Vulkan backend)
EGLBridge: LWJGL-side Vulkan loader requested the Vulkan handle
AdrenoSupp: Loaded Turnip, loader address: 0x...
```

## Attempt 4 — same game, same renderer, different launcher: **works**

**FCL 1.3.4.8**, MC 26.3, MobileGlues renderer, no manually installed `.so` (the
launcher's own bundled build), default settings.

**Result: the world loads and renders normally.** No pipeline errors.

What this establishes and what it does not:

- **Established:** the `_uniform` mangling is *not* triggered unconditionally on MC
  26.3 + Adreno 710. The same game and the same renderer family reach a running world.
- **Not established:** which variable is responsible. Candidates, in order of
  suspicion:
  1. **Different MG build.** ZPCL's bundled `MobileGlues (OpenGL 4.0, 1.17+)` may be an
     older revision than FCL's. Both show the same "OpenGL 4.0" label, but the label is
     a renderer name, not a build identity.
  2. **Launcher-injected MG configuration** (`mg.conf`, `ignoreError`, GL version
     override, `TU_DEBUG`, shader cache path).
  3. **Different GLFW / EGL context creation**, putting MG on a different code path
     (different context version or extension set).
  4. **Stale shader cache** on the ZPCL side serving broken compiled artifacts.
- **Open:** a byte-level comparison of the two `libmobileglues.so` builds would settle
  candidate 1 immediately. Not yet done.

## Interpretation (revised)

- Every renderer other than MG — Zink, MobileGL, Freedreno, and MC's own native
  Vulkan backend — requires a working Vulkan device. On this device they all fail.
- Turnip's `.so` **does load**, yet zero Vulkan physical devices are enumerated. The
  launcher's built-in Turnip build carries no Adreno 710 entry (chip_id `0x07010000`).
  The `glGetError mismatch` is a consequence of that, not a defect in Zink.
- Hardware Vulkan 1.1.0 is also below the 1.2 threshold MC 26.3 wants, so the native
  Vulkan backend cannot be used either.
- **So the only usable backend here is MobileGlues on the stock Adreno GLES driver —
  and that backend does work, provided the right launcher ships the right MG build.**
  The earlier conclusion that "the machine is locked onto the one broken path" was
  wrong, and is retracted here.

## Next steps

1. **Play 26.3 through FCL.** That is the working path on this hardware today; no
   downgrade needed.
2. Identify the exact MG build FCL bundles (version string in the startup log or in
   the launcher's renderer settings) and compare it with ZPCL's. That is the one
   measurement that turns this from "works over there" into "here is why it broke here".
3. Import an Adreno-710-capable Turnip build —
   `Turnip v26.3.0-20260717-r5 (710/720 Test)`, Vulkan 1.4.354, community build,
   package format `meta.json` + `libvulkan_freedreno.so` — *if* the launcher exposes a
   driver-import entry. Success criterion is objective: the startup log must stop
   saying `Vulkan: 1.1.0` and report `1.4.x`.

## Open questions

- Which MG build does FCL 1.3.4.8 bundle, and does it differ from ZPCL's?
- Does ZPCL 1.5.6 expose renderer/driver import at all?
- Does the A710 Turnip test build initialise on SM6450 / Android 15?
- Is the `_uniform` mangling fixed in any released MobileGlues build after 2.0.0?

## References

- Upstream issue: https://github.com/MobileGL-Dev/MobileGlues/issues/66
- Community patch, Mali-validated only: https://github.com/Dap-boss/MobileGlues-MC26.3-patch
- Adreno 710/720 Turnip test build: https://github.com/The412Banner/Banners-Turnip
