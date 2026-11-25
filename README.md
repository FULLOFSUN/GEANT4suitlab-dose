# GEANT4suitlab-dose

**Crew radiation dose inside a lunar south pole habitat — a Geant4 Monte Carlo study**

Does a crew in a simple surface habitat at the lunar south pole stay below the
ESA career dose limit of 1 Sv during a 100-day or one-year stay, at solar
minimum and at solar maximum? And if not, what shielding is needed?

This repository contains **SuitDose v8**, the Geant4 application, input spectra,
geometry macros, run scripts and analysis scripts used to answer that question
during a UCD Erasmus+ traineeship at the Austrian Space Forum (OeWF).

![Workflow of the study](docs/workflow_v8.png)

---

## Key results

| | Solar minimum | Solar maximum |
|---|---|---|
| Crew dose-equivalent rate, baseline 4 mm Al hull | 0.94 mSv/day | 0.40 mSv/day |
| Share of dose equivalent from heavy ions (C, O, Si, Fe groups) | 63 % | 61 % |
| Reduction with 10–13 g/cm² of water or polyethylene | −34 to −37 % | −28 to −29 % |

| Mission (worst case) | Baseline hull | Shielded variants |
|---|---|---|
| 100 days | 1249 mSv (fails: one SPE) | 54–71 mSv |
| 1 year | 1355 mSv (fails: one SPE) | 197–234 mSv (20–23 % of 1 Sv) |

- **Validation:** on the open lunar surface the model gives 1.00 mSv/day, 73 %
  of the Chang'e-4 LND measurement (1.37 mSv/day), with a mean quality factor of
  4.13 against the measured 4.3.
- **Heavy ions:** about 1 % of the GCR particles, but over 60 % of the dose
  equivalent; the Fe group alone, 0.08 % of the particles, gives about 30 %.
- **Solar particle event:** one August 1972-type event gives 1.21 Sv behind the
  bare hull and 27–43 mSv behind 10–16 g/cm² of extra wall. A storm shelter
  of about 10–15 g/cm² of water or polyethylene is the key requirement.

