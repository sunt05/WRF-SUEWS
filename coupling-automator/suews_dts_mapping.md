# SUEWS DTS mapping (for WRF wrapper rewrite)

This note summarises the new `SUEWS_cal_Main` DTS interface and how the current `module_sf_suews.F` data should map into the derived types.

## `SUEWS_cal_Main` signature
- `SUEWS_cal_Main(timer, forcing, config, siteInfo, modState, outputLine, debugState)`
- Intents: `timer` (IN), `forcing` (INOUT), `config` (IN), `siteInfo` (IN), `modState` (INOUT), `outputLine` (OUT), `debugState` (OPTIONAL OUT).

## DTS quick reference

**SUEWS_TIMER**: `id` (doy), `imin`, `isec`, `it` (hour), `iy`, `tstep`, `tstep_prev`, `dt_since_start`, `dt_since_start_prev`, derived: `nsh`, `nsh_real`, `tstep_real`, `dectime`, `dayofWeek_id(3)`, `DLS`, `new_day`.

**SUEWS_FORCING**: `kdown`, `ldown`, `RH`, `pres`, `Tair_av_5d`, `U`, `rain`, `Wu_m3`, `fcld`, `LAI_obs`, `snowfrac`, `xsmd`, `qf_obs`, `qn1_obs`, `qs_obs`, `temp_c`, optional `Ts5mindata_ir`.

**SUEWS_CONFIG**: physics switches: `DiagMethod`, `EmissionsMethod`, `RoughLenHeatMethod`, `RoughLenMomMethod`, `FAIMethod`, `SMDMethod`, `WaterUseMethod`, `NetRadiationMethod`, `StabilityMethod`, `StorageHeatMethod`, `Diagnose`, `SnowUse`, `use_sw_direct_albedo`, `ohmIncQF`, `DiagQS`, `EvapMethod`, `LAImethod`, `localClimateMethod`, `stebbsmethod`, `flag_test`.

**SUEWS_SITE (selected fields)**  
- Core: `lat`, `lon`, `alt`, `gridiv`, `timezone`, `surfacearea`, `z`, `z0m_in`, `zdm_in`, `pipecapacity`, `runofftowater`, `narp_trans_site`, `flowchange`, `n_buildings`, `h_std`, `lambda_c`, `sfr_surf(:)`, `VegFraction`/`ImpervFraction`/`PervFraction`/`NonWaterFraction`.  
- Land-cover parameter blocks (used per surface): `lc_paved`, `lc_bldg`, `lc_dectr`, `lc_evetr`, `lc_grass`, `lc_bsoil`, `lc_water` each carrying `sfr`, `emis`, `ohm%ohm_coef_lc`, `soil%{soildepth,soilstorecap,sathydraulicconduct}`, `state`, `statelimit`, `wetthresh`, irrigation fractions, `waterdist%to_*`. Vegetated classes also hold `lai%{baset,gddfull,basete,sddfull,laimin,laimax,laipower,laitype}` and `maxconductance` or `conductance` parameters, biogenic CO₂ parameters, FAI/height/porosity/albedo fields.  
- Other parameter blocks: `snow` (`crwmax/min`, `preciplimit`, `preciplimitalb`, `snowalbmax/min`, `snowdensmax/min`, `snowlimbldg/paved`, `snowpacklimit(:)`, `snowprof_24hr_*`, `tau_a/f/r`, `tempmeltfact`, `radmeltfact`), `conductance` (`g_max`, `g_k`, `g_q_base`, `g_q_shape`, `g_t`, `g_sm`, `kmax`, `gsmodel`, `s1`, `s2`, `TH`, `TL`), `lumps` (`raincover`, `rainmaxres`, `drainrt`, `veg_type`), `irrigation` (`faut`, `ie_a`, `ie_m`, `ie_start/end`, `internalwateruse_h`, `irr_daywater` flags/percents, hourly `wuprofa/m_24hr`), `anthroemis` (daylight saving, `anthroheat` profiles/thresholds, `TrafficRate_*`, `TrafficUnits`, `HumActivity_*`, emission factors).

