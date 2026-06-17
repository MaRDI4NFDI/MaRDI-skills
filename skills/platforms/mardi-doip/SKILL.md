---
name: mardi-doip
description: Operate the mardi-doip-cli to interact with the MaRDI DOIP server and knowledge graph. Use for CRUD and search operations on FDOs and knowledge graph items, and downloading files or components. Trigger whenever the user mentions DOIP, FDO, or the MaRDI portal.
---

# MaRDI DOIP Skill

You are operating `mardi-doip-cli`, a binary client for the MaRDI DOIP server.

Read `references/cli-reference.md` before running any command — it has the full action reference, critical gotchas, and known wrong behaviour to avoid.

Read `references/workflows.md` when the user asks for a multi-step task (label-sync, create+upload, type-FDO-then-update).

## Locating the binary

Run `which mardi-doip-cli` first. If not found or on PATH, ask the user where the binary is before proceeding. Suggest to download it from https://github.com/MaRDI4NFDI/mardi_doip_server/releases .

## Credentials

Write operations (`update`, `create`) require MediaWiki bot credentials.

1. Check `$DOIP_USERNAME` and `$DOIP_PASSWORD` — if both are set, use them silently.
2. If either is missing, ask: *"Which bot account should I use, and can you provide the password? (Or set DOIP_USERNAME / DOIP_PASSWORD in the environment to avoid being asked.)"*
3. Never log or echo passwords in command output. Pass them via `--username` / `--password` flags.
4. The designated service account for automated writes is `DoipBot@DoipBot`.

## Plan-then-confirm approach

Before running any command, state:
- What you are about to do and why
- The exact command with all arguments filled in
- Any irreversible side-effects (note: `purge` only evicts cache — it does NOT delete data)

Wait for the user to confirm before executing. For multi-step workflows, show the full plan upfront, then confirm each step individually as you reach it.

## Handling output

- Metadata responses are JSON — summarise the relevant fields rather than dumping the full blob.
- Component downloads write to a file; report the path and size when done.
- Search results: show `qid`, `title`, and `snippet` in a table.
- On error, quote the error message verbatim and suggest the likely cause (auth failure, wrong QID format, component not found, conflict on update).
