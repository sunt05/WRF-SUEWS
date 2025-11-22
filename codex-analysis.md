# Dependency map: how SUEWS is wired into the coupled WRF (as built in `compilation-20251121`)

## Build-time linkage
- `coupling-automator` writes `wrf_suews.mk` into `SUEWS/lib/` and `configure.wrf` includes it. That injects:
  - `SUEWS_INC_DIR = ../SUEWS/include`
  - `SUEWS_LIB_DIR = ../SUEWS/lib`
  - compiler/link flags: `-I$(SUEWS_INC_DIR)` and `-L$(SUEWS_LIB_DIR) -lsuews`
- `phys/Makefile` inherits these flags, so all EM builds (`wrf.exe`, `real.exe`, `ideal.exe`, `ndown.exe`, `tc.exe`) link against the prebuilt static library `SUEWS/lib/libsuews.a` from the submodule.
- SUEWS was **not** rebuilt during the successful WRF build; we used the existing `libsuews.a` as-is.

## WRF-side plumbing
- SUEWS state is declared in `Registry/registry.suews` and included via `Registry/Registry` → generated `inc/*` gives domain members: `landusef_SUEWS`, `MaxConductance_SUEWS`, etc.
- `phys/module_surface_driver.F` selects `SUEWSSCHEME` (sf_surface_physics=9) and dispatches to `phys/module_sf_suews.F`.
- `phys/module_sf_suews.F` contains the WRF↔SUEWS bridge:
  - `suewsinit` initializes SUEWS (reads tables/namelist, sets up state, bcasts constants).
  - `suewsdrv` is called each timestep to compute surface fluxes; it marshals WRF fields into the SUEWS call and writes back outputs (`z0m_SUEWS`, `AH_SUEWS`, etc.).
- The interface block in `module_sf_suews.F` expects symbols exported by `libsuews.a` (e.g., `suews_ctrl_sumin`, driver routines). If the library is missing these, link would fail.

## SUEWS library symbols in use
- `libsuews.a` (from submodule) exports the SUEWS driver and data modules; selected symbols (from `nm`):
  - Module copies for all SUEWS data types: `___suews_def_dts_MOD___copy_suews_def_dts_*` (Anthroemis, Conductance_prm, Output_line, etc.).
  - Main driver symbols (not fully expanded here) are the entry points `suews_ctrl_sumin`/`suews_ctrl_driver` called via the interface in `module_sf_suews.F`.
- We did not rebuild or alter `SUEWS/src/suews/src/suews_ctrl_sumin.f95`; the archive is whatever the submodule shipped.

## Runtime call chain (as currently failing)
1. WRF starts; `module_physics_init` calls `suewsinit` (in `module_sf_suews`) → `suewsinit` opens SUEWS text tables (e.g., `LANDUSE.TBL`, `SUEWS_*.txt`) and sets up SUEWS state.
2. During surface physics, `module_sf_suews` calls `suewsdrv`, which invokes the SUEWS library (from `libsuews.a`, ultimately `suews_ctrl_sumin`/`suews_ctrl_driver`).
3. Current fatal: inside the SUEWS library, conductance lookup yields `gsModel=0` (CondCode not found/parsed) → `ErrorHint` -> fatal “Check input file SUEWS_Conductance.txt.” This is an input-path/format mismatch, not a link issue.

## What we tried outside the submodules
- A scratch `suews-patched-build/` was started to add a gsModel fallback in `suews_phys_resist.f95`, compiled with `-Dwrf`, but the build fails in `suews_ctrl_sumin.f95` (type mismatch). This patched lib is **not** in use; the live build still uses the unmodified `libsuews.a`.

## Action options
1. **Input-only fix**: Provide matching `SUEWS_Conductance.txt`/`SUEWS_SiteSelect.txt` (codes and headers aligned) so runtime table reads succeed.
2. **Runtime bypass (recommended for “prebaked inputs” workflow)**: Patch `module_sf_suews.F`/SUEWS init to skip reading text tables when the fields are already in `wrfinput`; carry this patch in coupling-automator so submodules stay clean.
3. **Defensive library patch**: Finish patched `libsuews.a` (gsModel fallback to 1 on invalid code) and relink WRF.

## Open issue
- Runtime still aborts on `gsModel=0` → Conductance table fatal. Need to resolve via one of the above paths.