**SUEWS_STATE (components)**  
- `flagState` (convergence flags), `anthroemisState` (HDD cache + CO₂ flux components), `ohmState` (qn/dqndt accumulators), `solarState`, `atmState` (diagnostic met), `phenState` (LAI/GDD/SDD, albedos, `StoreDrainPrm(6,nsurf)`, porosity/canopy storage, conductance factors), `snowState` (snowpack/roughness), `hydroState` (soil stores, wetness, `WUDay_id(9)`, runoff pieces, smd, wu, etc), `heatState` (surface temps and fluxes incl. qs/qh/qe/qf/qsfc), `roughnessState` (FAI/PAI/z0/zdm), `stebbsState`, `nhoodState`.

**output_line**: concatenated arrays with datetime + outputs. `dataOutLineSUEWS` (85 values) is the main one; key indices:
- 1:kdown, 2:kup, 3:ldown, 4:lup, 5:tsurf, 6:qn, 7:qf, 8:qs, 9:qh, 10:qe, 11:QH_LUMPS, 12:QE_LUMPS, 13:QH_init, 14:qh_resist, 15:rain, 16:wu_ext, 17:ev_per_tstep, 18:runoff_per_tstep, 19:tot_chang_per_tstep, 20:surf_chang_per_tstep, 21:state_per_tstep, 22:NWstate_per_tstep, 23:drain_per_tstep, 24:smd, 25:FlowChange/nsh, 26:AdditionalWater, 27:runoffSoil_per_tstep, 28:runoffPipes, 29:runoffAGimpervious, 30:runoffAGveg, 31:runoffWaterBody, 32:wu_int, 33:wu_EveTr, 34:wu_DecTr, 35:wu_Grass, 36-41:smd_surf (paved…water), 42-48:state_surf, 49:zenith, 50:azimuth, 51:bulkalbedo, 52:Fcld, 53:LAI_wt, 54:z0m, 55:zdm, 56:zL, 57:UStar, 58:TStar, 59:l_mod, 60:RA, 61:RS, 62:Fc, 63:Fc_photo, 64:Fc_respi, 65:Fc_metab, 66:Fc_traff, 67:Fc_build, 68:Fc_point, 69:qn_snowfree, 70:qn_snow, 71:SnowAlb, 72:Qm, 73:QmFreez, 74:QmRain, 75:swe, 76:mwh, 77:MwStore, 78:chSnow_per_interval, 79-80:SnowRemoval, 81:tsfc_C, 82:t2_C, 83:q2_gkg, 84:avU10_ms, 85:RH2_pct.

## WRF → DTS mapping (current wrapper)

**Timer (SUEWS_TIMER)**  
- `iy/id/it/imin/isec` from driver args `year/day/hour/minute/second`.  
- `tstep` = `INT(DT)`, `tstep_prev` = `INT(DT_PREV)`.  
- `dt_since_start` = `INT(xtime*60)`.  
- `DLS`/`new_day` unused; `dayofWeek_id` left to SUEWS internals.

**Forcing (SUEWS_FORCING)**  
- `kdown` ← `SWDOWN1D` (may be overwritten by transmissivity correction vs `SWDNTC`).  
- `ldown` ← `GLW1D`.  
- `temp_c` ← `T1D-273.15`; `RH` ← `q2rh(QV1D,T1D,Press_hPa)*100` (clamped ≥5%).  
- `pres` ← `PSFC/100.` (hPa).  
- `U` ← `sqrt(U1D^2+V1D^2)`.  
- `rain` ← `PREC1D`.  
- `fcld` ← column-mean `cldfra`.  
- Fields not currently supplied (remain 0): `Tair_av_5d`, `Wu_m3`, `LAI_obs`, `snowfrac`, `xsmd`, `qf_obs`, `qn1_obs`, `qs_obs`, `Ts5mindata_ir`.

**Config (SUEWS_CONFIG)** – all read from `namelist.suews` `&method` or defaults:  
- `SnowUse`, `EmissionsMethod`, `NetRadiationMethod`, `RoughLenHeatMethod`, `RoughLenMomMethod`, `StorageHeatMethod`, `ohmIncQF`, `LAImethod`(fixed 1), `EvapMethod`(fixed 2), `DiagMethod`/`Diagnose`(from namelist), `FAIMethod`/`SMDMethod`/`WaterUseMethod`/`StabilityMethod`/`localClimateMethod`/`stebbsmethod` left at defaults unless added.

