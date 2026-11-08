# Apache Kafka: Engineering Knowledge Graph status

This folder holds the graph built from three months of public Apache Kafka activity (2026-07-31 to 2026-10-29): Jira, GitHub, git, the `dev@` and `users@` mailing lists, the Confluence wiki and the ASF rosters. It is stored in query-master's EKG schema (registry v1.0). Every record and link passes query-master's validator.

## What is here

| File | What it is |
|---|---|
| `records.jsonl`, `links.jsonl` | Stage 1: what the adapters parsed from the raw data |
| `report.json` | Stage 1 report: per adapter, coverage, identity, notes |
| `llm/records.jsonl`, `llm/links.jsonl` | Stage 2: what the LLM added or filled, on top of stage 1 |
| `llm/ledger.jsonl` | Every LLM verdict (accepted or rejected) with its quote and reason |
| `llm/responses.jsonl` | Every raw LLM response with its prompt. A stopped run resumes from it without paying twice |
| `llm/patterns.json` | Recurring cues behind accepted answers: rules the adapters could learn |
| `llm/report.json` | Counts per gap kind, and why each unfilled gap stayed unfilled |
| `graph.sqlite.gz` | Both stages in query-master's tables (`ekg_records`, `ekg_links`, `ekg_registry_*`) |

`graph.sqlite` is 201 MB, over GitHub's file limit, so only the gzip copy (39 MB) is committed. To use it, run `gunzip -k graph.sqlite.gz`. Read-only views `node` and `claim` show the same rows under the root schema's names. `_eval_meta` records how the file was built.

## How it was built

```
raw download → adapters (parse) → LLM fill (accept or reject) → validator → JSONL + SQLite
```

1. **Parse.** The adapters turn each tool's data into records and links: fields, conventions such as `KAFKA-123` in a PR title, and text rules.
2. **LLM fill.** Code finds every link or field still empty, plus the candidates for it. The LLM (`mistralai/devstral-2512` through query-master's own client) either accepts one candidate, quoting the text that proves it, or rejects them all. Code then checks each acceptance: the choice must be a candidate, the quote must appear word for word, and the result must pass the validator. No person reviews in between. Accepted answers are stored with method `llm_judge` and their evidence, and every pass has a DerivationRun record.
3. **Save.** Both stages go into the SQLite file, with every row validated again.

## Where the graph stands

| | Records | Links | Record types |
|---|---|---|---|
| After parsing | 76,489 | 145,463 | 32 of 106 |
| After the LLM | 78,191 | 149,427 | 38 of 106 |

| Measure | After parsing | After the LLM |
|---|---|---|
| Commits linked to their PR | 565 / 568 | 565 / 568 |
| PRs linked to a ticket | 235 / 300 | 237 / 300 |
| Tickets linked to a PR or commit | 882 / 1,120 | 882 / 1,120 |
| Emails linked to a ticket or decision | 1,231 / 1,947 | more, through 22 thread links |
| KIP pages with a status | 1,263 / 1,348 | 1,320 / 1,348 |
| Accounts linked to a person | 213 | 229 |
| Knowledge, guidelines, requirements | none | 898 / 179 / 220 |
| Action items, open questions | none | 162 / 38 |
| Decisions found in conversations (beside the 1,348 KIPs) | none | 256 |

For Kafka the parse stage already links code to tickets almost completely. Most of what the LLM added is meaning that parsing cannot reach: decisions and their reasons, guidelines, requirements, gotchas and workarounds, action items and open questions. Each one is quoted from the email, comment or document it came from and linked to it.

## The LLM run

The run asked 816 tasks. On the rebuild every answer came from the stored responses (0 new calls), with 0 errors.

| Gap | Asked | Accepted | Rejected | Why rejected |
|---|---|---|---|---|
| KIP status | 84 | 57 | 27 | The page states no status the schema has |
| Mail thread → ticket or KIP | 107 | 22 | 85 | Build noise, release votes, unrelated threads |
| Knowledge from conversations | 502 | 485 | 17 | Nothing worth keeping |
| Knowledge from documents | 86 | 86 | 0 | |
| Account → person | 20 | 16 | 4 | First name only |
| PR → ticket | 12 | 2 | 10 | The PR really has no ticket |
| Fixed ticket → its fix | 5 | 0 | 5 | Only look-alike candidates |

Code settles a further 1,233 gaps without asking the LLM. Examples: 51 `MINOR:` PRs, which need no ticket by Kafka's own rule, and 1,153 accounts with no name match to any person. Each is counted with its reason in `llm/report.json`.

**Quality.** In a random sample of 12 knowledge items, 10 are useful and 2 are trivial, so expect some noise. Identity links are stored as `inferred`: they name the person in credits but never grant access.

## Fixed since the first run

- **Wiki adapter** reads quoted KIP states (`Current state: "Under Discussion"`): status tasks fell from 323 to 84.
- **Schema** `Decision.status` has `rejected` and `withdrawn`; DECIDED_BY and ASSIGNED_TO also go to Alias and EmailAddress.
- **Mail adapter** mentions the ticket keys in an email's own body, not only its subject.
- **Adapter output** is byte-stable from run to run (branch `fix/eval-datasets` in ekg-eval-data): this folder no longer changes order on every rebuild.

## What to fix next

- **GitHub adapter:** a ticket key in the PR's branch name is not used.
- **Prompts:** reject first-name-only identity matches; make the knowledge judge drop one-off details.
- **Schema:** a decision made in a PR review cannot be stored, because `MADE_IN` does not allow a Review target.

## Rebuild

From the workspace root, with ekg-eval-data's `main`:

```bash
export LLM_BACKEND=openrouter OPENROUTER_API_KEYS=<key> EKG_LLM_MODEL=mistralai/devstral-2512
python -I ekg-eval-data/run_pipeline.py --data ekg-eval-raw/apache-kafka   # parse → LLM → SQLite
```

A rebuild re-asks only tasks whose text or candidates changed; every other answer comes from `llm/responses.jsonl`.
