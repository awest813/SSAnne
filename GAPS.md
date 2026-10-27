# GAPS.md — audit gap list

Audit of this fork as of 2026-10-27 (upstream HEAD `c99ed8e`, "project is no
longer maintained"). Scope: every tracked file. Nothing was built or run —
the `disasm/` and `recomp-ui/` submodules and the `lib/` companion repos are
not present in a fresh clone (see G-1). Line numbers refer to this commit.

Priority key: **P0** blocks anyone from building/continuing · **P1** user-facing
bug · **P2** robustness/security · **P3** portability · **P4** docs/hygiene ·
**P5** feature backlog.

---

## P0 — Reproducible build

- [ ] **G-1 Companion repos are not provisioned or pinned.** `CMakeLists.txt:71,80,179-186`
      needs `lib/rt64`, `lib/N64ModernRuntime`, `lib/concurrentqueue`, `lib/SlotMap`
      (and `_package_release.ps1` needs `lib/GamepadMotionHelpers`). `lib/` is
      gitignored (`.gitignore:90`); no submodule, setup script, or pin file creates it.
      N64ModernRuntime / rt64 SHAs exist only in prose comments in `n64recomp.pin`.
      → Add submodules (or a pin + fetch step) for every `lib/*` dependency.
- [ ] **G-2 `n64recomp.pin` is not enforced and points at the wrong remote.**
      The pin header claims CMake asserts it (`n64recomp.pin:3-4`); CMake never reads
      it and actually builds `${N64MODERN_RUNTIME_ROOT}/N64Recomp`. The pin names
      mstan/N64Recomp, but `setup.sh:15` / `setup.bat:13` clone upstream
      `N64Recomp/N64Recomp`, and only *print* a note on SHA mismatch (`setup.sh:37-43`).
- [ ] **G-3 `recomp-ui` submodule is never initialised.** Setup only inits `disasm`
      (`setup.sh:46`, `setup.bat:52`), but `CMakeLists.txt:313` includes
      `recomp-ui/recomp_ui.cmake`. Pin is also inconsistent: `.gitmodules` says
      `branch = work/n64-console`, commit `30d5de9` says "pin to master (23fba40)",
      gitlink is actually `4ed61d0`.
- [ ] **G-4 No scripted path from ROM → `generated/`.** Neither setup script runs the
      pret build, N64RecompCLI, or RSPRecomp. `docs/disasm-build.md` covers the disasm
      only. Document/script the full pipeline end-to-end.
- [ ] **G-5 Hand-edited generated RSP code.** `rsp/aspMain.cpp:4096` adds
      `case 0x1BB4` + label by hand; `0x1BB4` is not in
      `aspMain.us.1.0.toml` `extra_indirect_branch_targets`, so a regen silently loses
      menu-music dispatch. `rsp/njpgdspMain.cpp` has no toml at all (copied from
      Zelda64Recomp, `CMakeLists.txt:164`). → Move edits into the tomls.
- [ ] **G-6 No CI, no tests.** `.github/` contains only an image; no `tests/`, no
      `enable_testing()`. Minimum: a Windows CI job that configures + builds with
      pinned deps (ROM-free parts), plus a smoke test for `input_bindings` /
      `transfer_pak` cfg parsing.
- [ ] **G-7 Stale CMake cache leaks diag options into release.** `build_ssanne.bat:60`
      doesn't reset `PSR_TRACE_HOOKS` / `PSR_GEO_PROBE`, and `build_trace.bat` /
      `build_diag.bat` share `build/`. `build_diag.bat:3` comment is wrong (says
      TRACE_HOOKS=OFF, actually enables GEO_PROBE). `WITH_DEV_DIVERGENCE` is passed but
      not defined anywhere in `CMakeLists.txt`.
- [ ] **G-8 GPL source correspondence.** `n64recomp.pin` notes the v0.4.0 release was
      built against a different engine SHA than the pin; there is no record of which
      N64MR / rt64 / N64Recomp SHAs produced v0.4.6-beta. Record exact SHAs per release.

## P1 — Gameplay bugs

- [ ] **G-9 #13 Rental detail-view crash** (Free Battle → Rental → 2nd/3rd Pokémon
      detail). Fragment `0xEF` evicts an in-use section → `get_function` miss → wild
      jump. Repro 3/3. (`ISSUES.md:84`)
- [ ] **G-10 #10 residual — static on announcer lines / crowd bed.** Localised to the
      CPU-streamed PCM side-chain (`n_alMainBusPull` rings fed by `func_80044EA4`);
      not fixed. (`ISSUES.md:52-82`)