**Site basics (SUEWS_SITE)**  
- Geo/time: `lat/lon` ← `XLAT/XLONG`; `alt` ← `ht`; `surfacearea` ← `DX*DX`; `z` ← `dz8w`; `timezone` ← `timezone_SUEWS`; `gridiv` fixed 1.  
- Roughness inputs: `z0m_in`, `zdm_in` from registry fields `z0m_in_SUEWS`, `zdm_in_SUEWS`.  
- Fractions: `sfr_surf(:)` from `landusef_SUEWS` (via `toSUEWScat`, with paved/building split adjusted by `paved_ratio`).  
- Flow plumbing: `pipecapacity` ← `PipeCapacity`, `runofftowater` ← `RunoffToWater`, `flowchange` ← `FlowChange`.

**Land-cover parameters → `lc_*` blocks**  
- OHM coefficients: `OHM_coef_s(8,4,3)` → `ohm%ohm_coef_lc` per surface (snow row included in source array).  
- Surface drainage: `surf_attr_*` + `surf_var` build `StoreDrainPrm(6,nsurf)` (phenology state) used for `state_limit`, drainage eq/coeffs, current storage cap.  
- Water redistribution: `WaterDist_s(8,6)` → each surface’s `waterdist%to_*`.  
- LUMPS: `RAINCOVER`, `RAINMAXRES`, `DRAINRT`, `veg_type` → `lumps` members.  
- Soil: `SoilDepth`, `SoilStoreCap`, `SatHydraulicConduct` → `soil%soildepth/soilstorecap/sathydraulicconduct` per surface.  
- Albedo/min/max: `AlbMin/Max_DecTr/EveTr/Grass` → `lc_dectr%alb_min/max`, `lc_evetr%alb_min/max`, `lc_grass%alb_min/max`.  
- Canopy storage/porosity: `CapMin_dec/CapMax_dec/PorMin_dec/PorMax_dec` → `lc_dectr%capmin_dec/capmax_dec/pormin_dec/pormax_dec`.  
- State limits/wetness thresholds: `StateLimit(:)` → `statelimit`; `WetThresh(:)` → `wetthresh` for each surface.  
- Morphology/FAI: `FAIbldg`, `bldgH` → `lc_bldg%faibldg/bldgh`; `FAIEveTree`, `EveTreeH` → `lc_evetr%faievetree/evetreeh`; `FAIDecTree`, `DecTreeH` → `lc_dectr%faidectree/dectreeh`.  
- Conductance/stomatal params: `g1..g6` + `Kmax`, `th`, `tl`, `s1`, `s2` map to `conductance` (g_max/g_k/g_q_base/g_q_shape/g_t/g_sm, kmax, TH/TL, s1/s2, gsmodel from conductance table).  
- Vegetation phenology: `BaseT/BaseTe/GDDFull/SDDFull/LaiMin/LaiMax/LAIType/LaiPower` per veg surface → `lai%baset/basete/gddfull/sddfull/laimin/laimax/laipower/laitype`.  
- Max canopy conductance: `MaxConductance` → `lc_evetr/maxconductance`, `lc_dectr/maxconductance`, `lc_grass/maxconductance`.  
- Snow params: `CRWmax/CRWmin/PrecipLimit/PrecipLimitAlb/SnowAlbMax/Min/SnowDensMax/Min/SnowLimBldg/Paved/tau_a/tau_f/tau_r/TempMeltFact/RadMeltFact/SnowPackLimit(:)/snowProf_24hr` → `snow` block.  
- Irrigation: `Faut`, `IrrFracConif/Decid/Grass`, `DayWat`, `DayWatPer` → `irrigation%faut`, `irr_daywater` flags/percents, per-surface irrigation fractions; `Wu_m3` forcing is currently unset.  
- Anthropogenic heat/CO₂: `NumCapita`, `PopDensDaytime/Nighttime`, `AH_MIN`, `AH_SLOPE_*`, `QF0_BEU`, `Qf_A/B/C`, `T_CRITIC_*`, `TrafficRate`, `BaseTHDD` → `anthroemis%anthroheat` (working/holiday profiles), `TrafficRate_*`, population profiles, `startDLS/endDLS`. Emission factors kept at defaults from library unless populated.

