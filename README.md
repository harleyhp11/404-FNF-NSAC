# 404:FNF fire safety workbench

Formerly "FlameBench"; the interface now carries the team name, 404:FNF.

A web fire-safety workbench for space habitats (NASA Space Apps 2026, *Flame in Freefall* challenge). It brings the 24 NASA PSI combustion investigations into one comparable table and adds external evidence with its evidence level stated.

The dashboard interface is in English.

## What FlameBench is not

**FlameBench does not certify materials, and it does not predict in general what will burn on the Moon or on Mars.** Material certification is governed by NASA-STD-6001, not by this tool.

Where the app shows a boundary, a limit or a prediction, that number is a calculation or an extrapolation from a specific experiment, and the view says which. The flammability boundary it draws is calibrated on PMMA rods at about one atmosphere; outside that case it is a reference curve that classifies nothing. Treat it as a tool for finding which experimental evidence applies to a scenario, and where evidence is missing.

## Visual system

The interface uses the 404:FNF "two flames" look: graphite surfaces, **ember** for fire, risk and primary actions, and **CH\* blue** for extinguished outcomes and official PSI evidence. It is dark by default; the sun/moon button in the top bar switches to the light theme and remembers the choice in this browser.

All of it lives in the `<style>` block of `src/app.html`. The charts and views read the same CSS variables as before (`--s1`, `--s2`, `--g1`, `--accent` and so on), so restyling never touches the application logic. A small second `<script>` at the end of the file handles the theme button and replays the entrance of the work area when the visible view changes; it reads the app state and does not modify it.

Fonts: Urbanist and JetBrains Mono (both OFL), loaded from Google Fonts. The mark in the top bar is embedded in the file.

Third-party interface code: the three pillar folders in the evidence view are the **Glass Folder Cards** component from the 21st.dev component library (its source names no author). Its artwork generator, geometry and stylesheet are used as provided; the React rendering was rewritten as plain DOM so it runs in this single-file page. Each folder is a pillar, its files are that pillar's first PSI investigations (each opens its PSI record), and clicking a folder selects the pillar. The turning ring in the copilot window is the **AI loader** component from the same library (no author named), recoloured to the interface's ember and CH\* blue; it appears only inside the copilot.

## Structure

- `src/app.html` — **the application** (HTML, CSS and JS). Edit here.
- `src/build.py` — inserts `data/data.json` into `src/app.html` and writes the compiled app.
- `data/data.json` — the bundle the app reads (tests, investigation records, external evidence, report passages).
- `pipeline/` — the Python that builds the data and the tests:
  - `measurements.py` — pure parser that preserves measurement qualifiers
  - `normalize.py` — PSI tables into one test-level table
  - `investigations.py`, `gapmap.py`, `inventory.py`, `unpack.py` — supporting scripts
  - `build_data.py` — builds the app bundle, and holds the curated investigation text
  - `test_measurements.py`, `test_data.py`, `test_app.py` — the test suite
- `data/raw_tables/` — the 22 original PSI tables. `data/clean/` — normalised tables.
- `docs/screenshots/` — current screenshots of the interface.
- `index.html` and `site/index.html` — the compiled single-file app.

## Getting the source data

The pipeline scripts do **not** run without arguments. `normalize.py` and `build_data.py` both need a PSI data directory, and `data/raw_tables/` is not that directory on its own.

`PSI_DIR` must contain at least:

```
<PSI_DIR>/
  tables/          the 22 experimental CSV tables, named PSI-<id>_Experimental table_<name>.csv
  metadata/        PSI-<id>.json for every investigation in ORDER (build_data.py)
  clean/           written by normalize.py; build_data.py reads clean/tests.csv and clean/publications.csv
  documents/PSI-26/PSI-26_Science Documents_As Run_Test Matrix.xlsx
                   supplies BASS oxygen, pressure, flow and crew observations
  documents_text/  extracted report text, for the assistant's passages
```

To obtain it:

1. Paste `pipeline/harvest_in_browser.js` into the browser console with `psi.nasa.gov` open. It downloads `psi_bundle_metadata_tables.json` (metadata plus tables).
2. `python3 pipeline/unpack.py psi_bundle_metadata_tables.json <PSI_DIR>`
3. Copy the PSI documents into `<PSI_DIR>/documents/PSI-xx/` so BASS can read the As Run matrix.