- [ ] **G-11 #9 Pokémon switch-in crash (untracked).** Analysis exists only in
      `_issue_logs/ANALYSIS_issue9_geo.md` (stale fragment instance → garbage geo
      layout opcode). Design for the fix is in `plans/multi_instance_fragment_residency.md`
      but not implemented. Not listed in `ISSUES.md`.
- [ ] **G-12 #16 Gallery "enlarge" (C-Down) null-vtable crash (untracked).** Analysis
      in `_issue_logs/16/ANALYSIS.md`; likely same stale-fragment class as #9/#13.
      Not listed in `ISSUES.md`.
- [ ] **G-13 Party-select softlock in a cup (untracked).** Only a raw post-mortem
      (`_issue_logs/softlock_partyselect_cup.report.json`); no write-up.
- [ ] **G-14 Full cup / Gym Leader Castle runs unconfirmed.** README says never run
      end-to-end (`README.md:115-118`); `ISSUES.md:8-9` says validated. Re-verify and
      fix whichever doc is wrong. Given G-11/G-13 this is likely still open.
- [ ] **G-15 Fragment-id out-of-range clobber is unresolved.** `extras.c:912-926` applies
      a "functional fix" inside a diagnostic hook; `extras.c:948-977` records the
      root cause (fragment id −15 clobbering `gSegments[1]`) as unknown.
- [ ] **G-16 Workarounds that mask root causes** (keep, but track):
      - voluntary preemption (yield after 200 ms busy-wait) replaced three softlock
        patches — scheduling root causes unknown (`main.cpp:2011-2031`, `trace.h:38-42`);
      - forced `gExpansionRAMStart = 1` (`main.cpp:2170-2199`);
      - 100 ms sleep before `start_game` to dodge null VI mode (`main.cpp:2411-2430`);
      - `_Exit(0)` on window close to dodge a shutdown deadlock (`main.cpp:1442-1458`);
      - audio-UAF voice silencing on every pool pop with fixed vaddrs
        (`main.cpp:2236-2282`, `game.toml:3243-3255`).
      `TEMP_PATCHES.md` lists only retired entries; none of these are registered there.