**State persistence (SUEWS_STATE)**  
- `qn1_av/qd/dqnsdt` → `ohmState%qn_av/dqndt/dqnsdt`; `qn1_s` → `ohmState%qn_s_av`.  
- `state_SUEWS` → `hydroState%state_surf`; `soilmoist_SUEWS` → `hydroState%soilstore_surf`; `surf_var_SUEWS` → `phenState%StoreDrainPrm(6,:)` (current storage cap).  
- `WUDay_SUEWS(9)` → `hydroState%WUDay_id`; `MeltWaterStore_SUEWS` → `snowState%snowwater`; `SnowAlb_SUEWS` → `snowState%snowalb`; `smd_SUEWS` field currently pulled from output index 24.  
- `GDD_SUEWS` → `phenState%GDD_id`; `HDD_SUEWS` → `anthroEmis_STATE%HDD_id`.  
- `LAI_SUEWS`/`alb*_SUEWS`/`porosity_SUEWS`/`DecidCap_SUEWS` → `phenState%lai_id/alb*/porosity_id/decidcap_id`.  
- `z0m_in/zdm_in/g1..g6` stored per-tile but not yet in `SUEWS_STATE`; new design should keep them in `SUEWS_SITE`/`conductance`.

**Outputs currently consumed by WRF**  
- `qn/qf/qs/qh/qe` ← `dataOutLineSUEWS(6:10)`  
- `z0m_SUEWS`/`ZNT` ← `54`; `UST` calculated in wrapper (not from DTS).  
- `GRDFLX` ← `qs`; `HFX` ← `qh`; `LH` ← `qe`; `QFX` ← `qsfc` (`dataOutLineSUEWS(16)` in legacy code – note this is actually `wu_ext`, not a flux).  
- Radiation diagnostics: kdown/kup/ldown/lup/tsurf from indices 1–5.  
- Soil moisture deficit: `smd_SUEWS` currently uses index 24 (`smd`).  
- `tsk` uses index 81 (tsfc_C + 273.15).

## Known gaps to resolve during rewrite
- Many forcing fields (`Tair_av_5d`, `Wu_m3`, `LAI_obs`, `snowfrac`, `xsmd`, observed qn/qs/qf) are never populated—decide defaults or derive from WRF state.  
- `SUEWS_CONFIG` options like `FAIMethod`, `SMDMethod`, `WaterUseMethod`, `StabilityMethod`, `localClimateMethod`, `stebbsmethod` are not driven by WRF inputs.  
- Conductance parameter mapping (g1–g6 → `CONDUCTANCE_PRM` slots) should be confirmed against SUEWS expectations.  
- Output mapping in legacy wrapper misuses some indices (e.g., `qsfc` from `wu_ext`, `smd` vs `drain_per_tstep` if reordered); verify against new DTS before wiring back to WRF fluxes.  
- Persistent state should live in `SUEWS_STATE`; current scatter of registry arrays will need consolidation or explicit pack/unpack.

## Direct mapping hints (WRF vars → DTS fields)

**Forcing**  
- `SWDOWN` → `forcing%kdown`; `GLW` → `forcing%ldown`.  
- `T3D(i,1,j)` → `temp_c`; `PSFC` → `pres` (hPa); `QV3D(i,1,j)` with `T1D/Press` → `RH`.  
- `U3D/V3D` → `U`; `PREC` → `rain`; `cldfra` column mean → `fcld`.  
- Leave `Tair_av_5d`, `Wu_m3`, `LAI_obs`, `snowfrac`, `xsmd`, `qf_obs/qn1_obs/qs_obs` zero unless derived later.

**Config** (from namelist.suews `&method`)  
- `SnowUse`, `EmissionsMethod`, `NetRadiationMethod`, `RoughLenHeatMethod`, `RoughLenMomMethod`, `StorageHeatMethod`, `OHMIncQF`, `LAIType`/`LaiPower` (→ `LAImethod=1`), diagnostics flags.  
- Set fixed library defaults used previously: `EvapMethod=2`, `LAImethod=1`, `DiagQS=0`, `flag_test=.FALSE.` unless debugging.

