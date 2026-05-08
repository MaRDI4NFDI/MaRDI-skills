---
name: mardi-kg-query
description: Query the MaRDI Knowledge Graph and Portal (Mathematical Research Data Initiative). Use when the user wants to find mathematical software, models, formulas, datasets, or publications, resolve Portal Q-IDs, or run SPARQL against MaRDI or its linked sources (swMATH, zbMATH, arXiv, CRAN, MathModDB) — even when they don't say "MaRDI" and only name a linked source. Read-only.
---

# MaRDI Knowledge Graph — read-only CLI

The MaRDI portal is a Wikibase instance (same stack as Wikidata) holding ~5M items and ~500M relationships across mathematical research data sources (DLMF, swMATH, zbMATH Open, arXiv, CRAN, PolyDB, OpenML, …). Anyone can query it without authentication.

This skill drives `wikibase-cli` against MaRDI's endpoints. Stay strictly read-only — writes are deliberately out of scope.

## Endpoints

| What | URL |
|---|---|
| MediaWiki API (entity lookup, search) | `https://portal.mardi4nfdi.de/w/api.php` |
| SPARQL query service | `https://query.portal.mardi4nfdi.de/sparql` |
| Entity URI base (in SPARQL results) | `https://portal.mardi4nfdi.de/entity/Q<id>` |
| Human-readable item page | `https://portal.mardi4nfdi.de/wiki/Item:Q<id>` |

## Prerequisites

`wikibase-cli` provides the `wb` binary. Check first; install only if missing:

```sh
wb --version || npm install -g wikibase-cli
```

## Configure for MaRDI — use env vars, not `wb config`

Set these for the current shell:

```sh
export WB_INSTANCE=https://portal.mardi4nfdi.de/w/api.php
export WB_SPARQL_ENDPOINT=https://query.portal.mardi4nfdi.de/sparql
```

**Why not `wb config instance …`?** That command mutates the user's persistent CLI config and would silently redirect any *other* Wikibase work they do (Wikidata, a private instance) to MaRDI. Env vars are scoped to the shell, take priority over persistent config, and leave no side effects when the session ends. Only fall back to `--instance …` / `--sparql-endpoint …` per-command flags if env vars aren't viable.

If the user already has `wb` pointed at MaRDI, you can skip this step.

## Common commands

Search by label (free-text):

```sh
wb search "Riemann zeta"            # default lang en, type item
wb search --type property "cites"   # search properties
```

Fetch raw entity data (single Q-id or several):

```sh
wb data Q1089 --simplify            # cleaner JSON
wb data Q1089 --props labels,descriptions,claims
```

List claims for an entity (all, or one property):

```sh
wb claims Q1089                     # every claim
wb claims Q1089 P31                 # only "instance of" claims
```

Run SPARQL — inline string, file, or with format flag:

```sh
wb sparql 'SELECT * WHERE { ?s ?p ?o } LIMIT 5'
wb sparql ./query.rq --format table
wb sparql ./query.rq --format csv > results.csv
```

Discover the property catalogue (you will need this — MaRDI has its own P-numbers, not Wikidata's):

```sh
wb props | jq                       # full property list with labels
wb props | jq 'to_entries | map(select(.value | test("cites"; "i"))) | from_entries'
```

## SPARQL prefixes

Wikidata-style prefixes work because MaRDI runs the same Wikibase Query Service:

```sparql
PREFIX wd:       <https://portal.mardi4nfdi.de/entity/>
PREFIX wdt:      <https://portal.mardi4nfdi.de/prop/direct/>
PREFIX wikibase: <http://wikiba.se/ontology#>
PREFIX bd:       <http://www.bigdata.com/rdf#>
PREFIX rdfs:     <http://www.w3.org/2000/01/rdf-schema#>
```

`wb sparql` injects sensible defaults, but inline queries that travel outside `wb` (e.g. via `curl`) need them.

## Read-only contract

Decline write requests (creating items, editing labels, adding claims, deleting). Tell the user this skill is read-only and point them at the Wikibase write API + an authenticated tool if they need it. Do not attempt writes via `curl` either.

## Example queries

Canonical SPARQL templates live in `references/queries.md`. Read that file when the user's request maps to one of:
- find items by label / partial match
- look up MaRDI items linked to an external ID (zbMATH, arXiv, swMATH, CRAN…)
- list items of a given class / "instance of" type
- count items by category
- explore an unfamiliar property's usage
- query MathModDB content (mathematical models, formulas, quantities, computational tasks) — see `references/mathmoddb.md`

If a query template references a property by name (e.g. "instance of"), resolve the actual `P<n>` for MaRDI via `wb props` before running — MaRDI's IDs do not match Wikidata's.

## When something goes wrong

- `wb` returns empty results → re-check `WB_SPARQL_ENDPOINT` is set; check the property ID exists in MaRDI (`wb props`).
- HTTP 4xx from SPARQL → the URL path matters: it's `/sparql`, not the Wikidata-style `/proxy/wdqs/bigdata/namespace/wdq/sparql`.
- Slow queries → MaRDI's endpoint has a query timeout. Add `LIMIT`, narrow the predicate, or bind a specific subject before fanning out.
