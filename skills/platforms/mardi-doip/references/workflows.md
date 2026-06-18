# Complex Workflows — mardi-doip-cli

---

## 1. Create a workflow item and upload an RO-Crate

**Goal**: Register a new computational workflow in the MaRDI KG and attach its RO-Crate package.

### Step 1 — Create the workflow item

```bash
mardi-doip-cli --action create \
  --json '{
    "label": "Reproduce results from: Foo, Doe et al. (2024)",
    "claims": {
      "P31": ["Q68657", "Q6830884"],
      "P1460": "Q6534216",
      "P16": "<author-QID>",        
      "P43": "<author name string — use if QID of the author is unknown; one of P16/P43 must be set>",
      "P28": "<Date of workflow creation>",
      "P163": "<license-QID>",
      "P286": "<citation-QID>",
      "P557": "<uses-QID>",
      "P227": "<zenodo-id>",
      "P1459": "Long description of the workflow."
    }
  }' \
  --username DoipBot@DoipBot --password <pw>
```

`P31` and `P1460` are **required** for the item to be recognised as a MaRDI workflow. The remaining claims are optional. Full Workflow property reference:

| Field | P-ID | Type | Multi | Required |
|---|---|---|---|---|
| instance of | P31 | item (QID) | yes | yes — always `["Q68657", "Q6830884"]` |
| MaRDI profile type | P1460 | item (QID) | no | yes — always `Q6534216` |
| author | P16 | item (QID) | yes | one of P16/P43 |
| author name string | P43 | string | no | one of P16/P43 |
| datePublished | P28 | time (ISO 8601) | no | no |
| license | P163 | item (QID) | yes | no |
| citation | P286 | item (QID) | yes | no |
| uses | P557 | item (QID) | yes | no |
| zenodoId | P227 | string | no | no |
| description_long | P1459 | string | no | no |

Note the returned `qid` — you will need it in subsequent steps.

### Step 2 — Link software dependencies (optional)

For each software package or dataset the workflow uses, search the KG and link it via `P557`. Always search without `--type` first to avoid missing items with non-standard profile types.

```bash
# Search for a package
mardi-doip-cli --action search --query "NumPy" --limit 5

# Link it to the workflow (P557 is multi-value — include all QIDs together)
mardi-doip-cli --action update --object-id <new-QID> \
  --properties '{"claims": {"P557": ["<qid1>", "<qid2>"]}}' \
  --username DoipBot --password <pw>
```

If a package is not yet in the KG, create a software item first (see **§ 2. Create a software item**) and then link it.

### Step 3 — Upload the RO-Crate component

```bash
mardi-doip-cli --action update \
  --object-id <new-QID> \
  --component rocrate.zip \
  --input /path/to/rocrate.zip \
  --media-type application/zip \
  --username DoipBot@DoipBot --password <pw>
```

### Step 4 — Verify

```bash
mardi-doip-cli --action retrieve --object-id <new-QID>
```

Confirm that `kernel["fdo:hasComponent"]` contains an entry with `"componentId": "rocrate.zip"`.

### Step 5 — Purge cache (optional)

If the item was just created the cache won't have a stale entry, but if you ran a retrieve in between you can force a fresh fetch:

```bash
mardi-doip-cli --action purge --object-id <new-QID>
```

---

## 2. Create a software item

**Goal**: Register a software repository (e.g. a GitHub repo referenced by a workflow) as an item in the MaRDI KG.

```bash
mardi-doip-cli --action create \
  --json '{
    "label": "my-software-repo",
    "claims": {
      "P31": "Q56614",
      "P1460": "Q5976450",
      "P339": "https://github.com/owner/my-software-repo",
      "P43": "<author name string — use if no MaRDI person QID is known>"
    }
  }' \
  --username DoipBot --password <pw>
```

`P31` and `P1460` are **required** to mark the item as a MaRDI software item. Full software property reference:

| Field | P-ID | Type | Multi | Required |
|---|---|---|---|---|
| instance of | P31 | item (QID) | no | yes — always `Q56614` |
| MaRDI profile type | P1460 | item (QID) | no | yes — always `Q5976450` |
| codeRepository | P339 | url | no | no |
| author name string | P43 | string | no | no |
| author | P16 | item (QID) | yes | no |
| license | P163 | item (QID) | yes | no |
| softwareVersion | P132 | string | no | no |
| programmingLanguage | P114 | item (QID) | yes | no |
| softwareHeritageId | P1454 | string | no | no |

To link the new software item to a workflow, update the workflow with `P557`:

