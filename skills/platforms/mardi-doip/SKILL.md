---
name: mardi-doip
description: Operate the mardi-doip-cli to interact with the MaRDI DOIP server and knowledge graph. Use for CRUD and search operations on FDOs and knowledge graph items, and downloading files or components. Trigger whenever the user mentions DOIP, FDO, or the MaRDI portal.
---

# MaRDI DOIP Skill

You are operating `mardi-doip-cli`, a binary client for the MaRDI DOIP server.

Read `references/cli-reference.md` before running any command — full action reference, critical gotchas, and known wrong behaviour.

Read `references/workflows.md` before any `create` or `update` operation — it lists required fields per item type and common multi-step patterns. Missing a required claim (e.g. `P1460`) silently breaks FDO routing and is hard to detect after the fact.

## Locating the binary

Run `which mardi-doip-cli` first. If not found, ask the user where the binary is. Suggest downloading from https://github.com/MaRDI4NFDI/mardi_doip_server/releases.

## Credentials

Write operations require MediaWiki bot credentials. Check `$DOIP_USERNAME` / `$DOIP_PASSWORD` first — if set, use silently. Otherwise ask: *"Which bot account and password? (Or set DOIP_USERNAME / DOIP_PASSWORD to avoid being asked.)"* Pass via `--username` / `--password`; never echo passwords. The username is a plain bot name (e.g. `DoipBot`), not `User@BotName` format.

## Plan-then-confirm

Before each command, state what you'll do, show the exact command, and note irreversible effects. Wait for confirmation. For multi-step workflows, show the full plan upfront, then confirm each step as you reach it.

## Output

- JSON responses: summarise key fields, don't dump the blob.
- Search results: `qid`, `title`, `snippet` in a table.
- Errors: quote verbatim, suggest likely cause.
- Always use `--no-banner` when piping or parsing output.
