# SOEP Variable Finder

Project-specific guide. Workspace-wide rules live in `~/kwandel/CLAUDE.md` and apply here too.

## What this is

Semantic search over SOEP-Core **variable metadata** (~22k variables). A researcher types a
plain-language concept ("net individual income from labour", "Geschlechterrollen") and gets a
ranked list of variables with dataset, label, description, and the years they appear in. No
microdata is loaded, served, or aggregated: the finder points at variables, and the researcher
uses their own SOEP distribution. Live at <https://soep-faiss.geolab.soz.uni-bielefeld.de/>.

## This is one of two scoped publications of the same codebase

- this checkout pushes to `KonstantinWandel/soep-variable-finder` (SOEP finder, own Zenodo DOI)
- `/home/researcher/kwandel/destatis-rag` pushes to `KonstantinWandel/geolab-finder`
  (INKAR / GeoDB regional-indicator finder at <https://geodb.geolab.soz.uni-bielefeld.de/>)

The backend is the same file in both. **A backend change belongs in both checkouts**, as two
commits with the same content; check the other one before assuming a fix is shipped.

**`destatis-rag/CLAUDE.md` is the fuller document**: architecture, the common record schema, the
production VM (systemd, ports, caches), the embedding-cache footgun, the e5 query prefix, the
reranker decision, and the ongoing work to fold many German georeferenced data sources into the
GeoDB finder. Read it before making non-trivial changes here.

## SOEP-specific facts that live in this repo

- `GEOLAB_APP_MODE=soep` loads only the SOEP rows; `GEOLAB_ENABLE_DESTATIS=0` keeps the legacy
  Destatis table index and its e5-large model out of the process.
- Ranking carries a **dataset-authority prior**: `pgen`/`pequiv`/`ppathl`/`hgen`/`hpathl` and the
  main `pl`/`hl` questionnaires get a bonus; interviewer, fieldwork, instrumentation and design
  files get a penalty (the `interviewer` file re-uses person-questionnaire names and labels, so it
  used to win dedup over the real `pl` item).
- The SOEP missing-value boilerplate (`-1` to `-9`) is stripped before embedding; it is identical
  across nearly every variable and otherwise dominates the signal.
- Dedup is on `(variable_name, label)`, never on `variable_name` alone: 272 of 430 recurring
  variable names differ in meaning across datasets (`sex` is "Geschlecht" in one file and
  "Geschlecht des Kindes" in another). Collapsed rows are annotated with `also_in_datasets`.
- `DATASET_TITLE` maps dataset ids to real names for display ("Individual questionnaire (long)
  (pl)"), while the raw id stays the filter/URL key. Do not surface `.rds` filenames in the UI.
- Known limitation: querying a construct or battery *name* ("big five personality traits") does
  not surface the BFI-S items, because they are labelled by trait ("Bin gesellig", "gründlich").
  German trait-word queries do hit it. The fix is tagging battery items with their construct name
  during metadata enrichment.

## Environment

`/home/researcher/miniconda3/envs/geolab-rag/bin/python` on this box. Use the full path; do not
run bare `python3` and do not install into `base`.
