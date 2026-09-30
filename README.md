# wat-signatures

This repository holds the data behind the article *Mapping Adipocyte Heterogeneity Across Single
Cell/Single Nucleus Studies by Machine-Assisted Marker-Network Analysis* (Arbones-Mainar JM,
Clavel-Millan M, Perez-Grijalba V, Encabo-Berzosa MM). It contains the screening and extraction
database, the marker-concordance map that the eleven recurrent adipocyte-lineage signatures come
from, the first-pass map that the sensitivity re-run compares against, and the signature catalogue
itself.

The review searched PubMed and Europe PMC for primary studies, published from 2018 onward, that
profiled human or mouse abdominal white adipose tissue by single-cell, single-nucleus or spatial
transcriptomics. Of 1,199 deduplicated records, 182 were included. Of those, 109 generated original
data, and 63 of the 109 reported at least one adipocyte-lineage subclass with a usable marker list,
giving 288 subclass entries. Clustering those entries by marker overlap gave 137 harmonized
populations, and eleven of them form the recurrent-signature catalogue.

## Files

| File | Size | What it is |
|---|---|---|
| `wat_screening_v35.db` | 8.3 MB | SQLite. Every record, its screening decision, and the structured extraction of the 182 included studies. |
| `adipose_concordance_map_v4.db` | 356 KB | SQLite. The concordance map used throughout the manuscript. |
| `adipose_concordance_map.db` | 307 KB | SQLite. The first-pass concordance map; kept only as the run the sensitivity re-run compares against. |
| `signature_catalogue.json` | 84 KB | The eleven recurrent signatures with their markers and cross-run stability. |

Both databases are in rollback-journal mode (`journal_mode=DELETE`), so they open read-only with no
sidecar files. Open them with `sqlite3 -readonly` or `?mode=ro` to be sure nothing is written back.

## `wat_screening_v35.db`

Ten tables, joined on `record_key`.

| Table | Rows | Contents |
|---|---|---|
| `studies` | 1199 | One row per deduplicated record: PMID, DOI, PMCID, title, abstract, journal, year, publication types, MeSH terms, and which query blocks of which source retrieved it. |
| `screening_log` | 1199 | One title-and-abstract decision per record (`include` / `exclude`), the exclusion code, a one-line reason, the screener model, and abstract-level impressions of species, depot, input type and modality. |
| `exclusion_vocabulary` | 12 | The prespecified exclusion codes and their definitions (Supplementary Table S1). |
| `depot_audit` | 164 | Re-adjudication of the records first excluded as `NON_WHITE_ADIPOSE_ONLY`. |
| `fulltext_status` | 187 | Full-text retrieval tier and route per record. Five rows belong to records later excluded; join to `screening_log` before counting. |
| `extraction_runs` | 182 | One completed extraction per included study, with full-text and supplement retrieval flags. |
| `extraction_fields` | 3822 | The extraction in long format: one row per study and field, with the value, `is_null`, a note on what was searched when null, source type, source locator and a verbatim flag. |
| `extraction_field_vocab` | 21 | The frozen extraction rubric, one row per field. |
| `adipocyte_subclasses` | 614 | One row per author-named adipocyte-lineage subpopulation, with its verbatim marker genes and their own source locator. |
| `data_provenance` | 182 | ORIGINAL / REUSED / MIXED / UNCLEAR per study (109 / 38 / 11 / 24). |

Values were copied verbatim from each paper and never inferred. An unreported field carries the
literal string `NULL` and a note of what was searched. The database holds no harmonized nomenclature;
all reconciliation across studies lives in the concordance map. The `species`, `depot`,
`input_type` and `modality` columns of `screening_log` are screener impressions from the abstract and
are adequate for corpus-level breakdowns only; use `extraction_fields` for anything per study. The
`abstract` column is the text as retrieved from PubMed or Europe PMC (present for 1,188 records).

The version suffix is historical. In v35 the `adipocyte_subclasses` rows of the 109 ORIGINAL studies
were re-extracted from scratch after a blind re-extraction of 10 studies found the first pass
collapsing distinct clusters that shared an author label. The 264 rows of REUSED, MIXED and UNCLEAR
studies still carry first-pass values; they do not enter the map.

