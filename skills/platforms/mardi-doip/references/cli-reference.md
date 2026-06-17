# CLI Reference — mardi-doip-cli

## Global flags

| Flag | Default | Description |
|------|---------|-------------|
| `--host` | `doip.portal.mardi4nfdi.org` | Server hostname |
| `--port` | `3567` | Server port |
| `--no-tls` | off | Disable TLS (use for local dev) |
| `--secure` | off | Verify TLS certificate (only for non-self-signed certs) |

---

## Actions

### demo
Quick smoke-test: runs `hello` then retrieves metadata for `--object-id` (default `Q123`).
```bash
mardi-doip-cli --action demo
mardi-doip-cli --action demo --object-id Q424905
```

---

### hello
Check server version and capabilities. No extra flags.
```bash
mardi-doip-cli --action hello
```
Response includes `server_version`, `availableOperations`, and `typeRegistry`.

---

### list_ops
List operations the server advertises (useful for discovering custom workflows).
```bash
mardi-doip-cli --action list_ops
```

---

### retrieve

**Metadata only** (no `--component`):
```bash
mardi-doip-cli --action retrieve --object-id Q424905
```
Response structure:
- `kernel["fdo:hasComponent"]` — list of component descriptors (check for `"componentId": "rocrate.zip"`)
- `profile["name"]` — the item's current label
- `kernel["digitalObjectType"]` — URL of the type FDO (e.g. `…/types/ScholarlyArticle`)

**Type FDO** (property mappings for a schema type):
```bash
mardi-doip-cli --action retrieve --object-id types/ScholarlyArticle
# or full URI:
mardi-doip-cli --action retrieve --object-id https://fdo.portal.mardi4nfdi.de/fdo/types/ScholarlyArticle
```
Available type IDs: `ScholarlyArticle`, `Dataset`, `Workflow`, `Person`, `SoftwareApplication`, `SoftwareSourceCode`.
Response includes `propertyMappings` — consult this before building an UPDATE payload (maps Schema.org field names → Wikibase P-IDs).

**Download a component**:
```bash
mardi-doip-cli --action retrieve --object-id Q424905 --component rocrate.zip --output rocrate.zip
```
⚠️ `--output` must be a **file path**, not a directory. The binary calls `open(path, "wb")` directly — passing `.` will raise `IsADirectoryError`.

---

### update

**Property mode** — update label, description, or Wikibase claims:
```bash
mardi-doip-cli --action update --object-id Q424905 \
  --username User@BotName --password <pw> \
  --properties '{"label": "New title"}'

mardi-doip-cli --action update --object-id Q424905 \
  --username User@BotName --password <pw> \
  --properties '{"claims": {"P16": ["Q111", "Q482723"]}, "do_override": true}'
```

`--properties` keys:

| Key | Type | Notes |
|-----|------|-------|
| `label` | string | New English label |
| `description` | string | New English description |
| `claims` | object | `P-ID → value` or `P-ID → [values]` |
| `do_override` | bool | Replace existing claim values (default false) |

**Conflict guard**: if a property already has values and `do_override` is absent, the server returns a 409 with the existing values. Retrieve current values, merge, then resubmit with `"do_override": true`.

**PID formats accepted**: bare (`P16`), prefixed (`wdt:P16`), or full URL — all normalised server-side to bare form.

**Component mode** — upload a file:
```bash
mardi-doip-cli --action update --object-id Q424905 \
  --component paper.pdf --input paper_v2.pdf --media-type application/pdf \
  --username User@BotName --password <pw>
```
`--properties` and `--input`/`--component` are mutually exclusive.

**Credentials via env vars**:
```bash
export DOIP_USERNAME=User@BotName
export DOIP_PASSWORD=<pw>
mardi-doip-cli --action update --object-id Q424905 --properties '{"label": "New title"}'
```

---

### purge

**Cache eviction only.** Removes the object's manifest from the server's in-memory cache, forcing a fresh fetch on the next access. Does NOT delete the object or any components.

```bash
mardi-doip-cli --action purge --object-id Q424905
```
Call this after an `update` if you want the next `retrieve` to see the updated data immediately.

**Type cache eviction.** Use the `types/` prefix to evict a type FDO from the DOIP server's in-memory type cache — useful after updating `type_registry.py` in the FDO server without restarting the DOIP pod:

```bash
mardi-doip-cli --action purge --object-id types/Workflow
```

---

### create

Create a new Wikibase item. Requires bot credentials.

**Raw format** (specify P-IDs directly):
```bash
mardi-doip-cli --action create \
  --json '{"label": "Jane Doe", "claims": {"P31": "Q57162", "P1460": "Q5976445"}}' \
  --username User@BotName --password <pw>
```
- `label` — required
- `description` — optional
- `claims` — map of bare P-IDs (`P<number>`) to values. Item → QID string; string/external-id → plain string; multi-value → JSON array (e.g. `"P31": ["Q68657", "Q6830884"]`).

Response:
```json
[{"operation": "create", "status": "created", "qid": "Q7260359"}]
```

---

### search

```bash
mardi-doip-cli --action search --type workflow --limit 20
mardi-doip-cli --action search --query "Conrad" --type person
mardi-doip-cli --action search --query "10.1371/JOURNAL.PONE.0306704"
```

| Flag | Description |
|------|-------------|
| `--query` | Fulltext search string |
| `--type` | MaRDI profile type name or raw QID |
| `--limit` | 1–50, default 10 |

At least one of `--query` or `--type` is required.

Known type names: `workflow`, `dataset`, `person`, `publication`, `software`, `model`, `algorithm` (and more — check `doip_shared/constants.py` `MARDI_PROFILE_TYPES`).

⚠️ `--type` filtering matches against a specific profile type QID. Items created with a different or missing `P1460` value will not appear. If a search with `--type` returns no results, retry without `--type` to do a fulltext search across all item types.

Each result has: `qid`, `title`, `snippet`, `timestamp`.

---

### list

There is no explicit list command. But, when using search with only the `--type` parameter, this behaves like a list, as in, it will list all items of the given type. 

---

## Claims reference for persons

| Property | Datatype | Example | Meaning |
|----------|----------|---------|---------|
| `P31` | item | `Q57162` | instance of (researcher type) |
| `P1460` | item | `Q5976445` | profile type (author profile) |
| `P676` | external-id | `anderson.james-h` | zbMATH author ID |
| `P673` | external-id | `a/JamesHAnderson` | arXiv author ID |
| `P12` | external-id | `Q27983415` | Wikidata QID |

P-IDs are local to the MaRDI Wikibase and may differ from Wikidata.

---

## Key gotchas

1. `purge` = cache eviction only, not deletion. It removes the manifest (or type FDO when using `types/` prefix) from the server's in-memory cache — the object and its components are untouched.
2. `--output` for retrieve must be a file path, not a directory. Passing `.` raises `IsADirectoryError`.
3. UPDATE `--properties` uses the same credential env vars as create. Both fail clearly if credentials are missing.
4. The `hello` response includes `"describe"` in `availableOperations` — this is a server-internal operation not exposed via the CLI.
