# Same game, same renderer, different launcher: FCL runs, ZPCL crashes

Written 2026-11-26. Companion to `mobileglues-mc263-shader-issue.md`.

**中文摘要**：同一台手机、同一个 MC 26.3 原版、同一个 MobileGlues 渲染器、两边都没有手动放 `.so`，FCL 1.3.4.8 能进世界，ZPCL 1.5.6 必崩。这篇不宣布答案，只把**差异变量逐条拆开**，标出哪一条最可疑、怎么用一次实验把它排除。

## What is actually identical

| Item | FCL 1.3.4.8 | ZPCL 1.5.6 |
|---|---|---|
| Device | vivo V2410A / SM6450 / Adreno 710 | same |
| OS | Android 15 | same |
| Game | MC Java 26.3 vanilla, no mods, no packs | same |
| Renderer family | MobileGlues | MobileGlues |
| Renderer label shown | "OpenGL 4.0" | "OpenGL 4.0" |
| Manually installed `.so` | none | none |
| Driver | stock Adreno GLES | stock Adreno GLES |
| Result | world loads, renders normally | all 37 `minecraft:pipeline/*` fail, `0:224`, cannot start |

**The first thing to kill is the idea that the identical label means an identical binary.**
`"OpenGL 4.0"` is a *profile name* the launcher prints for the renderer slot. It identifies
the renderer family, not the build. Two launchers can print the same string while shipping
different `libmobileglues.so` revisions.

## Why this matters

The documented `_uniform` bug is a pure code-scanning defect:
`process_uniform_declarations()` matches the 7 characters `uniform` without checking token
boundaries, so the substring inside `_uniform_instance_00_03` is taken as a keyword and the
declaration is chopped. A defect like that is deterministic *for a given build*. If it does
not fire in FCL, then either

- that build no longer has the defect (fixed upstream after 2.0.0), or
- the code path that reaches the scanner is not taken under FCL's conditions, or
- the same broken output is produced but something downstream tolerates it in FCL.

Each of those maps to a different variable, and the rest of this note is about separating
them.

## Ranked hypotheses

### H1 — Different MobileGlues build (highest suspicion, cheapest to settle)

**Mechanism.** ZPCL ships whatever MG revision its release was pinned to; FCL ships its own.
If FCL's differs, the crash is simply "old build", and every other hypothesis here becomes
noise.

**Why plausible.** The renderer label proves nothing; MG has no version string in the
launcher UI on either side; MG 2.0.0 is recent and upstream moves fast.

**How to test.** One measurement: the MG version/build line in the FCL startup log, compared
with ZPCL's. If FCL supports importing a renderer package, the decisive test is running
**the same `.so`** in both launchers.

**Confirming evidence.** Different load/build strings, or same `.so` crashing in both.

### H2 — Launcher-injected MG configuration

**Mechanism.** MG reads its behaviour from a config (`mg.conf` and launcher-supplied settings:
ignore-error, GL version override, multidraw backend, shader cache path, `TU_DEBUG`, ANGLE on/off).
A launcher can force a value that walks MG into the vulnerable path.

**Why plausible.** ZPCL's advanced page exposes exactly these knobs (自定义 JVM 参数, 图形 API,
驱动, 跳过校验); launchers routinely write their own defaults into such configs.

**How to test.** Dump the effective `mg.conf`/renderer settings on both sides. Differences in
`ignoreError`, GL version, or multidraw backend are read directly.

### H3 — Parallel shader compilation race (the most *interesting* hypothesis)

**Mechanism.** MC 26.3 compiles its pipeline set at startup. If that compile runs on multiple
threads, MG's scanner is entered concurrently. A token scanner that keeps position or buffer
state outside the call frame is not re-entrant: two threads can interleave `scan_pos` and
produce mangled output even from correct source. That failure mode looks exactly like a
"bug that only happens sometimes, on some launchers" — and it explains why the identical game
correctly compiles 37 programs on one launcher and mangles all 37 on another.

