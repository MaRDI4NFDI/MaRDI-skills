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
      "P1961": "Long description of the workflow."
    }
  }' \
  --username DoipBot@DoipBot --password <pw>
```

`P31` and `P1460` are **required** for the item to be recognised as a MaRDI workflow. The remaining claims are optional. Full Workflow property reference:

| Field | P-ID | Type | Multi | Required |
|---|---|---|---|---|
| instance of | P31 | item (QID) | yes | yes — always `["Q68657", "Q6830884"]` |
| MaRDI profile type | P1460 | item (QID) | no | yes — always `Q6534216` |
| creator | P16 | item (QID) | yes | one of P16/P43 |
| author name string | P43 | string | yes | one of P16/P43 |
| datePublished | P28 | time (ISO 8601) | no | no |
| license | P163 | item (QID) | yes | no |
| citation | P286 | item (QID) | yes | no |
| uses | P557 | item (QID) | yes | no |
| zenodoId | P227 | string | no | no |
| description_long | P1961 | string | no | no |

Note the returned `qid` — you will need it in subsequent steps.

### Step 2 — Upload the RO-Crate component

```bash
mardi-doip-cli --action update \
  --object-id <new-QID> \
  --component rocrate.zip \
  --input /path/to/rocrate.zip \
  --media-type application/zip \
  --username DoipBot@DoipBot --password <pw>
```

### Step 3 — Verify

```bash
mardi-doip-cli --action retrieve --object-id <new-QID>
```

Confirm that `kernel["fdo:hasComponent"]` contains an entry with `"componentId": "rocrate.zip"`.

### Step 4 — Purge cache (optional)

If the item was just created the cache won't have a stale entry, but if you ran a retrieve in between you can force a fresh fetch:

```bash
mardi-doip-cli --action purge --object-id <new-QID>
```

---

## 3. Type FDO lookup before UPDATE

**Why**: The UPDATE handler expects bare Wikibase P-IDs as property keys (e.g. `{"P28": "2024-01-15"}`), not Schema.org names. Retrieve the type FDO first to get the correct P-IDs.

```bash
# Retrieve the type FDO for ScholarlyArticle
mardi-doip-cli --action retrieve --object-id types/ScholarlyArticle
```

`propertyMappings` example (abbreviated):

```json
{
  "author":        {"pid": "P16",  "type": "item",   "multi": true},
  "datePublished": {"pid": "P28",  "type": "time",   "multi": false},
  "identifier":    {"pid": "P27",  "type": "string", "multi": false, "note": "DOI"}
}
```

Then build the UPDATE payload using the P-IDs:

```bash
mardi-doip-cli --action update --object-id <QID> \
  --username User@BotName --password <pw> \
  --properties '{"claims": {"P28": "2024-01-15"}}'
```