## `adipose_concordance_map_v4.db`

Built from `wat_screening_v35.db`, restricted to ORIGINAL-provenance studies so that reanalyses of
the same public dataset are not counted as independent evidence.

| Table | Rows | Contents |
|---|---|---|
| `build_metadata` | 11 | The build parameters, stated as key/value pairs. |
| `population_members` | 288 | One row per contributing subclass entry: study, author label, verbatim and normalized marker genes (the latter a JSON list), species, depot and source locator. |
| `harmonized_populations` | 137 | One row per population: working name, naming rationale, mapping grade, member count, and core, contested and idiosyncratic markers. |
| `disagreeing_markers` | 475 | Per population and gene, how many members report it. |
| `excluded_studies_note` | 1 | Why 46 ORIGINAL studies contribute nothing. |

Marker strings were normalized (Greek letters transliterated, parenthetical prose removed, compound
lists split, gene-shaped tokens kept, uppercased, stop words removed). Every pair of the 288 entries
was compared by overlap coefficient, the intersection over the smaller set. Two entries are linked
when they share at least two genes and the coefficient is at least 0.30, and the weighted graph was
partitioned by Louvain community detection at resolution 2.0, seed 0. That gave 30 multi-entry
communities and 107 isolated entries. A population is `FULL` when at least one gene is reported by
every member, `PARTIAL` when it is clustered but no gene is unanimous, and `SINGLE_STUDY` when it has
no cross-study link (21 / 9 / 107). Working names were derived from recurring markers across members,
never from one study's own label. Four multi-member populations were left as
`heterogeneous/unresolved marker set` rather than forced into a name.

## `adipose_concordance_map.db`

The first-pass map, same schema, 292 entries from 68 studies in 133 populations. It was built from
the first extraction of `adipocyte_subclasses`, before the v35 re-extraction; its `build_metadata`
names that source as `wat_screening.db`, which was a working filename. It is superseded as a map and
is included only because the manuscript's run-to-run reliability analysis compares v4 against it.
The comparison works at three levels:

1. Marker transcription: entries aligned by the same PMID and the same `author_label` in both maps,
   keeping only pairs that are unique in each; Jaccard of their normalized gene sets (134 aligned
   entries, median 1.000).
2. Co-clustering: over all pairs of aligned entries, whether both maps place the pair in the same
   population (pair Jaccard 0.337, adjusted Rand index 0.493).
3. Supporting-study sets: each catalogue signature matched to its first-pass counterpart by marker
   overlap coefficient, then the Jaccard of the two sets of contributing studies (median 0.444).

## `signature_catalogue.json`

A list of eleven objects, one per recurrent signature, keyed by the v4 `population_id`. The fields:

| Field | Meaning |
|---|---|
| `population_id`, `working_name`, `mapping_type`, `n_member_labels` | As in `harmonized_populations`. |
| `core_markers` | Genes reported by every member label. |
| `contested` | Recurrent genes with `n_members_with_gene` of `n_total_members`. |
| `n_idiosyncratic` | Genes reported by one member label only. |
| `stable_across_runs`, `v4_only_markers`, `n_v4_markers` | Which markers the first-pass counterpart also recovers. |
| `pass1_counterpart`, `marker_overlap_with_pass1`, `support_set_jaccard` | The matched first-pass population and the level-3 agreement. |
| `reporting_studies` | Each member label with PMID, DOI, species, depot category, input material and source locator. |
| `transcription_note` | Free-text caveat where one applies. |

A catalogue signature was required to include labels from at least three ORIGINAL studies and to
have an interpretable adipocyte-lineage identity. The file was computed from the two maps during the
project's first, hosted phase, and the script that wrote it was not preserved, so it is provided as
a frozen input. Its fields derive from the two databases in this repository.


## Contact

Jose M. Arbones-Mainar,   
Instituto Aragonés de Ciencias de la Salud (IACS),  
Zaragoza, Spain.  
jmarbones.iacs@aragon.es. 