# GRAPE-CAST data repository — concept demo (tiered / de-identified)

A working proof of concept for Objective 1. It shows historical grape ripening data served as a static file and
queried live in the browser, with no server to keep running. It demonstrates a two-tier model. A private tier holds
the full per-site data and is never published. A public tier pools that data up to the AVA and region level, so no
individual vineyard appears, and only the public tier is served on the web.

The demo combines two New York programs for the same years (2024 to 2025), the NYWGF ripening dataset and the
Cornell Veraison to Harvest program, harmonized to one schema. The production repository would span every project
data source.

## What is in this repository

- `index.html` — the page. It loads a full SQL engine (DuckDB) in the browser, reads the public pooled file, and
  lets a user filter by cultivar, program, region, and year, switch the colour grouping, the Y-measure, and the
  X-axis (calendar date or cumulative GDD), see per-region trajectories with an overall average, a variance band,
  and a "now" marker, download the filtered records as CSV, and run raw SQL.
- `data/ripening_pooled.parquet` — the public, de-identified data. 832 AVA-level rows (about 25 KB). Each row is a
  regional mean for one program, cultivar, year, and GDD window, with `n_samples` and `n_sites`. No vineyard identity.
- `data/ripening_pooled.csv` — the same data as CSV.
- `README.md`, `.gitignore`.

The full per-site data and the build scripts are kept private and are not part of this repository. The `.gitignore`
excludes them so they are never published.

## The tiered / privacy model

- The public file is pooled to `program × region × cultivar × year × GDD-window`, and the vineyard `site` column is
  dropped, so the published data reports regional means rather than individual vineyards.
- Each public row carries `n_samples` (how many measurements) and `n_sites` (how many distinct vineyards) behind
  the mean. In this demo `MIN_SITES = 1`, so every group is kept. A production release would raise `MIN_SITES` (for
  example to 2 or 3) to suppress groups thin enough to re-identify a single grower. With this data, requiring at
  least two contributing vineyards leaves 123 of the 832 rows, which shows the real cost of strict suppression.
- Because DuckDB queries run in the visitor's browser, whatever is in the published file is fully downloadable. The
  privacy protection therefore lives in the build, not in the page. The private per-site data provides that
  protection only by never being published.

## View it locally

Serve the folder over http. Opening `index.html` from a bare file path will not work, because the browser blocks
the data fetch.

```bash
python -m http.server 8000
```

Then open `http://localhost:8000`.

## How the public data was produced

The public file is built from a private per-site dataset in two steps that run outside this repository. The first
harmonizes the source programs, NYWGF and Veraison to Harvest, into one per-site table. The second pools that table
to the AVA and region level, drops the vineyard `site` column, and keeps `n_samples` and `n_sites`, so no individual
vineyard appears in the published data.

## Publish the public tier on GitHub Pages

1. Create a GitHub repository.
2. Add `index.html`, the `data` folder, `README.md`, and `.gitignore`, then push to `main`. The `.gitignore` keeps
   `private/` out, so only the pooled public data is published.
3. In the repository, go to Settings, then Pages, and set Source to `Deploy from a branch`, branch `main`, folder
   `/ (root)`.
4. GitHub gives a public URL of the form `https://<user>.github.io/<repo>/`. That link is the live public tier.

## How this maps to the grant

- GitHub Pages serves the public tier for free, with nothing to keep running, and DuckDB-WASM runs the query in the
  visitor's browser. This backs the word "queryable" in Objective 1.
- The public data is de-identified by pooling to AVA and region, which is how grower and industry data can be shared
  openly without exposing an individual vineyard. This is the tiered-access model the Data Management Plan describes,
  open tier public, restricted tier held back and released only in aggregate or under data-use agreements.
- Each row keeps a `provenance` tag, the Objective 1 principle that harmonized records stay labeled by source.
- For a citable, preserved snapshot, deposit the public file to Zenodo or USDA Ag Data Commons at each release.

## Notes and limits

- Everything published is public. Row-level access control for named users would need a hosted backend with
  authentication (for example Postgres row-level security or Datasette with auth), which adds cost and maintenance.
  The pooled public tier avoids that by removing identity instead of gating access.
- The query runs client-side, so the whole public file loads into the browser. For this size that is trivial. A much
  larger public file would move to a hosted query service.
