# Complex Workflows — mardi-doip-cli

---

## 0. LLM provenance

Before creating or updating any item, ask: **was the content (label, description, claims, symbol descriptions) derived with the help of an LLM?**

If yes, add `P1642: Q7266517` (Claude Sonnet 4.6) as a **top-level claim** on the item. This one claim covers the entire item — there is no need to repeat P1642 as a qualifier on individual claim values. Setting it once at the item level signals that the item as a whole was produced by the LLM.

If only specific claim values were LLM-generated (e.g. a P1962 symbol description or a P1459 summary added later to an otherwise manually curated item), add P1642 as a **qualifier on those individual values** instead of at the item level.

Q7266517 is the MaRDI KG item for Claude Sonnet 4.6. Replace with the appropriate QID if a different model was used.

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
    "P1642": "Q7266517",
    "P989": "\\zeta(s) = \\prod_{p \\text{ prime}} \\frac{1}{1-p^{-s}}",
    "P983": [
      {"value": "\\zeta", "qualifiers": {"P1962": "Riemann zeta function"}},
      {"value": "s",      "qualifiers": {"P1962": "complex variable"}}
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

**P983 symbol format**: each symbol can be a bare string (notation only) or an object with qualifiers describing what it represents:

| Qualifier | P-ID | Type | When to use |
|---|---|---|---|
| symbol represents (string) | P1962 | string | free-text description when no KG concept item exists yet |
| symbol represents | P984 | item (QID) | QID of the KG concept item when it exists |

Bare strings and object form can be mixed in the same array. Default to P1962 for new formula items. Use P984 only when the symbol is a well-known concept that is likely already in the KG — see **Step 1a** below. LLM provenance is covered by the item-level P1642 claim (see §0) — no need to repeat it as a qualifier on individual symbols.


Full Formula property reference:

| Field | P-ID | Type | Multi | Required | Notes |
|---|---|---|---|---|---|
| MaRDI profile type | P1460 | item (QID) | no | **yes** — always `Q5981696` |
| mathExpression | P989 | math string | no | no | generic defining formula; preferred over P14 |
| symbol | P983 | math string | yes | no | symbol notation; add P1962 qualifier (string) or P984 qualifier (item QID) to indicate what it represents |
| definesSymbol | P3 | item (QID) | yes | no | symbols this formula defines |
| DLMF equation ID | P2 | string | no | no | e.g. `25.2.E2` — only for DLMF-sourced items |
| Wikidata QID | P12 | string | no | no | e.g. `Q187235` |
| namedAfter | P558 | item (QID) | yes | no | person the formula is named after |
| about | P1495 | item (QID) | yes | no | mathematical community or subject area |
| contains | P1560 | item (QID) | yes | no | sub-formulas; qualifier P560 (item QID) for the role |
| related URL | P1690 | string (url) | yes | no | external resources (e.g. DLMF page, Wikipedia) |
| description_long | P1459 | string | no | no | extended description, supports Markdown (see § 4) |

Note the returned `qid` — use it to verify and enrich the item.

### Step 1a — Search for symbol concept items (optional)

**Ask the user first.** This step is not standard — only perform it when the user explicitly wants to link symbols to existing KG concept items (e.g. because the formula uses well-known objects like the Hamiltonian operator, the Laplacian, or the Dirac delta function). For routine formula creation, P1962 free-text qualifiers are sufficient.

If the user says yes, search for each symbol before writing the JSON payload:

```bash
# Search without --type to find concept items of any kind (quantities, operators, etc.)
mardi-doip-cli --action search --query "Hamiltonian operator" --no-banner
```

Check the first result's snippet to confirm it is the right concept. If a matching item is found, replace the P1962 qualifier for that symbol with a P984 qualifier using the QID:

```json
"P983": [
  {"value": "\\hat{H}", "qualifiers": {"P984": "Q6534301"}},
  {"value": "\\psi",    "qualifiers": {"P1962": "wave function"}}
]
```

If no KG item is found for a symbol, fall back to P1962. Mixed arrays (some P984, some P1962) are fine — use whatever is available per symbol.

### Step 2 — Update symbol qualifiers on an existing item (optional)

If the formula item already exists without P984 qualifiers and you want to add or replace them, use `update` with `do_override: true`:

```bash
cat > /tmp/formula_update.json << 'ENDJSON'
{
  "claims": {
    "P983": [
      {"value": "\\zeta", "qualifiers": {"P1962": "Riemann zeta function"}},
      {"value": "s",      "qualifiers": {"P1962": "complex variable"}}
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

## 5. Create a publication item

**Goal**: Register a scholarly article in the MaRDI KG.

Both `P31=Q56887` and `P1460=Q5976449` are **required** for the item to be recognised as a MaRDI publication. All other claims are optional.

```bash
mardi-doip-cli --action create \
  --json '{
    "label": "Title of the paper",
    "claims": {
      "P31": "Q56887",
      "P1460": "Q5976449",
      "P21": "<arXiv ID, e.g. 2212.00168>",
      "P27": "<DOI>",
      "P43": "<author name string — use if no MaRDI person QID is known>",
      "P16": "<author QID — preferred over P43 when available>",
      "P28": "<date published, ISO 8601>",
      "P1450": ["<keyword1>", "<keyword2>"],
      "P1642": "Q7266517"
    }
  }' \
  --username DoipBot --password <pw>
```

Full publication property reference:

| Field | P-ID | Type | Multi | Required |
|---|---|---|---|---|
| instance of | P31 | item (QID) | no | yes — always `Q56887` |
| MaRDI profile type | P1460 | item (QID) | no | yes — always `Q5976449` |
| arXiv ID | P21 | string | no | no |
| DOI | P27 | string | no | no |
| author name string | P43 | string | no | one of P16/P43 |
| author | P16 | item (QID) | yes | one of P16/P43 |
| date published | P28 | time (ISO 8601) | no | no |
| arXiv subject classification | P22 | string | no | no |
| zbMATH keywords | P1450 | string | yes | no |
| journal / proceedings | P1433 | item (QID) | yes | no |
| publisher | P200 | item (QID) | yes | no |
| MSC codes | P226 | string | yes | no |
| license | P275 | item (QID) | yes | no |
| cited articles | P223 | item (QID) | yes | no |
| contains (formula/theorem) | P1560 | item (QID) | yes | no |
| LLM provenance | P1642 | item (QID) | no | if AI-assisted |

---

## 6. Type FDO lookup before UPDATE

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
