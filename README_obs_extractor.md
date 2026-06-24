# Obs_extractor — Swift/UVOT Source Extraction Pipeline (AO 0235+164)

A SciServer notebook that queries and photometrically reduces archival *Swift*/UVOT
imaging of the blazar AO 0235+164, producing per-epoch, per-filter magnitude
measurements. This is the upstream extraction step that feeds the multiwavelength
light-curve / SED analysis in the
[AO-235-164_Study](https://github.com/Joey756/AO-235-164_Study) pipeline.

---

## What it does

1. **Query** — uses `pyvo` to TAP-query HEASARC's `swiftmastr` catalog for every
   observation pointed within 0.05° of AO 0235+164 (RA 39.66208°, Dec 16.61639°)
   inside a chosen MJD window.
2. **Locate** — for each returned `obsid`, finds the UVOT sky image directory on
   the SciServer `headata` FTP mirror (`/home/idies/workspace/headata/FTP/swift/data/obs/`).
3. **Extract** — for each filter image (`*_sk.img.gz`) in that directory:
   - decompresses it,
   - runs `uvotimsum` to coadd snapshots into one summed exposure,
   - runs `uvotsource` to do aperture photometry — 5″ source circle, 7″–40″
     background annulus, 5σ detection threshold — producing a magnitude/count-rate
     measurement.
4. **Save** — writes one `.fits` file per observation per filter to persistent
   SciServer storage. Already-extracted files are skipped, so the cell is safe to
   re-run after an interruption.
5. **Package** (second cell) — zips the results folder into `UVOT_data.zip` for
   download off SciServer.

---

## Requirements / environment

- Must run inside a **SciServer** compute container with HEASoft installed and the
  `headata` FTP mirror mounted.
- Python: `pyvo` (TAP queries); everything else is stdlib (`os`, `subprocess`,
  `glob`, `gzip`, `shutil`).
- The `username` variable must match your own SciServer persistent storage folder
  (currently hardcoded to `joel756`).

---

## Inputs

- `targets` — list of `{name, ra, dec}` dicts. Currently populated with just
  AO 0235+164; the loop is written to support multiple blazars but only one is
  defined.
- Source/background region files (`source.reg`, `bkg.reg`) are generated
  automatically from each target's `ra`/`dec` — no manual region-drawing needed.

## Outputs

- `{target_name}_{obsid}_{filter_id}_mag.fits` — one per filter per observation,
  in `output_dir`.
- `UVOT_data.zip` — packaged results, created by the second cell.

---

## Known issues (as of last run)

- **Silent failures**: the `uvotimsum`/`uvotsource` success checks have no
  `else` branch, so a failed extraction prints nothing. The last captured run
  found 89 candidate observations but only produced 3 output files, with no way
  to tell why the other ~86 didn't produce a result. Add
  `else: print(res.stderr)` after each `subprocess.run` call before trusting a
  full run.
- **MJD window**: confirm the `start_time` range in the TAP query matches the
  epoch range intended for the study — an earlier version of this pipeline used
  a different window and returned a different (larger) observation count.
- `filter_id` is parsed as the full filename prefix (e.g. `sw00030880103uvv`),
  not just the 3-letter filter code — harmless for output-file uniqueness, but
  don't reuse it expecting a clean filter label.

---

## How to run

On SciServer, in a Jupyter session with HEASoft loaded:

```bash
jupyter notebook Obs_extractor.ipynb
```

Run cells top to bottom. Re-running after a partial/interrupted run is safe —
existing output files are detected and skipped.

---

## Author

**Joel Owusu Boateng Kwakye**
GitHub: [@Joey756](https://github.com/Joey756)