The version used for this build was downloaded on 14 and 15 November 2026. Do not point `PSI_DIR` at `.` unless that structure has been prepared there, and do not depend on one person's local path or commit any credentials.

Reading the As Run matrix needs `openpyxl`:

```bash
python3 -m pip install --break-system-packages openpyxl
```

## Rebuilding

From the repository root, with `PSI_DIR` set to the prepared directory:

```bash
PSI_DIR="/path/to/psi_data"

# 1. Normalise the PSI tables and documents into clean/tests.csv
python3 -I pipeline/normalize.py "$PSI_DIR"

# 2. Build the JSON bundle the app consumes
python3 -I pipeline/build_data.py "$PSI_DIR" data/data.json

# 3. Insert data/data.json into src/app.html, writing index.html and site/index.html
python3 src/build.py

# 4. Run the tests
python3 -m pytest pipeline/ -q
```

`data/data.json`, `index.html` and `site/index.html` are generated. Regenerate them from the sources and commit them together; never hand-edit them.

To change investigation text, the environments or the external evidence, edit the `C`, `MEAS`, `ENV`, `EXT` and `EXT_TESTS` structures in `pipeline/build_data.py` and rebuild.

## Tests

```bash
python3 -m pytest pipeline/ -q
```

- `test_measurements.py` — the qualifier parser, with no I/O and no pipeline import.
- `test_data.py` — the data layer: record classification, provenance, per-variable quality, and the figures the interface shows.
- `test_app.py` — the interface in a real browser, asserting behaviour rather than printing it.

The browser tests need Playwright:

```bash
python3 -m pip install --break-system-packages playwright
python3 -m playwright install chromium
```

If a Chromium or Chrome build is already installed, point `CHROME_PATH` at it instead:

```bash
CHROME_PATH=/usr/bin/google-chrome python3 -m pytest pipeline/test_app.py -q
```

`test_app.py` skips itself when neither Playwright nor a browser is available, so the data tests still run. Paths are resolved relative to the repository, so no absolute path needs editing.

## Record inclusion rules

One definition of "individual test" is used by the whole pipeline and by all three
modules of the app. `level` carries it:

| `level` | Meaning | Counted as a test | Plotted |
| --- | --- | --- | --- |
| `test` | one individual experimental run | yes | yes, if its variables are valid |
| `test-day summary` | aggregate over a test day | no | no |
| `campaign summary` | aggregate over a campaign | no | no |
| `dataset`, `session` | metadata | no | no |
| `note` | a footnote carried in the source table | no | no |
| `visualization` | a pre-test flow-visualisation record | no | no |

`is_individual_test` (CSV) and `ind` / `plot` (bundle) are derived from `level` alone. `plot` says the record is an individual test; it does **not** promise that every one of its variables is valid. That is what the per-variable quality fields are for.

Of 2,009 source rows, **1,871 are individual tests**. Eight rows that earlier counted as tests are not: two pre-test flow visualisations in SAFFIRE-I, two in SAFFIRE-II, and four SAFFIRE-II footnotes whose Sample Number holds the note text. They are kept with their source text and row number, so they remain available as documentation, but they are not counted, plotted or modelled, and they carry no outcome.

## Data-layer fields

Beyond the original columns, `data/clean/tests.csv` and `data/data.json` carry provenance so that no value is presented as something it is not.

| CSV | Bundle | Meaning |
| --- | --- | --- |
| `is_individual_test` | `ind`, `plot` | one individual experimental run |
| `pressure_source` | `pstate` | `measured`, `assumed` or `no_data` |
| — | `pas` | 1 when the pressure is the cabin assumption |
| `o2_initial_pct` | `o2i` | calibrated initial oxygen |
| `o2_final_pct` | `o2f` | calibrated final oxygen, kept apart from the initial |
| `o2_final_valid` | `o2fv` | 0 when the final value is anomalous; the value is still preserved |
| `o2_raw`, `o2_kind` | `o2raw`, `o2k` | the source cell and its qualifier kind |
| `pressure_raw`, `pressure_kind` | `pk` | the source pressure cell and its kind |
| `thickness_raw`, `thickness_kind` | — | the source thickness cell and its kind |
| `flow_cm_s_raw`, `flow_cm_s_min_raw` | `flmaxr`, `flminr` | flow **with its sign**, as recorded |
| `flow_cm_s`, `flow_cm_s_min` | `flb`, `fla` | flow **magnitudes**, which the charts compare |
| `flow_valid` | `qfl` | 0 when the flow is not valid for a magnitude analysis |
| `fuel_raw` | — | the material cell exactly as the source wrote it |
| `material_inherited` | `matinh` | the material was inherited from another row |
| `material_source_row` | `matsrc` | the row it was inherited from |
| `material_unresolved` | — | the material is not stated and continuity is not confirmed |
| `outcome` | `out`, `outr` | display label, and the raw code for matching |
| `qa_flag` | `qa` | the legacy single row-level warning |
| — | `qo2`, `qp`, `qfl`, `qout` | per-variable validity |