```bash
mardi-doip-cli --action update --object-id <workflow-QID> \
  --properties '{"claims": {"P557": "<software-QID>"}}' \
  --username DoipBot --password <pw>
```

> **Tip**: If the author's real name is unknown, look it up from a GitHub username:
> ```bash
> gh api users/<github-login> --jq '.name'
> ```

---

## 3. Create a formula item

**Goal**: Register a mathematical formula in the MaRDI KG so it gets a Formula FDO.

The trigger is `P1460=Q5981696` — this is what causes the FDO server to route the item to the Formula profile builder. There is no required `P31` value (formula items are inconsistently typed across import sources).

### Step 1 — Create the item

For formulas with LaTeX expressions or symbol qualifiers, write the payload to a JSON file and use `--json @file` to avoid shell escaping issues:

```bash
cat > /tmp/formula.json << 'ENDJSON'
{
  "label": "Euler product formula",
  "description": "Representation of the Riemann zeta function as a product over primes",
  "claims": {
    "P1460": "Q5981696",
    "P989": "\\zeta(s) = \\prod_{p \\text{ prime}} \\frac{1}{1-p^{-s}}",
    "P983": [
      {"value": "\\zeta", "qualifiers": {"P984": "<QID of the concept the symbol represents>"}},
      {"value": "s",      "qualifiers": {"P984": "<QID of the concept the symbol represents>"}}
    ],
    "P558": "<QID of the person the formula is named after, if any>",
    "P1495": "<QID of the mathematical community, if any>",
    "P1459": "Long-form description of the formula and its context.",
    "P12": "<Wikidata QID, e.g. Q187235>",
    "P2": "<DLMF equation ID, e.g. 25.2.E2 — only if the formula appears in DLMF>"
  }
}
ENDJSON

mardi-doip-cli --action create \
  --json @/tmp/formula.json \
  --username DoipBot@DoipBot --password <pw>
```

**P983 symbol format**: each symbol can be a bare string (notation only) or an object `{"value": "...", "qualifiers": {"P984": "<concept-QID>"}}`. P984 links the notation to the KG item representing the concept the symbol stands for. Bare strings and object form can be mixed in the same array.

Full Formula property reference:

| Field | P-ID | Type | Multi | Required | Notes |
|---|---|---|---|---|---|
| MaRDI profile type | P1460 | item (QID) | no | **yes** — always `Q5981696` |
| mathExpression | P989 | math string | no | no | generic defining formula; preferred over P14 |
| symbol | P983 | math string | yes | no | symbol notation; add P984 qualifier (item QID) to indicate what it represents |
| definesSymbol | P3 | item (QID) | yes | no | symbols this formula defines |
| DLMF equation ID | P2 | string | no | no | e.g. `25.2.E2` — only for DLMF-sourced items |
| Wikidata QID | P12 | string | no | no | e.g. `Q187235` |
| namedAfter | P558 | item (QID) | yes | no | person the formula is named after |
| about | P1495 | item (QID) | yes | no | mathematical community or subject area |
| contains | P1560 | item (QID) | yes | no | sub-formulas; qualifier P560 (item QID) for the role |
| related URL | P1690 | string (url) | yes | no | external resources (e.g. DLMF page, Wikipedia) |
| description_long | P1459 | string | no | no | extended description, supports Markdown (see § 4) |

Note the returned `qid` — use it to verify and enrich the item.

### Step 2 — Update symbol qualifiers on an existing item (optional)

If the formula item already exists without P984 qualifiers and you want to add or replace them, use `update` with `do_override: true`:

```bash
cat > /tmp/formula_update.json << 'ENDJSON'
{
  "claims": {
    "P983": [
      {"value": "\\zeta", "qualifiers": {"P984": "<concept-QID>"}},
      {"value": "s",      "qualifiers": {"P984": "<concept-QID>"}}
    ]
  },
  "do_override": true
}
ENDJSON

mardi-doip-cli --action update --object-id <QID> \
  --properties @/tmp/formula_update.json \
  --username DoipBot --password <pw>
```

**Qualifier clearing**: passing `{"value": "...", "qualifiers": {}}` (empty qualifiers dict) with `do_override: true` explicitly removes all qualifiers from that symbol claim. A bare string `"..."` leaves any existing qualifiers untouched.

### Step 3 — Verify

```bash
mardi-doip-cli --action retrieve --object-id <new-QID>
```

Confirm that `kernel["digitalObjectType"]` ends with `/Formula` and that `profile["mathExpression"]` is populated.

