# EKG evaluation output

One folder per dataset: public Apache project data turned into an Engineering Knowledge Graph in query-master's EKG schema (registry v1.0) by ekg-eval-data's adapters and LLM fill stage. Every record and link passes query-master's validator; nothing was dropped.

| Dataset | Sources | Records | Links | LLM added (records / links) | LLM tasks: accepted / rejected / error |
|---|---|---|---|---|---|
| `apache-kafka` | Jira, GitHub, git, mail, wiki, ASF roster | 76,489 | 145,463 | 1,702 / 3,964 | 668 / 148 / 0 |
| `apache-flink` | Jira, GitHub, git, code, 8 mail lists, ASF roster | 74,430 | 179,425 | 2,215 / 3,278 | 917 / 509 / 1 |
| `apache-iceberg-python` | GitHub issues and PRs, git, code, 4 mail lists, ASF roster | 29,354 | 82,770 | 499 / 731 | 247 / 522 / 0 |
| `apache-maka` | GitHub issues and PRs, git, code, 2 mail lists, ASF roster | 16,808 | 55,058 | 1,517 / 2,533 | 572 / 373 / 0 |
| `apache-ossie` | GitHub issues and PRs, git, code, 3 mail lists, ASF roster | 5,824 | 17,443 | 278 / 552 | 114 / 166 / 0 |

Records and links are after parsing; the LLM column is what the LLM stage added on top. The one error is a Flink README whose text is empty: the model has nothing to read, and the task is recorded with its reason in `llm/ledger.jsonl`.

## In each folder

| File | What it is |
|---|---|
| `records.jsonl(.gz)`, `links.jsonl(.gz)` | Stage 1: what the adapters parsed |
| `report.json` | Stage 1 report: which adapters ran, coverage, identity, every skipped row counted by reason |
| `llm/records.jsonl`, `llm/links.jsonl` | Stage 2: what the LLM added or filled |
| `llm/ledger.jsonl` | Every LLM verdict with its quote and reason |
| `llm/responses.jsonl(.gz)` | Every raw LLM response with its prompt; a rerun answers from it at no cost |
| `llm/patterns.json`, `llm/report.json` | Cues behind accepted answers (adapter hints); counts and reasons per gap kind |
| `graph.sqlite.gz` | Both stages in query-master's tables: `ekg_records`, `ekg_links`, `ekg_registry_*` |
| `manifest.json`, `inventory.json`, `state.json` | Four new datasets only: the run's settings, every raw file and what read it, each step's counts |

Files over GitHub's size limit are committed gzipped: `gunzip -k <file>.gz`.

## The four new datasets

They come from our own download (`test_data_sorce_graph/<name>/raw` and `codebase/`), laid out for the adapters by `tools/shim_all.py` in ekg-eval-data's `fix/eval-datasets` branch. The layout only links and regroups raw API responses; it parses nothing. `tools/ekg_run.py` runs layout, inventory, parse, LLM fill and save, each step resumable.

**Not read by any adapter yet** (listed per dataset in `inventory.json`): Jira remote links (full URLs, including PRs in other repos), Jira issues referenced from outside the window, GitHub's GraphQL commit → PR answers, and GitHub user profiles.

**Settings:** `infer_people: true`. On public data the committer roster is only part of who works on a project, so contributors are still inferred from commit and mail authorship. Inferred people name a person in credits and grant no access.

## Fixes made for these runs

In ekg-eval-data, branch `fix/eval-datasets`:

| Fix | Effect |
|---|---|
| Iterate sets in sorted order (12 loops) | Two runs give byte-identical files; before, every rebuild reordered thousands of rows |
| People adapter: a project with no committee (incubating) | maka and ossie had lost every person; now roster plus inferred contributors (ossie: 53 people, no duplicates) |
| Redact secrets in text instead of refusing the record | 5 emails and 27 links were dropped (R10); now kept with `[redacted …]` markers |
| Recognise ASF push notices by their body | They were read as people's mail and sent to the LLM: LLM tasks fell from 6,161 to 3,375 |
| Push notices for another repo stay unlinked | Their `#n` was linked to this repo's unrelated PR (an iceberg-rust push → an iceberg-python PR) |
| Own-repo push notices keep their ticket, `#n` and URL mentions | Most pushed commits fall outside the git window, so these are often the only links |

In query-master, branch `fix/registry-spec-hash`: the spec page is hashed with LF line endings, so `build_ekg_registry.py --check` passes on every OS (it failed outside Windows although the registry was identical).

On Kafka, every fix leaves the output byte-identical; query-master's EKG tests (354) and ekg-eval-data's tests (30 and 36) pass.