Quality is judged **per variable**. A suspect flow sets `qfl` to 0 and leaves that row's oxygen and pressure usable, so one bad column never discards a whole observation. Row-level `qa_flag` is kept for continuity and counts 23 rows: 11 `source_anomaly`, 8 `flow_invalid`, 2 `o2_invalid`, 2 `o2_corrected`.

### Measurement qualifiers

`pipeline/measurements.py` parses a source cell into a value, an operator, a kind, endpoints and a unit. Kinds are `exact`, `approximate`, `censored`, `interval`, `empty` and `unparseable`.

- `~ 21.5` is **approximate**, not exact.
- `< 0.1` is **censored**: a bound, not a measurement. Censored values are excluded from ordinary regressions by default, and the app reports how many it excluded.
- `22.1 to 22.0` is an **interval**: both endpoints are kept, and it is never silently reduced to its first number.
- `See Fig. 5`, `n/a` and `Insignificant` are **unparseable**, and never become 0.

`pipeline/measurements.py` and the `parseCell` function in `src/app.html` implement the same rules and are tested against the same cases.

### Flow sign convention

The source records a negative minimum flow in four BASS rows (tests 3, 7, 8 and 10: −4, −3, −2 and −2 cm/s). The sign is preserved in `flow_cm_s_min_raw` / `flminr` and shown in the test detail marked with `*`. Charts that compare speed use the magnitude and say so.

