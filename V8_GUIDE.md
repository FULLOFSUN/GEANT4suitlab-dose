# SuitDose v8: heavy ions (HZE) and per-element GCR spectra

v8 answers the review comments on v7:

1. **Heavy ions are now simulated explicitly.** GCR = six species groups, each
   run as its own Geant4 primary and added up afterwards.
2. **Helium has its own interstellar spectrum** instead of 0.18 x the proton
   rigidity spectrum.
3. **The Chang'e-4 "validation-gap" scaling is available as a cross-check**
   (`mission_dose.py --scale changE4`), so both routes the reviewer suggested
   can be reported side by side.

Geometry, materials, physics list, scoring and the SPE are unchanged from v7.

## GCR source definition (v8)

| Group | Charges included | Simulated as | Share of particles (solar min) |
|---|---|---|---|
| p  | Z = 1 | proton | 85 % |
| He | Z = 2 | alpha (4He) | 14 % |
| C  | Z = 3-7 (Li, Be, B, C, N) | 12C | 0.5 % |
| O  | Z = 8-10 (O, F, Ne) | 16O | 0.5 % |
| Si | Z = 11-20 (Na ... Ca) | 28Si | 0.2 % |
| Fe | Z = 21-28 (Sc ... Ni) | 56Fe | 0.08 % |

- **Spectra:** local interstellar spectrum (LIS) of every element Z = 1-28 from
  the Badhwar-O'Neill 2020 model (Slaba & Whitman 2020, NASA/TP-2019-220419,
  Table B1): f(T) = j0 * beta^delta * (T + m)^-gamma, T in GeV/n.
- **Absolute scale:** one constant for all elements, fixed so that BON2020
  hydrogen equals the Burger et al. (2000) proton LIS used (and validated) in
  v7 between 10 and 100 GeV/n. Element-to-element ratios are BON2020's.
- **Solar modulation:** force-field approximation (Gleeson & Axford 1968), each
  element with its own Z/A, phi = 400 MV (minimum) and 1100 MV (maximum). BON2020
  itself solves the Fokker-Planck equation; the force-field approximation is
  the usual simplification and is stated as such.
- **Groups:** each group's flux per nucleon is the sum of its members
  (particle number is conserved). Members heavier than the representative ion
  (e.g. Ca in the Si group) are slightly under-weighted in LET, lighter ones
  over-weighted.
- **Checks built in:** with T in GeV/n the spectra give C/O = 0.86, Si/O = 0.16,
  Fe/O = 0.13 at 1-10 GeV/n, and 85 % p, 14 % He, 1.3 % heavier by number,
  consistent with measured GCR composition.

**Check before the final runs:** confirm the Table B1 values and the sign of
delta for each element against your copy of the BON2020 paper. They are listed
at the top of `tools/make_spectra.py`. After any change, run
`python3 tools/make_spectra.py`, then `cmake` again so the build folder gets the
new macros.

## Cases per habitat variant

14 runs instead of 6:
- `gcr_min_p`, `gcr_min_He`, `gcr_min_C`, `gcr_min_O`, `gcr_min_Si`, `gcr_min_Fe`
- `gcr_max_p`, `gcr_max_He`, `gcr_max_C`, `gcr_max_O`, `gcr_max_Si`, `gcr_max_Fe`
- `spe_aug1972_lo`, `spe_aug1972_hi`

The v7 spectra (`gcr_min`, `gcr_min_alpha`, ...) are still in `macros/spectra`
for comparison, but the v8 scripts do not use them.

## How to run (from `suitdose_v8-build`)

Build:
```bash
cd ~/geant4/work
unzip -o ~/Downloads/suitdose_v8.zip
mkdir suitdose_v8-build && cd suitdose_v8-build
cmake ../suitdose_v8 -DCMAKE_PREFIX_PATH="$(brew --prefix qt);$(brew --prefix qt)/lib/cmake"
make -j8
```

