# MC 26.3 Terrain Shader Compilation Failure — Debugging Notes

Date: 2026-11-23 | Device: vivo V2410A / Adreno 710 / Android 15 / Vulkan 1.1.0
Upstream issue: MobileGL-Dev/MobileGlues#66

## Symptom

Vanilla MC 26.3 crashes on startup under MG 2.0.0. All 37 `minecraft:pipeline/*`
programs fail to compile, reporting `_uniform` / `_instance_xx_xx` errors at a fixed
line number of `0:224`.

## Ruled out (all tested, all crash identically)

- "Ignore shader/program error" toggle (setting does not persist, and it cannot
  bypass MC's own program validation)
- Hand-written `mg.conf` with `ignoreError=1` / all multidraw backends forced to unroll
- ANGLE enabled and disabled
- `ARB_compute_shader` / DSA / `timer_query` disabled
- GLSL cache cleared / custom GL version disabled / heap raised to 2560M

## Root cause (source: third-party reproduction, not independently verified by me)

MG's `process_uniform_declarations()` scans with `compare(scan_pos, 7, "uniform")`
without checking token boundaries, so the `uniform` substring inside the
SPIRV-Cross-generated identifier `_uniform_instance_00_03` is treated as the keyword,
and the code is chopped into invalid GLSL.

Secondary: `oit_add_transmittance.glsl` indexes fragment output arrays with a runtime
variable, which mobile drivers reject unconditionally.

## Still unverified

- Behaviour of community patch `v1.0-mc26.3` on Adreno 710 (author only validated on Mali)
- Feasibility of using Turnip to raise the Vulkan version and switch to the native
  Vulkan backend

## Conclusion

This is an error in MG's software translation layer, independent of GPU vendor.
The role of Vulkan 1.1 is that it removes any path around that layer.

## References

- Upstream issue: https://github.com/MobileGL-Dev/MobileGlues/issues/66
- Community patch (unofficial, Mali-validated only):
  https://github.com/Dap-boss/MobileGlues-MC26.3-patch