**Why plausible.** Both outcomes here are all-or-nothing (0/37 vs 37/37), which is consistent
with a race whose scheduling differs wholesale between two environments (different JVM args,
different `ActiveProcessorCount`, different thread pools), rather than with a per-shader bug.

**How to test.** Pin the thread count identically: pass `-XX:ActiveProcessorCount=2` (or 4) in
**both** launchers and re-run. If ZPCL starts working (or starts failing intermittently) with a
changed core count, H3 is confirmed. A second signal: reproduce in ZPCL with a single-threaded
JVM and see whether the error count changes.

**Confirming evidence.** A failure count that moves with core count, or a partially-failing run
(e.g. 12 of 37) that never occurs otherwise.

### H4 — Stale shader cache

**Mechanism.** ZPCL may be serving previously compiled (broken) artifacts from its cache;
FCL's cache is fresh or stored elsewhere.

**Why plausible.** Cheap to test, and cache paths differ per launcher.

**How to test.** Clear the GLSL/shader cache on the ZPCL side only, then re-run. Note: this was
already tried once on ZPCL and the crash persisted — so H4 is *weak*, but "cache cleared" must
mean *every* MG cache location, not just the game's.

### H5 — Different EGL/GL context creation

**Mechanism.** Context version and exposed extension set decide which branches inside MG and
inside MC's capability detection get taken. A GLES 3.2 context with one extension set can reach
code a GLES 3.0 context never touches.

**Why plausible.** Both launchers use their own forks of the same Pojav-lineage EGL shim, but
their requested context attributes are configured independently.

**How to test.** Compare the `GL_VERSION` / `GL_RENDERER` / extension strings in both startup
logs. Any divergence in context version or in a shader-related extension
(`GL_ARB_shader_*`, `GL_EXT_*`) is a lead.

### H6 — Different LWJGL / GL loading shim

**Mechanism.** MC's GL calls go through a shim. A different LWJGL build can load functions
differently (direct vs. dispatched), change call order, or change which validation runs first.

**Why plausible.** Weakest of the six: the failure is inside MG's GLSL translation, upstream of
anything LWJGL does. Kept here only because "all 37 fail identically" is suspiciously uniform.

## The experiment matrix that settles this

| # | Change | Rules out | Cost |
|---|---|---|---|
| 1 | Read both logs for the MG version string | H1 | minutes |
| 2 | Same `.so` in both launchers (if import exists) | H1, H2, H5, H6 | needs import entry |
| 3 | `-XX:ActiveProcessorCount=2` in both | **H3** | one run each |
| 4 | Dump effective `mg.conf` on both | H2 | minutes |
| 5 | Compare `GL_VERSION` + extension list | H5 | minutes |
| 6 | Wipe all MG caches in ZPCL, re-run | H4 | one run |

**Order matters.** Run 3 early: it is the only test that can turn "some launchers crash" into
"a thread race crashes", and it costs nothing but a JVM flag.

## Strings to grep in the startup log

```
libmobileglues        # the loaded renderer library
MobileGlues           # version/build banner, if printed
GL_VERSION            # context version actually granted
GL_RENDERER
GL_EXTENSIONS
POJAV_RENDERER
LIBGL_ES / LIBGL_GL
ActiveProcessorCount  # or the reported core count
```

## What this note does **not** claim

- It does **not** claim ZPCL's MG is an older build. That is H1, untested.
- It does **not** claim the upstream bug is fixed. A non-triggering environment does not
  disprove a defect; it shows the defect is conditional on environment.
- It does **not** retract earlier findings. The Vulkan/Turnip conclusion stands independently:
  on this hardware every Vulkan-dependent path is still closed.

## Status

**Unresolved.** One working configuration exists (FCL 1.3.4.8). The cause of the divergence is
not yet measured — only ranked. The next measurement is #1 or #3 in the matrix above.

## References

- Crash record and attempts: `notes/mobileglues-mc263-shader-issue.md`
- Upstream issue: https://github.com/MobileGL-Dev/MobileGlues/issues/66
