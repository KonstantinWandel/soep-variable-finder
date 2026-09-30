# SOEP Variable Finder

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.21134306.svg)](https://doi.org/10.5281/zenodo.21134306)

The SOEP Variable Finder is a semantic search over the variable metadata of SOEP-Core v41, the
German Socio-Economic Panel. You describe what you are looking for in plain language, in German or
English ("wie oft Fleisch", "net labour income"), and get back the variables that measure it, each
with its labels, dataset, topic, question wording, answer categories, survey years and a link to its
page on [paneldata.org](https://paneldata.org/).

It runs at <https://soep-faiss.geolab.soz.uni-bielefeld.de/> and is part of the
[GeoLAB](https://geolab.soz.uni-bielefeld.de/) of the Leibniz ScienceCampus SOEP-RegioHub at
Bielefeld University and DIW Berlin.

> Status: research prototype. Retrieval is semantic and imperfect; check a hit against the SOEP
> documentation before you use it.

## Where the metadata comes from

125,496 variables from the 622 datasets of SOEP-Core v41. The raw per-wave files are hidden by
default, which leaves 23,855 variables in a default search.

- **Labels** in German and English, **concepts**, the **topic tree** and the dataset descriptions come
  from the SOEP-Core metadata repository on GitHub,
  [`paneldata/soep-core`](https://github.com/paneldata/soep-core).
- **The question each variable was asked with** and the **English value labels** come from the
  SOEP-Core documentation of DIW Berlin,
  [`git.soep.de/kwenzig/publicecoredoku`](https://git.soep.de/kwenzig/publicecoredoku). Its schema is
  described in [`git.soep.de/mds/soep-meta`](https://git.soep.de/mds/soep-meta) and in Knut Wenzig,
  [*Die Tabellen des SOEP-Metadatensystems*](https://www.diw.de/documents/publikationen/73/diw_01.c.1017827.de/diw_spp1660.pdf),
  SOEPpaper 1660 (2026).
  Where the documentation links no question, the question of the variable's concept is used.
- **Value labels** (German), a **distribution summary** (range and mean) and the **survey years** are
  read from the SOEP-Core v41 data files, because the export carries no value labels. The years are the
  waves in which a variable has valid values; for files without a survey year they come from the
  documentation's question links, from the waves a generated variable is built from, or from the wave
  file's own year. 22,065 of the 23,855 variables in a default search have years.
- For about 2,400 variables whose label exists in one language only, a machine translation of the other
  language is added as extra search text and marked as such. It never replaces an official label.

A few hand-written additions support the ranking: short notes on ten datasets (for example that `pgen`
holds the processed standard variables), synonyms for a handful of frequently searched variables such
as `pglabnet`, and a small preference for generated datasets over subsample instruments.

## How the search works

- Each variable becomes a short text: name, labels, dataset, topic, answer categories, question and
  years. A multilingual embedding model,
  [`intfloat/multilingual-e5-large-instruct`](https://huggingface.co/intfloat/multilingual-e5-large-instruct),
  turns each text into a vector once, when the index is built.
- A query is turned into a vector the same way, and the variables with the closest vectors are the
  candidates. This is why "how often do people eat meat" finds `ple0179` "Wie oft Fleisch" although
  the two share no word.
- A cross-encoder, [`BAAI/bge-reranker-base`](https://huggingface.co/BAAI/bge-reranker-base), reads
  each of the best candidates together with the query and puts them in a new order. The final rank
  also counts shared words, and a typed variable name such as `pglabnet` goes straight to the top.
- Filters by sample group, dataset, topic and survey years narrow the search beforehand.
- The finder only retrieves: every hit is an existing metadata record, and nothing is generated.

`scripts/eval_soep_search.py` is the retrieval test: 59 queries whose correct variables were fixed in
advance. On 30 September 2026 a correct variable stood at rank 1 for 50 of them and within the top ten
for 58.

## Models

Downloaded from Hugging Face at runtime and cached locally:

- `intfloat/multilingual-e5-large-instruct`, bi-encoder, MIT.
- `BAAI/bge-reranker-base`, cross-encoder, MIT. Production runs an int8 ONNX export of it.

## Building the index

```bash
OPENBLAS_NUM_THREADS=1 Rscript scripts/extract_soep_v41_value_labels.R   # labels, distributions, years
python scripts/build_soep_v41_metadata.py                                  # -> soep_metadata_output/
```

The first step reads the SOEP-Core v41 data files, which are available under the SOEP data-use
agreement, and writes one summary line per variable. The second needs the paneldata export and a clone
of the DIW documentation; the script's docstring names both. The service builds the embeddings
(`build_and_save_embeddings`), and `bash frontend/build.sh soep` builds the interface.

## Running it

The production service runs natively: uvicorn under systemd, behind Caddy, on a virtual machine of
the Faculty of Sociology (8 cores, no GPU; a query takes about a second). `deploy/` holds container
stacks for hosts with a container runtime.

## Data and licences

This repository holds code only. The SOEP metadata is published by DIW Berlin; the data files needed
to rebuild value labels, distributions and years are covered by the SOEP data-use agreement. The
running service holds metadata only, no microdata. Embeddings, indexes and any credentials are
git-ignored.

## Repository layout

```
backend/        FastAPI app; app/services/soep_rag_advisor.py is the retrieval service
frontend/       React/Vite interface (build.sh)
scripts/        metadata builders, the label extractor, retrieval tests
deploy/         container stacks and Caddyfiles
MAINTENANCE.md  operating notes, in German
```

The same code also serves GeoDB, the search over German regional data
([`KonstantinWandel/geolab-finder`](https://github.com/KonstantinWandel/geolab-finder)).

## Author

Konstantin Wandel, research fellow at Universität Bielefeld (SOEP-RegioHub):
<https://konstantinwandel.github.io/>

## Citing

Please cite the archived release; see `CITATION.cff`. Zenodo archives every GitHub release, and the
DOI above always resolves to the newest one.

## License

[MIT](LICENSE) © 2026 Konstantin Wandel. Part of the GeoLAB project, Universität Bielefeld.