### Step 4 — Search

```bash
mardi-doip-cli --action search --type formula --limit 10
mardi-doip-cli --action search --query "Euler" --type formula
```

---

## 4. Formatting P1459 (description_long) for portal rendering

P1459 values are rendered on the MaRDI portal via a `markdownToMediawiki` Lua function. Use the following conventions to produce well-structured output:

| Element | Syntax | Notes |
|---|---|---|
| Paragraph break | `\N\N` | Use between paragraphs and around headings |
| Line break | `\N` | Single newline — use between list items |
| Bold | `**text**` | Use for tool/library/step names |
| Section heading | `### Heading` | Level 3 (small) — surround with `\N\N` on both sides |
| Link | `[text](url)` | Renders as MediaWiki external link |

⚠️ Backticks (`` `code` ``) are **not** converted — avoid them; use `**bold**` for inline tool names instead.

**Example** (as stored in P1459):
```
A **Snakemake** workflow that reproduces the results of Doe et al. (2024).\N\NAll dependencies are pinned in a conda environment file.\N\N### Main steps\N\N**step_one** — fetches input data from the MaRDI data store.\N\N**step_two** — runs the analysis script and renders the output figure.
```

---

## 5. Type FDO lookup before UPDATE

**Why**: The UPDATE handler expects bare Wikibase P-IDs as property keys (e.g. `{"P28": "2024-01-15"}`), not Schema.org names. Retrieve the type FDO first to get the correct P-IDs.

```bash
# Retrieve the type FDO for ScholarlyArticle
mardi-doip-cli --action retrieve --object-id types/ScholarlyArticle
```

`propertyMappings` example (ScholarlyArticle, abbreviated):

```json
{
  "author":                 {"pid": "P16",   "type": "item",   "multi": true},
  "datePublished":          {"pid": "P28",   "type": "time",   "multi": false},
  "identifier":             {"pid": "P27",   "type": "string", "multi": false, "note": "DOI"},
  "about":                  {"pid": "P226",  "type": "string", "multi": true,  "note": "MSC codes"},
  "keywords":               {"pid": "P1450", "type": "string", "multi": true,  "note": "zbMATH keywords"},
  "identifier/zbmath-de":   {"pid": "P1451", "type": "string", "multi": false},
  "identifier/zbmath-open": {"pid": "P225",  "type": "string", "multi": false},
  "arxivId":                {"pid": "P21",   "type": "string", "multi": false},
  "arXivClassification":    {"pid": "P22",   "type": "string", "multi": false},
  "relatedLink":            {"pid": "P1643", "type": "item",   "multi": true,  "note": "recommended articles"}
}
```

> **`identifier` in the FDO profile**: when a publication has more than one identifier (DOI, arXiv, zbMATH-DE, zbMATH-Open), `profile["identifier"]` is a JSON array of `PropertyValue` objects. With a single identifier it is a plain object. Consumers should handle both forms.

Then build the UPDATE payload using the P-IDs:

```bash
mardi-doip-cli --action update --object-id <QID> \
  --username User@BotName --password <pw> \
  --properties '{"claims": {"P28": "2024-01-15"}}'
```

---

## 6. Link mathematical content items to a publication

**Goal**: Record which formulas, theorems, proofs, or other mathematical content items appear in a paper so the publication FDO lists them under `hasPart`.

The link goes on the **publication item** (not the content item). P1560 is used — the same property that Model items use to enumerate their components. Currently formulas (P1460=Q5981696) are the primary use case, but the same pattern applies to any future content type (theorems, proofs, lemmas, etc.).

```bash
# Single formula
mardi-doip-cli --action update --object-id <paper-QID> \
  --properties '{"claims": {"P1560": "<formula-QID>"}}' \
  --username DoipBot@DoipBot --password <pw>

# Multiple formulas at once
mardi-doip-cli --action update --object-id <paper-QID> \
  --properties '{"claims": {"P1560": ["<QID1>", "<QID2>", "<QID3>"]}, "do_override": true}' \
  --username DoipBot@DoipBot --password <pw>
```

After the update the publication FDO will contain:

```json
"hasPart": [
  {"@id": "https://portal.mardi4nfdi.de/entity/<QID1>"},
  {"@id": "https://portal.mardi4nfdi.de/entity/<QID2>"}
]
```

**Reverse lookup** — find all papers a content item appears in:

```bash
mardi-doip-cli --action search --type publication --query "<item-QID>"
```

No qualifier on P1560 is needed (unlike Model → Formula, which uses P560 for roles).