Results are statistical-uncertainty only (1σ); see [Limitations](#limitations).

---

## Contents

- [What is simulated](#what-is-simulated)
- [Repository structure](#repository-structure)
- [Requirements](#requirements)
- [Build](#build)
- [Quick start](#quick-start)
- [Running the full study](#running-the-full-study)
- [Outputs](#outputs)
- [Options](#options)
- [Run times](#run-times)
- [Validation and the heavy-ion question](#validation-and-the-heavy-ion-question)
- [Limitations](#limitations)
- [Version history](#version-history)
- [How to cite](#how-to-cite)
- [References](#references)
- [Acknowledgements](#acknowledgements)

---

## What is simulated

### Habitat

- Horizontal cylinder, 1 m inner radius, 3 m long, lying on lunar regolith.
- Baseline wall: 4 mm Al-2219 hull (2.84 g/cm³, 1.14 g/cm²); cabin filled with air.
- Crew member: water cylinder, 15 cm radius, 1.70 m long (about 120 kg), lying
  along the habitat axis (a resting or sleeping astronaut). The dose averaged
  over this phantom is the whole-body crew dose.
- Ground: lunar regolith, 1.5 g/cm³, average oxide composition
  (45 % SiO₂, 17 % Al₂O₃, 12 % FeO, 12 % CaO, 10 % MgO, 3 % TiO₂, 1 % Na₂O).

| Variant | Wall, outside to inside | Total wall (g/cm²) |
|---|---|---|
| `baseline` | 4 mm Al-2219 | 1.1 |
| `water10` | hull + 0.5 cm PE + 10 cm water + 0.5 cm PE | 12.1 |
| `water10_pe1cm` | hull + 1 cm PE + 10 cm water + 1 cm PE | 13.0 |
| `pe10` | hull + 10 cm polyethylene | 10.5 |
| `regolith10cm` | 10 cm regolith + hull | 16.1 |
| `regolith25cm` | 25 cm regolith + hull | 38.6 |
| `regolith1m` | 1 m regolith + hull | 151.1 |
| `noground` | baseline, no regolith (albedo study) | 1.1 |
| `deepground` | baseline on 1.5 m of regolith (albedo study) | 1.1 |
| `open_surface` | 0.1 mm water film (validation against Chang'e-4) | 0.01 |

### Radiation environment

**Galactic cosmic rays (GCR):** six species groups, each simulated as its own
primary particle and added up afterwards.

| Group | Charges included | Simulated as | Share of particles (solar min) |
|---|---|---|---|
| p  | Z = 1 | proton | 85 % |
| He | Z = 2 | alpha (⁴He) | 14 % |
| C  | Z = 3–7 (Li, Be, B, C, N) | ¹²C | 0.5 % |
| O  | Z = 8–10 (O, F, Ne) | ¹⁶O | 0.5 % |
| Si | Z = 11–20 (Na … Ca) | ²⁸Si | 0.2 % |
| Fe | Z = 21–28 (Sc … Ni) | ⁵⁶Fe | 0.08 % |

- **Spectra:** local interstellar spectrum (LIS) of every element Z = 1–28 from
  the Badhwar–O'Neill 2020 model (BON2020, Table B1):
  f(T) = j₀ β^δ (T + m)^−γ, with T in GeV/n.
- **Absolute scale:** one constant for all elements, chosen so that BON2020
  hydrogen equals the Burger et al. (2000) proton LIS between 10 and
  100 GeV/n. Element-to-element ratios are BON2020's.
- **Solar modulation:** force-field approximation (Gleeson & Axford 1968), each
  element with its own Z/A; φ = 400 MV (solar minimum) and 1100 MV (solar
  maximum). BON2020 itself solves the Fokker–Planck equation; the force-field
  approximation is the usual simplification.
- **Groups:** each group's flux per nucleon is the sum of its members, so the
  number of particles is conserved. Members heavier than the representative ion
  are slightly under-weighted in LET, lighter ones over-weighted.
- **Built-in checks:** C/O ≈ 0.86, Si/O ≈ 0.16 and Fe/O ≈ 0.13 at 1–10 GeV/n,
  and 85 % p, 14 % He, 1.3 % heavier ions by number, consistent with measured
  GCR composition.

**Solar particle event (SPE):** the August 1972 event, King (1974) fit
F(>E) = 7.9×10⁹ exp[(30 − E)/26.5] cm⁻², 10–1000 MeV protons. It is simulated in
two energy bands (10–100 and 100–1000 MeV) whose doses are added, so the
thick-wall variants get usable statistics.

**Source geometry:** isotropic (cosine-law) source on a sphere enclosing the
habitat; the simulated fluence is Φ = N / (π R²). The regolith fills the lower
part of the sphere and upward primaries are discarded, so the habitat sees the
upper half-sky plus albedo from the ground.

### Physics and scoring

- Geant4 11.4.2, multithreaded, reference physics list **QGSP_BIC_HP**
  (Binary Cascade for hadron and ion interactions, high-precision neutron
  transport below 20 MeV).
- **Absorbed dose D** in the phantom and in each wall layer.
- **Dose equivalent H** step by step with the ICRP 60 quality factor Q(L), using
  the unrestricted LET in water at the mid-point of each step. Q = 1 for photons,
  neutrons and electrons (neutron dose is scored through their charged
  secondaries).
- **Nuclear share:** the fraction of the phantom dose carried by particles that
  descend from a hadronic interaction (spallation products, fragments, neutrons,
  albedo).
- **Mission dose:** H = Σ(species) Ḣ × days + N_SPE × H_SPE, compared with
  the ESA 1 Sv career limit.

---

## Repository structure

```
GEANT4suitlab-dose/
├── CMakeLists.txt            builds two programs: habitat and suitdose
├── habitat.cc                main program used for this study
├── suitdose.cc               earlier slab model of a spacesuit (training only)
├── include/, src/            geometry, materials, physics, Q(LET) scoring, nuclear tagging
├── macros/
│   ├── lunar_pole/           habitat variants: geometry_<variant>.mac
│   ├── spectra/              GCR (6 species × min/active/max) and SPE spectra
│   └── habitat/              viewer (vis.mac), isotropic source, training exercises
├── tools/
│   ├── run_all.sh            the whole study in one command
│   ├── run_lunar_pole.sh     one habitat variant through all cases
│   ├── run_wall_scan.sh      shielding curve: materials × thicknesses
│   ├── make_spectra.py       regenerates the spectrum macros (BON2020, King 1974)
│   ├── make_figures.py       report figures and tables -> figures/
│   ├── mission_dose.py       mission dose table -> mission_dose_table.csv
│   └── gcrlib.py             shared helpers for the two analysis scripts
├── docs/                     workflow diagram and result figures for this README
├── V8_GUIDE.md               what changed in v8 and why
├── LUNAR_POLE.md             notes for the lunar south pole study
└── TRAINING.md               Geant4 habitat exercises
```

---

## Requirements

| Tool | Version used | Notes |
|---|---|---|
| Geant4 | 11.4.2, multithreaded, with data sets | build with Qt for the viewer |
| CMake | ≥ 3.16 | |
| C++ compiler | C++17 | Xcode Command Line Tools on macOS |
| Qt | 6 | optional, only for the 3D viewer |
| Python | ≥ 3.9 | with `matplotlib` |

On macOS:

```bash
xcode-select --install
brew install cmake qt python
pip3 install matplotlib
```

Geant4 itself is built from source following the
[Geant4 installation guide](https://geant4.web.cern.ch/download/). Make sure its
environment is loaded before building, for example:

```bash
source ~/geant4/geant4-v11.4.2-install/bin/geant4.sh
```

---

## Build

```bash
git clone https://github.com/FULLOFSUN/GEANT4suitlab-dose.git
mkdir GEANT4suitlab-dose-build && cd GEANT4suitlab-dose-build
cmake ../GEANT4suitlab-dose -DCMAKE_PREFIX_PATH="$(brew --prefix qt);$(brew --prefix qt)/lib/cmake"
make -j8
```

If CMake cannot find Geant4, add
`-DGeant4_DIR=$HOME/geant4/geant4-v11.4.2-install/lib/cmake/Geant4`.

All macros are copied into the build folder at configure time. **Run every
command below from the build folder.** After editing or regenerating macros,
run `cmake` again.

---

## Quick start

**Look at the geometry (opens the 3D viewer):**

```bash
./habitat --view macros/lunar_pole/geometry_baseline.mac
```

In the viewer, fire a few iron nuclei:

```
/control/execute macros/habitat/source_isotropic.mac
/control/execute macros/spectra/gcr_min_Fe.mac
/run/beamOn 5
```

Viewer runs are for looking only: with a handful of particles the statistical
error is close to 100 %.

**Test the whole chain in a few minutes** (numbers not for use):

```bash
QUICK=1 bash ../GEANT4suitlab-dose/tools/run_all.sh
open figures/*.png
```

---

## Running the full study

Time one heavy-ion case first; heavy ions are slower per event:

```bash
time N=100000 CASES="gcr_min_Fe" bash ../GEANT4suitlab-dose/tools/run_lunar_pole.sh baseline
```

Then run everything (on macOS, `caffeinate -i` keeps the computer awake):

```bash
caffeinate -i bash ../GEANT4suitlab-dose/tools/run_all.sh
```

`run_all.sh` runs these steps in order; each can also be run on its own with
`STEPS="..."`:

| Step | What it runs | Gives |
|---|---|---|
| `variants` | 7 habitat variants × 14 cases | tables 5.1, 5.8; figures 5.2, 5.7, 5.9 |
| `albedo` | `noground`, `deepground`, solar minimum | table 5.5 |
| `validation` | `open_surface`, solar minimum | table 4.1 |
| `scan_min`, `scan_max` | wall scan, solar minimum and maximum | figures 5.3, 5.4 |
| `figures` | `make_figures.py` and `mission_dose.py` | all figures and tables |

Examples:

```bash
# the runs the mission table needs, first
caffeinate -i env STEPS="variants validation albedo figures" bash ../GEANT4suitlab-dose/tools/run_all.sh

# a lighter wall scan afterwards
caffeinate -i env N=30000 AREALS="1 5 20 70" MATERIALS="G4_WATER G4_Al LunarRegolith" \
  STEPS="scan_min scan_max figures" bash ../GEANT4suitlab-dose/tools/run_all.sh

# one variant only, or only some cases
bash ../GEANT4suitlab-dose/tools/run_lunar_pole.sh water10
CASES="gcr_max_Si gcr_max_Fe" bash ../GEANT4suitlab-dose/tools/run_lunar_pole.sh regolith1m
```

Every finished run appends one line to `habitat_summary.csv`, so a run can be
stopped (Ctrl+C) and resumed later; only the case in progress is lost. When the
same case was run more than once, the analysis uses the run with the most
primaries and ignores runs below 1000 primaries (viewer tests).

---

## Outputs

All in the build folder.

| File | Content |
|---|---|
| `habitat_summary.csv` | one line per run: D, H, errors, Q̄, nuclear share, N |
| `lunar_logs/`, `scan_logs/` | full Geant4 output of every run |
| `figures/table_4_1_validation.csv` | open surface vs Chang'e-4: D, H, Q̄, share of H per species |
| `figures/table_5_1_baseline.csv` | baseline habitat per species and total, solar min and max |
| `figures/fig_5_2_variants.png` | crew dose rate per habitat variant |
| `figures/fig_5_3_shielding_curve_{min,max}.png` | crew dose against wall areal density, per material |
| `figures/fig_5_4_nuclear_share_{min,max}.png` | share of the dose from nuclear secondaries |
| `figures/table_5_5_albedo.csv` | no ground / default ground / deep ground |
| `figures/table_5_6_horizon.csv` | crater horizon (analytic scaling by 1 − sin h) |
| `figures/fig_5_7_spe.png` | dose per August 1972-type SPE (log scale) |
| `figures/fig_5_9_species.png` | share of the GCR dose equivalent per species |
| `mission_dose_table.csv` | 100-day and 1-year doses against the 1 Sv limit |

Every PNG has a CSV with the same name, so every number can be traced back.

| Crew dose by habitat variant | Which species deliver the dose |
|---|---|
| ![](docs/fig_5_2_variants.png) | ![](docs/fig_5_9_species.png) |

---

## Options

All options are environment variables placed before the command, for example
`N=50000 bash .../run_lunar_pole.sh baseline`.

| Script | Option | Default | Meaning |
|---|---|---|---|
| `run_lunar_pole.sh <variant>` | `N` | 100000 | primaries per case |
| | `THREADS` | all performance cores | Geant4 worker threads |
| | `SPECIES` | `p He C O Si Fe` | GCR species to run |
| | `SOLARS` | `min max` | solar conditions |
| | `NOSPE` | unset | `1` skips the SPE |
| | `HORIZON` | 0 | crater-rim elevation in degrees |
| | `CASES` | all of the above | list the cases yourself |
| `run_wall_scan.sh` | `SOLAR` | `min` | `min` or `max` |
| | `MATERIALS` | `G4_WATER G4_Al G4_POLYETHYLENE LunarRegolith` | wall materials |
| | `AREALS` | `1 2 5 10 20 40 70` | wall areal densities (g/cm²) |
| `run_all.sh` | `QUICK` | unset | `1` runs a fast test of the whole chain |
| | `STEPS` | all | `variants albedo validation scan_min scan_max figures` |
| `make_figures.py` | `--species` | all six | e.g. `--species p He` for a protons + helium comparison |
| `mission_dose.py` | `--species` | all six | species to add up |
| | `--scale` | 1 | GCR factor: a number, or `changE4` (see below) |
| | `--days` | `100 365` | mission durations |
| | `--spe` | 1 | number of SPEs at solar maximum |
| | `--limit` | 1000 | dose limit in mSv |

---

## Run times

| Part | Runs at default settings | Note |
|---|---|---|
| 7 variants × 14 cases | 98 | thick regolith covers and heavy ions are the slowest |
| albedo | 12 | solar minimum only |
| validation | 6 | solar minimum only |
| wall scan, min + max | 336 | 4 materials × 7 thicknesses × 6 species × 2 |

Run time depends strongly on the computer. On a laptop, the variants alone can
take more than a day; the full wall scan can take weeks. Use the lighter
scan settings above, or run steps separately.

---

## Validation and the heavy-ion question

`mission_dose.py` prints the validation line first, then the mission table and
the share of the dose equivalent per species.

| Route | Command | Meaning |
|---|---|---|
| Explicit heavy ions (main result) | `mission_dose.py` | GCR = p + He + C + O + Si + Fe, all simulated |
| Validation-gap scaling | `mission_dose.py --species p He --scale changE4` | protons + helium only, multiplied by k = H(Chang'e-4) / H(open surface, p + He) |
| Consistency check | `mission_dose.py --scale changE4` | with all six species k is close to 1 (k = 1.37 for v8) |

With protons and helium only (v7), the model reached 29 % of the Chang'e-4 dose
equivalent; adding the heavy-ion groups raised this to 73 % and brought the
mean quality factor from 1.84 to 4.13.

---

## Limitations

- The habitat is a generic placeholder, not a specific design.
- Heavy ions are represented by four charge groups, one ion each; Z > 28 is neglected.
- Force-field modulation with typical modulation potentials, not the full
  BON2020 transport solution or a date-specific φ.
- BON2020 absolute scale tied to the Burger proton LIS at high energy; check the
  Table B1 parameters in `tools/make_spectra.py` against the original paper
  before reusing them.
- Whole-body water phantom; organ doses are not resolved.
- One reference SPE (August 1972), protons only.
- Crater horizon treated analytically; crater-wall albedo not modelled.
- Uncertainties are statistical only; hadronic-model uncertainties are not included.

---

## Version history (V8 available for public)

| Version | Main change |
|---|---|
| v3–v4 | habitat on regolith ground; ground blocks the lower half-sky |
| v5 | nuclear-secondary share (`nuc%`), albedo variants, wall scan on regolith |
| v6 | horizontal habitat, lying phantom, PE water tank, 10/25 cm regolith covers |
| v7 | N = 100 000 by default, SPE in two energy bands, best-run selection, `run_all.sh` |
| v8 | heavy ions (C, O, Si, Fe groups), BON2020 element spectra, separate helium LIS, Chang'e-4 scaling option, species figure |

Details of v8: [V8_GUIDE.md](V8_GUIDE.md).

---

## How to cite

If you use this code, please cite the repository and the release you used:

```bibtex
@software{geant4suitlab_dose,
  author  = {FULLOFSUN},
  title   = {{GEANT4suitlab-dose}: crew radiation dose in a lunar south pole habitat (SuitDose v8)},
  year    = {2026},
  url     = {https://github.com/FULLOFSUN/GEANT4suitlab-dose},
  version = {v8.0}
}
```

---

## References

- Agostinelli, S. et al. (2003). Geant4 — a simulation toolkit. *Nucl. Instrum. Meth. A* 506, 250–303.
- Allison, J. et al. (2016). Recent developments in Geant4. *Nucl. Instrum. Meth. A* 835, 186–225.
- Burger, R. A., Potgieter, M. S. & Heber, B. (2000). *J. Geophys. Res.* 105(A12), 27447–27455.
- Gleeson, L. J. & Axford, W. I. (1968). Solar modulation of galactic cosmic rays. *ApJ* 154, 1011.
- ICRP (1991). 1990 Recommendations of the ICRP. ICRP Publication 60. *Ann. ICRP* 21(1–3).
- King, J. H. (1974). Solar proton fluences for 1977–1983 space missions. *J. Spacecraft Rockets* 11(6), 401–408.
- Slaba, T. C. & Whitman, K. (2020). The Badhwar–O'Neill 2020 GCR Model. *Space Weather* 18, e2020SW002456; parameters in NASA/TP-2019-220419.
- Usoskin, I. G. et al. (2005). Heliospheric modulation of cosmic rays: monthly reconstruction for 1951–2004. *J. Geophys. Res.* 110, A12108.
- Zhang, S. et al. (2020). First measurements of the radiation dose on the lunar surface. *Sci. Adv.* 6(39), eaaz1334.

---

## Acknowledgements

This work was carried out during a UCD Erasmus+ traineeship at the Austrian
Space Forum (OeWF)

## License

To be added (for example MIT or BSD-3-Clause). Geant4 itself is distributed
under the [Geant4 Software License](https://geant4.web.cern.ch/download/license).