Look at an iron nucleus going through the habitat (viewer):
```bash
./habitat --view macros/lunar_pole/geometry_baseline.mac
```
then, in the viewer:
```
/control/execute macros/habitat/source_isotropic.mac
/control/execute macros/spectra/gcr_min_Fe.mac
/run/beamOn 5
```

Quick test of the whole chain (minutes, numbers not for the report):
```bash
QUICK=1 bash ../suitdose_v8/tools/run_all.sh
```

Time one heavy-ion case before the full run (heavy ions are slower per event):
```bash
time N=100000 CASES="gcr_min_Fe" bash ../suitdose_v8/tools/run_lunar_pole.sh baseline
```

Full study (overnight or longer; see run times below):
```bash
caffeinate -i bash ../suitdose_v8/tools/run_all.sh
```

Results:
```bash
python3 ../suitdose_v8/tools/make_figures.py
open figures/*.png
python3 ../suitdose_v8/tools/mission_dose.py
```

## The two routes the reviewer asked for

| Route | Command | Meaning |
|---|---|---|
| Explicit HZE (main result) | `mission_dose.py` | GCR = p + He + C + O + Si + Fe, all simulated |
| Validation-gap scaling | `mission_dose.py --species p He --scale changE4` | p + He only, multiplied by k = H(Chang'e-4) / H(open surface, p + He) |
| Both, as a consistency check | `mission_dose.py --scale changE4` | with all six species k should be close to 1: this is the validation result |

`mission_dose.py` prints the validation line first, for example
`H = 1.2 mSv/d = 88 % of Chang'e-4`, and the share of the dose equivalent per
species for every variant.

## Run times

| Part | Runs | Note |
|---|---|---|
| 7 variants x 14 cases | 98 | heavy-ion runs are expected to be slower than proton runs: time one first |
| albedo (noground, deepground) | 12 | solar minimum only |
| validation (open_surface) | 6 | solar minimum only |
| wall scan, min + max | 336 | 4 materials x 7 thicknesses x 6 species x 2 |

The wall scan is the expensive part. Options:
- `N=30000` for the scan only;
- fewer points: `AREALS="1 5 20 70"`;
- fewer materials: `MATERIALS="G4_WATER G4_Al LunarRegolith"`;
- one step at a time: `STEPS="variants validation figures"` first, the scan later.

## New and changed outputs

| File | Content |
|---|---|
| `figures/table_4_1_validation.csv` | open surface vs Chang'e-4: D, H, Qbar, share of H per species |
| `figures/table_5_1_baseline.csv` | per species and total, solar min and max |
| `figures/fig_5_9_species.png` | share of the crew GCR dose equivalent per species (baseline) |
| `figures/fig_5_7_spe.png` | now on a log scale |
| `mission_dose_table.csv` | adds the species used, the scale factor k and per-species H |

All figures and tables can be made for a subset, e.g. `--species p He`, to
show the v7-style lower bound next to the full result.

## Options summary

- `run_lunar_pole.sh <variant>`: `N`, `THREADS`, `HORIZON`, `SPECIES="p He C O Si Fe"`,
  `SOLARS="min max"`, `NOSPE=1`, `CASES="..."`.
- `run_wall_scan.sh`: `N`, `SOLAR=min|max`, `SPECIES`, `MATERIALS`, `AREALS`.
- `run_all.sh`: `QUICK=1`, `STEPS="variants albedo validation scan_min scan_max figures"`, plus all of the above.
- `make_figures.py`: `--species`, `--files`.
- `mission_dose.py`: `--species`, `--scale 1|<k>|changE4`, `--days`, `--spe`, `--limit`.

## Remaining limitations (state in the report)

- Charge groups represented by one ion each; Z > 28 neglected (very low abundance).
- Force-field modulation, not the BON2020 transport solution; phi values are typical, not date-specific.
- BON2020 absolute scale tied to the Burger proton LIS at high energy.
- Placeholder habitat; whole-body water phantom; one reference SPE (protons only).
