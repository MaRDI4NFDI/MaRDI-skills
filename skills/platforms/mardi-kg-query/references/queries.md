# MaRDI SPARQL query templates

Templates use Wikidata-style prefixes (`wd:`, `wdt:`) which the MaRDI Query Service accepts.

**Property and class IDs in these templates are placeholders** — MaRDI's `P<n>` and `Q<n>` numbers do **not** match Wikidata's. Before running, look up the real IDs:

```sh
wb props | jq 'to_entries | map(select(.value | test("PROPERTY_NAME_HERE"; "i")))'
wb search --type item   "CLASS_NAME_HERE"
wb search --type property "PROPERTY_NAME_HERE"
```

Then substitute the resolved IDs into the query.

---

## 1. Search items by label substring

Free-text matching with case-insensitive `CONTAINS` over the English label.

```sparql
SELECT ?item ?itemLabel ?desc WHERE {
  ?item rdfs:label ?label .
  FILTER(LANG(?label) = "en" && CONTAINS(LCASE(?label), LCASE("riemann zeta")))
  OPTIONAL { ?item schema:description ?desc . FILTER(LANG(?desc) = "en") }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". ?item rdfs:label ?itemLabel. }
}
LIMIT 25
```

For interactive lookup, `wb search "riemann zeta"` is faster — use SPARQL when you need to combine the search with structural conditions.

## 2. Look up a MaRDI item by an external ID

Many MaRDI items reference external IDs (zbMATH document, arXiv id, swMATH software, CRAN package, DOI). Resolve the property first (`wb props | jq …`), then bind it:

```sparql
# Replace P_EXTERNAL with the MaRDI property for the external ID type
SELECT ?item ?itemLabel WHERE {
  ?item wdt:P_EXTERNAL "EXTERNAL_ID_VALUE" .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
```

Common external-ID property names to look up: `zbMATH`, `arXiv ID`, `swMATH ID`, `CRAN package`, `DOI`.

## 3. List items of a given class

Wikibase models class membership with "instance of" (Wikidata's P31; MaRDI uses a different P-number — resolve it).

```sparql
# Replace P_INSTANCE_OF and Q_CLASS with resolved MaRDI IDs
SELECT ?item ?itemLabel WHERE {
  ?item wdt:P_INSTANCE_OF wd:Q_CLASS .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
LIMIT 50
```

## 4. Count items by class

Useful for getting a sense of the graph's distribution.

```sparql
SELECT ?class ?classLabel (COUNT(?item) AS ?count) WHERE {
  ?item wdt:P_INSTANCE_OF ?class .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
GROUP BY ?class ?classLabel
ORDER BY DESC(?count)
LIMIT 20
```

## 5. Explore claims of a single item

Quick way to learn an item's structure when you don't know its schema yet.

```sparql
SELECT ?prop ?propLabel ?value ?valueLabel WHERE {
  wd:Q1089 ?p ?value .
  ?prop wikibase:directClaim ?p .
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
```

Or simpler from the CLI: `wb claims Q1089`.

## 6. Sample of items linked to a topic

When the user asks "what do you have on X", combining label search + outgoing links gives a richer answer than a label-only search.

```sparql
SELECT DISTINCT ?item ?itemLabel ?type ?typeLabel WHERE {
  ?item rdfs:label ?label .
  FILTER(LANG(?label) = "en" && CONTAINS(LCASE(?label), LCASE("TOPIC")))
  OPTIONAL { ?item wdt:P_INSTANCE_OF ?type }
  SERVICE wikibase:label { bd:serviceParam wikibase:language "en". }
}
LIMIT 50
```

## 7. Property usage frequency

How often a given property appears — useful for sanity checks ("does anyone actually use P123?").

```sparql
SELECT (COUNT(*) AS ?n) WHERE {
  ?s wdt:P_TARGET ?o .
}
```

---

## Tips

- **Always cap with `LIMIT`** while exploring. The endpoint has a server-side timeout; an unbounded query fans out fast.
- **Bind a subject first.** Queries that start from a known `wd:Q…` and walk outward are orders of magnitude cheaper than `?s ?p ?o` patterns.
- **`SERVICE wikibase:label`** auto-resolves labels for any `?varLabel` in the SELECT — cheaper than `OPTIONAL { ?var rdfs:label … }`.
- **Output formats:** `wb sparql … --format table` for terminal reading, `--format csv` to pipe into analysis, `--format json` (default) for programmatic use.

---

For MathModDB-specific queries (mathematical models, formulas, quantities, computational tasks), see `mathmoddb.md` in this folder.