- [ ] **G-17 Supersampling 2D residual noted in ISSUES.md**: transition-diagonal caveat and
      first-render 2D stale-pipeline state under supersampling ("exact RT64 path not yet
      pinned", `ISSUES.md:225-240`).

## P2 — Robustness & security

- [ ] **G-18 Unauthenticated debug server on by default in release.** Always started
      on `127.0.0.1:4371` (`main.cpp:2107-2114`); any local process can poke RDRAM
      (`rdram_poke`), write files to arbitrary paths (`*_dump` commands), suspend all
      threads, or `quit` (which skips the save-thread flush, `debug_server.cpp:2393`).
      Unbounded recv buffer, ignored partial `send()` (`debug_server.cpp:2438-2453`).
      → Gate behind `PSR_DEBUG_PORT` / a CMake option, default off in release.
- [ ] **G-19 Out-of-bounds RDRAM reads in live hooks.** `extras.c:2736-2741` reads
      offset `0x07906050` (~121 MiB) unchecked; `extras.c:1888,1897,2094,2845` bound-check
      against 1 GiB instead of 8 MiB; `main.cpp:282-288` `taskpcm_record` lacks the
      8 MiB check used at `:416`.
- [ ] **G-20 Post-mortem path hazards.** Suspends threads then calls CRT/`fprintf`/DbgHelp
      (`post_mortem.cpp:296-337`, `debug_server.cpp:2348-2380`) → possible hang; non-recursive
      mutex re-entered on a fault during dump (`post_mortem.cpp:844` + VEH at
      `main.cpp:1868-1886`); `atexit` writes an 8 MiB RDRAM dump on every clean exit
      (`main.cpp:1965`) into CWD-relative `build/` (silently fails if missing).
- [ ] **G-21 Struct mirror mismatch.** `post_mortem.cpp:173-184` (32 B) vs
      `debug_server.cpp:1091-1105` (40 B) mirror the same mesg struct — one always reports
      `size_mismatch`; op names 9/10 missing in post_mortem.
- [ ] **G-22 `input_bindings::get_binding` returns a reference after releasing the lock**
      (`input_bindings.cpp:322-325`).
- [ ] **G-23 `UNIMPL_LIBULTRA` aborts on `rmonPrintf`** and other rarely-hit libultra
      calls (`extras.c:163-168`); every `recomp_unhandled_*` aborts (`extras.c:81-124`).
- [ ] **G-24 Opt-in WASAPI output can go silent.** Reports success before device init
      and exits unless mix format is f32 stereo (`audio_host_probe.cpp:250-253,330-336`).
- [ ] **G-25 Always-on diagnostics cost in release.** ~34 MiB audcmd ring with mutex +
      4 KB copy per audio task (`main.cpp:393`); ~44 log/probe-only `game.toml` hooks
      print to stderr; `trace_mode = true` still set (`game.toml:23`). Trace ring is empty
      unless `PSR_TRACE_HOOKS=ON`, so crash logs from users have no function history.
- [ ] **G-26 `ares_worker::dispatch` blocks forever if the worker isn't running**
      (`include/ares_worker.h:40-58`).

## P3 — Portability (Windows x64 only today)

- [ ] **G-27 Linux/macOS cannot link.** SDL2 is fetched and linked only `if (WIN32)`
      (`CMakeLists.txt:227-255`); no `find_package(SDL2)` despite the comment. MSVC-only
      flags (`/MT`, `/Z7`, `/WHOLEARCHIVE`), `-march=nehalem` (blocks ARM).
- [ ] **G-28 Win32 APIs used unconditionally.** `main.cpp:37-41` (`windows.h`,
      `commdlg.h`, `dbghelp.h`); `debug_server.cpp` is Winsock/Toolhelp only;
      `post_mortem.cpp` is Win32 + x64 `CONTEXT` only (also breaks Windows ARM64).
- [ ] **G-29 No ROM file picker off Windows** (`main.cpp:2331-2333`); X11-only window
      handle with no Wayland/else branch (`main.cpp:1418-1422`).
- [ ] **G-30 Vulkan/Metal lack conservative raster** — the retired fragment-57 seam
      fix depends on it (D3D12 only, per `TEMP_PATCHES.md`); seams may return on Vulkan
      fallback. Launcher offers no Metal option (`recompui_launcher.cpp:362,476`).
- [ ] **G-31 Tooling assumes the author's machine.** `tools/derive_function_sizes.py:41-44`
      hardcodes WSL + `/mnt/f/Projects/…`; `diff_aspmain.py` / `peek_ostask.py` default to
      `../N64Recomp/build-aresbridge/…exe` and call `taskkill`; `setup.sh` uses `md5sum`
      (absent on macOS); `setup.bat` has no ROM MD5 check.

## P4 — Docs & repo hygiene

- [ ] **G-32 Docs reference files that don't exist:** `CLAUDE.md`, `tests/`,
      `ares-bridge/`, `overlays/`, `_crackle_data/`, `memory/*.md`, ~12 `tools/_*.py|ps1`
      scripts (gitignored by `.gitignore:55`, e.g. `_memmap_ring.py`, `_arcload_ring.py`,
      `_rdram_poke.py`, `_stream_tear2.py`, `_decompress_fragment78.py`),
      `F:\Projects\_audio_round2\AUDIO.md`. Either commit the tools or drop the refs.
- [ ] **G-33 README stale/contradictory:** Layout tree lists `ares-bridge/`, `tests/`,
      `n64recomp/`; "per-button remapping is planned" (`README.md:158`) though the launcher
      enables a rebind page; cup/GLC status contradicts `ISSUES.md` (G-14).
- [ ] **G-34 `tools/README.md` says inventory is empty**; 11 tools exist. Also refers to
      `[[patches.stubs]]`, which `game.toml` doesn't use ("NO STUBS", `game.toml:6`).
- [ ] **G-35 `DEBUG.md` "Tools (planned)" all still TODO;** `extract_overlays.py` described
      as present but was removed.
- [ ] **G-36 `ISSUES.md` historical checklist is stale** — dozens of `[ ]` items
      (first boot, ELF wiring, runner target…) are evidently done. Close or delete them.
- [ ] **G-37 `ISSUES.md` "cleanup still owed"** (`ISSUES.md:353-357`): `recomp_cf_note`
      scaffolding, `PKMNSTADIUM_TRACE_HOOKS` CMake block, `[jrdiag]` block in `extras.c`
      (`extras.c:431-438,477-490,2862-2883`).
- [ ] **G-38 Env vars undocumented.** ~25 runtime knobs are read but not in README/docs,
      e.g. `PSR_DEBUG_PORT`, `PSR_TURBO`, `PSR_DISABLE_CRASH_HANDLER`,
      `PSR_DISABLE_VOLUNTARY_PREEMPTION`, `PSR_AUDIO_{LEAD_MS,TARGET_MS,MAX_CORR,BRIDGE,WASAPI,NO_DECIMATE}`,
      `PSR_AI_SMOOTH`, `PSR_RT64_{FILTERING,UPSCALE2D,THREEPOINT}`, `PSR_ASPMAIN_*`,
      `PSR_RSP_DMA_TRACE`, `RECOMP_AUDIO_*`, `PSR_GB_FRAG_TRACE`, `PSR_GBTOWER_TRACE_CPU`.
      `PSR_VOLUME` / `set_volume` / `g_audio_volume` are documented in
      `include/debug_server.h:34-38` but **not implemented**. Add an env-var reference doc.
- [ ] **G-39 Dead code / leftovers:** `shaders/Interface{PS,VS}.hlsl` (unused since the
      RmlUi removal); `tools/disasm_fragment78_split.patch` (unreferenced);
      `pkmnstadium_cri_exit` (no hook, `extras.c:2134-2150`); `enable_instant_present`
      no-op (`debug_server.cpp:1048`); `screenshot` command documented but unhandled
      (`debug_server.cpp:16`); unused `kBootDmaSafeDmemOffset` and a "— wait" comment
      (`rsp_aspmain_hook.cpp:73-76`); unused DRC fields `em_low_ms/em_high_ms`
      (`recomp_audio_drc.h:49-50`).
- [ ] **G-40 Misc inconsistencies:** debug-server comment says port 4370, code uses 4371
      (`debug_server.cpp:5`); `project_version{0,1,0}` vs release v0.4.6
      (`main.cpp:2118`); `_package_release.ps1` / release notes tracked despite being
      gitignored; SlotMap "license" packaged from its README; mojibake / BOM in
      `aspmain_replay.cpp` and `game.toml:81`; `game.toml:25` claims CMake asserts the
      ROM MD5 but `CMakeLists.txt:58-65` only warns.
- [ ] **G-41 Fork identity.** README still points at `mstan/PokemonStadiumRecomp`
      releases/issues and the "no longer maintained" banner; decide this fork's status,
      issue tracker, and release channel.

## P5 — Feature backlog

- [ ] **G-42 Input:** per-player bindings (one shared kb + one shared pad table today,
      `input_bindings.cpp:420,441-467`); choose which pad goes to which port (currently
      enumeration order, `recompui_launcher.cpp:316-319`); controller hotplug (no
      `SDL_CONTROLLERDEVICEADDED/REMOVED` handling); configurable deadzone/sensitivity
      (hardcoded 8000, `main.cpp:1666-1682`); hotkey config (`config_path = nullptr`).
- [ ] **G-43 Launcher persistence:** volume, `audio_freq`, `enable_audio`,
      `window_scale` have defaults but are never saved/loaded
      (`recompui_launcher.cpp:169-173,253-302`); `pN_enabled` / `pN_tpak` are written but
      never read.
- [ ] **G-44 Real `Pak::TransferPak`** in N64ModernRuntime instead of the `RumblePak`
      stand-in (`main.cpp:1769-1776`); multi-port (P2–P4) Transfer Pak is unverified.
- [ ] **G-45 Ares oracle bridge** (`ares_bridge.cpp`, `n64_snapshot`, `verify_mode`,
      `watchdog`) and the Tier 1–4 reverse-debug roadmap (`DEBUG.md`) — none written.
- [ ] **G-46 Emulator fallback for unhandled instructions/branches** (`extras.c:11-13`).
- [ ] **G-47 Multi-instance fragment residency** (`plans/multi_instance_fragment_residency.md`,
      option B then A) — likely the shared fix for G-9/G-11/G-12.
- [ ] **G-48 Architectural backlog** from `ISSUES.md`: pre-decompressed ROM build step;
      per-site kseg1 ROM-read patches instead of `mirror_rom_to_kseg1`; indexed-deref
      step for the voice UAF chain language (`TEMP_PATCHES.md`).
- [ ] **G-49 Modding** — `MODDING.md` is a placeholder; no manifest/RecompModTool
      integration.
- [ ] **G-50 Texture packs** — subsystem removed (`rt64_render_context.cpp:5-7`); RT64
      gets a zeroed dummy ROM header (`:268`).
- [ ] **G-51 Code signing** for release binaries (README troubleshooting section).

---

## Suggested order

1. G-1 → G-4 (anyone can build from a clean clone), then G-6 CI.
2. G-18 (turn the debug server off in release), G-19 (OOB reads).
3. G-47 multi-instance residency → retest G-9, G-11, G-12, G-13, then settle G-14.
4. G-10 announcer static.
5. Docs sweep G-32 → G-41 (cheap, unblocks new contributors).
6. Portability (G-27 → G-31), then P5 features.
