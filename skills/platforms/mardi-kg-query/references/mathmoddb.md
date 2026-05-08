# MathModDB queries

MathModDB — the Mathematical Models Database, ~2,140 items at last check — was merged into the MaRDI Portal in early 2025 and now lives in the same Wikibase as the rest of the graph. There is no separate endpoint: query it through the same SPARQL service the rest of this skill uses (`https://query.portal.mardi4nfdi.de/sparql`).

The canonical way to scope a query to MathModDB content is to filter on the community property:

```sparql
?item wdt:P1495 wd:Q6534265 .   # P1495 = "community"; Q6534265 = "MathModDB"
```

## Verified core IDs

Counts and labels confirmed against the live endpoint. Properties and class IDs in MaRDI generally do **not** match Wikidata's; re-verify with `wb props` / `wb data` if you suspect drift.

| ID | Label | Use |
|---|---|---|
| `P1495` | community | scope filter (`Q6534265` = MathModDB) |
| `P31` | instance of | ontology class of an item |
| `P558` | named after | model/formula → person it's named for |
| `Q6534265` | MathModDB | community marker |
| `Q68663` | mathematical model | ontology class |
| `Q96183` | formula | ontology class |
| `Q6534237` | quantity | ontology class |
| `Q6534247` | computational task | ontology class |
| `Q6482820` | dimensionless quantity | ontology class |
| `Q6775552` | dimensional quantity | ontology class |
| `Q6481447` | ordinary differential equation | example formula sub-class |

`P31` and `P558` happen to match Wikidata's numbering here — treat that as coincidence, not a guarantee for other properties.

## M1. List MathModDB mathematical models

```sparql
SELECT ?model ?modelLabel ?desc WHERE {
  ?model wdt:P1495 wd:Q6534265 ;
         wdt:P31  wd:Q68663 .
  OPTIONAL { ?model schema:description ?desc . FILTER(LANG(?desc) = "en") }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
LIMIT 50
```

Swap `wd:Q68663` for `wd:Q96183` (formula), `wd:Q6534237` (quantity), `wd:Q6534247` (computational task), etc., to list other ontology classes.

## M2. Models named after a person

```sparql
SELECT ?model ?modelLabel ?person ?personLabel WHERE {
  ?model wdt:P1495 wd:Q6534265 ;
         wdt:P31  wd:Q68663 ;
         wdt:P558 ?person .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
```

## M3. Formulas of a specific type (e.g. ODEs)

```sparql
SELECT ?formula ?formulaLabel WHERE {
  ?formula wdt:P1495 wd:Q6534265 ;
           wdt:P31  wd:Q6481447 .   # ordinary differential equation
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
```

## M4. Ontology class distribution within MathModDB

Useful overview of what MathModDB actually contains:

```sparql
SELECT ?class ?classLabel (COUNT(?item) AS ?n) WHERE {
  ?item wdt:P1495 wd:Q6534265 ;
        wdt:P31  ?class .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
GROUP BY ?class ?classLabel
ORDER BY DESC(?n)
```

## Discovering MathModDB-specific predicates

The classes above are the *types* of items; the *relations* between them (model contains formula, formula uses quantity, model addresses research problem, …) live in MathModDB-specific properties. Find them with:

```sh
wb claims Q<some-mathmoddb-Q-id>          # see what predicates a real item uses
wb props | jq 'to_entries | map(select(.value | test("formula|quantity|model"; "i"))) | from_entries'
```

The Portal hosts a curated example page worth skimming for richer query patterns: <https://portal.mardi4nfdi.de/wiki/MathModDB/Example_SPARQL_queries>.

## See also

`queries.md` for the general MaRDI SPARQL templates (label search, external-ID lookup, class membership, etc.). Those work for MathModDB items too — just add the `wdt:P1495 wd:Q6534265` filter.