The sign is **not** used to infer flow direction. The direction in `flow_dir` comes from the investigator, following the BASS-II ReadMe (Olson concurrent; Fernández-Pello, Bhattacharjee and Miller opposed; T'ien concurrent). Establishing what the sign means in the "Air display" column needs the original BASS report; until then the signed value is carried as data and nothing is concluded from it.

### Pressure provenance

- **measured** — a pressure recorded in the source.
- **assumed** — no pressure in the table and the experiment ran in a pressurised cabin (MSG or Cygnus): 101.3 kPa is assumed, `pas` is set, and the value is marked `≈` wherever it appears.
- **no_data** — no measured pressure and no applicable assumption. Shown as "no data", never as out of range.

Counts: 1,129 measured, 804 assumed, 76 no data. A derived oxygen partial pressure inherits the provenance of the total pressure it came from and is labelled as calculated.

### Corrections applied to the source, and why

Each is logged in `notes` and is reversible.

- **SAFFIRE-II thickness `037 mm`** is read as 0.37 mm. The leading zero with no decimal point is a dropped decimal; the NASA Saffire II sample layout (NTRS 20150000894) gives 0.33 mm for SIBAL and 0.36 mm for the silicone downward-burn sample.
- **SAFFIRE-I thickness column** is headed `(cm)` with the value 0.37 for SIBAL fabric. Read as centimetres it gives 3.7 mm, ten times the SAFFIRE-II value for the same fabric and the NASA layout's 0.33 mm. It is read as millimetres instead. The switch is the single constant `SAFFIRE1_THICKNESS_UNIT_IS_MM` at the top of `normalize.py`; set it to `False` to follow the column header literally.
- **SAFFIRE-II blank materials** in rows `2-6` and `2-9` are inherited from `2-5` (SIBAL) and `2-8` (PMMA). The NASA sample layout lists samples 5 and 6 as SIBAL fabric 1 and 2, and samples 8 and 9 as Structured PMMA and Flat PMMA, so the continuity is supported by the source rather than guessed. Every inheritance records the row it came from. A blank whose continuity is not supported would be left unresolved instead.
- **A composite material stays composite.** `PMMA & Nomex` is not counted as pure PMMA when the PMMA boundary is under discussion.
- Pre-existing corrections are unchanged: two ACME BRE capture errors (`3400%` to 34 %), two impossible Flame Design oxygen values dropped, and eight negative SPICE flows flagged.

## Limitations and assumptions

- **The flammability boundary** is equation 6.1 of the BASS-II report, calibrated on PMMA rods in microgravity at about 1 atm. The app draws it as a reference inside that domain. It does not partition the chart into burning and non-burning regions, because the plotted points come from different materials, geometries and conditions. The fidelity of the implemented equation, its constants and its units has not been independently verified against the report; that check is still outstanding.
- **The gravity bridge** `U_b = 20·g^n` is a modelling hypothesis of the team, not a measured relationship. Checking it at 1 g cannot validate `n`, because `1^n = 1` for every `n`; that check only confirms the 1-g reference point. The exponent sits under "Advanced assumptions" and the view shows a sensitivity table over n = 0.25, 0.333 and 0.5, stating whether the conclusion changes.
- **The rod radius** (default 0.318 cm) applies only where the calculation is formulated for a rod and a radius is documented or explicitly assumed.
- **Fit is not validation.** R² measures agreement with the rows used to fit. Where the data allow it, the app reports a grouped out-of-sample validation (Q²) separately, with folds built from whole campaigns or samples rather than from individual rows, because rows from one campaign are not independent. Where validation is not possible, the app says so rather than letting the fit figure stand in for it.
- **Residual spread is not a prediction interval** and is labelled as scatter about the fit.
- **Extrapolation** outside the measured range of X is labelled, drawn differently and never presented as a prediction.
- **Relevance rankings** cover six environments (ISS, nominal vehicle, lunar base, pressurised rover, Mars habitat, spacesuit). Earth and EVA preparation reuse the closest rated environment, and the interface says so instead of presenting a rating that was never made.
- **Identical scenarios are flagged.** The ISS and the nominal cabin share the configured oxygen, pressure and gravity, as do the lunar base and the pressurised rover. The app says so, so no difference is read into a calculation that cannot see one.
- **Compatibility has four states** — experimental match, partial match, related evidence, insufficient evidence — so a missing variable is never reported as a mismatch of conditions.
- **A forced comparison does not pool data.** Comparing two investigations below the affinity threshold shows their features and conditions with the warning kept visible; it does not combine their rows into one model.
- **The AI-written content** is the investigation summaries, the pillar classification and the per-environment relevance, written from the PSI metadata and reports and reviewed by the team. Numbers in tables and charts come from the data. The assistant retrieves passages and is instructed to abstain when they do not cover the question.
- **Rockets and propellants** are outside the scope of this dataset.
- The headline gap: no PSI solid-material test reaches 30 % oxygen. The highest is 22.7 %, all at about 1 atm, so the lunar-base region (≥30 % O₂, ≤10.2 psi) has no solid-material data at all.

## External and time-dependent claims

Every external evidence entry carries a `status` and the date it was last checked, and mission status, experiment status and results availability are stated separately. As last checked on 15 November 2026:

- **FM²** is **planned, not flown.** NASA targets a launch in late 2026 on the robotic CLPS lander mission CP-21. No surface measurements or results exist. Some press reports tie FM² to a crewed Artemis landing; NASA's own description assigns it to CP-21.
- **SoFIE** has been **operating on the ISS** since its first GEL test on 13 January 2023. GEL, MIST and RTDFS have run; NCA and SMuRF have not. Results have been presented at conferences and papers are planned, so published results are limited.
- **LUCI** flew on 4 February 2025 and its results were presented at ASGSR 2025.
- **Saffire IV, V and VI** were performed and their results published (2020, 2021 and December 2023).

## Publishing

GitHub → Settings → Pages → Deploy from branch `main`.

`src/build.py` writes `index.html` and syncs `site/index.html` by default, so whichever folder Pages is configured to serve matches the build that was tested. **Confirm the real setting under Settings → Pages before the final push.** Use `python3 src/build.py --root` to write only `index.html`.

Test the published page, not only the local file: browser storage and the download flow behave differently when the app is served. Workspace saving uses `localStorage`, wrapped so that a blocked or full store leaves the app working; when storage is unavailable the app says so and the data exports remain available.

## Data and citation

NASA Physical Sciences Informatics, licensed CC0-1.0. PSI asks that the DOI of every investigation used be cited; the DOIs are in `data/clean/investigations.csv` and in the app's export of citations.