**Site geometry/roughness**  
- `lat/lon/alt` from `XLAT/XLONG/ht`; `surfacearea=DX*DX`; `z=dz8w`; `timezone=timezone_SUEWS`; `gridiv=1`.  
- `z0m_in/zdm_in` from registry fields `z0m_in_SUEWS/zdm_in_SUEWS`.  
- `sfr_surf` from `landusef_SUEWS(i,:,j)` (after `paved_ratio` adjustment).  
- `flowchange`, `pipecapacity`, `runofftowater` from respective registry arrays.

**Land-cover params → `lc_*` blocks**  
- OHM coeffs: `OHM_coef_s(8,4,3)` → `ohm%ohm_coef_lc(:)%{summer_dry,summer_wet,winter_dry,winter_wet}` per surface (rows 1–7).  
- Water redistribution: `WaterDist_s(8,6)` row 1–7 → `waterdist%to_{paved,bldg,evetr,dectr,grass,bsoil}`; use row 8 (snow) only if needed.  
- Soil: `SoilDepth/SoilStoreCap/SatHydraulicConduct` → `soil%{soildepth,soilstorecap,sathydraulicconduct}` per surface.  
- Wetness limits: `StateLimit` → `statelimit`; `WetThresh` → `wetthresh`; `surf_attr_Min/MaxStorCap` → `surf_store%store_min/store_max`; `surf_attr_DrainEquat/DrainCoef1/DrainCoef2` → `surf_store%drain_eq/drain_coef_*`.  
- Albedo/min/max: `AlbMin/Max_*` to veg surfaces; emissivity from `emis_SUEWS`.  
- Morphology/FAI: `FAIbldg/bldgH` → `lc_bldg`; `FAIEveTree/EveTreeH` → `lc_evetr`; `FAIDecTree/DecTreeH` → `lc_dectr`.  
- Conductance: map `g1..g6`, `Kmax`, `th/tl`, `s1/s2` into `conductance%g_*`, `kmax`, `TH/TL`, `s1/s2`; set `gsmodel` non-zero (e.g., 1) to avoid fatal.  
- Phenology: `BaseT/BaseTe/GDDFull/SDDFull/LaiMin/LaiMax/LaiPower/LAIType` per veg surface → `lai` fields.  
- Max canopy conductance: `MaxConductance` per veg surface → `maxconductance` (evetr/dectr/grass).  
- Snow: `CRWmax/CRWmin/PrecipLimit/PrecipLimitAlb/SnowAlbMax/Min/SnowDensMax/Min/SnowLimBldg/Paved/tau_a/tau_f/tau_r/TempMeltFact/RadMeltFact/SnowPackLimit(:)/snowProf_24hr` → `snow` block.  
- Irrigation: `Faut`, `IrrFrac*`, `DayWat`, `DayWatPer`, `internalwateruse_h` zero unless provided, `ie_a/ie_m` constants from SuMin remain reasonable defaults.  
- Anthro heat/CO₂: `NumCapita`, `PopDensDaytime/Nighttime`, `AH_MIN`, `AH_SLOPE_*`, `QF0_BEU`, `Qf_A/B/C`, `T_CRITIC_*`, `TrafficRate`, `BaseTHDD`, `startDLS/endDLS` → `anthroemis%anthroheat` + `anthroemis%startdls/enddls`.

**State → `SUEWS_STATE`**  
- `GDD/HDD` → `phenState%GDD_id`, `anthroEmis_STATE%HDD_id`.  
- `LAI/alb*/porosity/DecidCap` → `phenState` members; `StoreDrainPrm(6,:)` ← `surf_var` current storage.  
- `state/soilmoist` → `hydroState%state_surf/soilstore_surf`; `WUDay` → `hydroState%WUDay_id`; `MeltWaterStore` → `snowState%snowwater`; `SnowAlb` → `snowState%snowalb`.  
- `qn1_av/dqndt/qns/dqnsdt` → `ohmState%qn_av/dqndt/qn_s_av/dqnsdt`.
