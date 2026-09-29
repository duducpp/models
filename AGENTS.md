# Syncing with upstream

This repo is a fork of `router-for-me/models` (**upstream**). The fork **mirrors** upstream and adds a small **overlay**. Every difference from upstream is recorded in the overlay ledger below.

## Overlay policy

The overlay holds only:
- the Daybreak models;
- models upstream lacks;
- fixes for invalid upstream configurations, each backed by proof such as a validator failure, an official source, or a CLIProxyAPI rejection.

Everything else follows upstream, including fields, IDs, section membership, and order. Once upstream carries an overlay item itself, remove that item from the overlay and from the ledger.

## Overlay ledger

| Item | Location | Reason |
| --- | --- | --- |
| `gpt-daybreak-blue-latest`, `gpt-daybreak-red-latest` | `models.json`: all four `codex-*` sections. `codex_client_models.json`: after `gpt-5.6-luna` | Fork addition from the official Codex catalog (`openai/codex`, `codex-rs/models-manager/models.json`) |
| `native_capabilities.web_search` removed | `claude/claude-opus-5-5`, `claude/claude-sonnet-5-5`, `xai/grok-4.7-build-fast` | Upstream asserts it without evidence, so `scripts/validate-native-capabilities.mjs` fails on upstream |
| Codex `web_search` table | `README.md`, `codex-*` rows | Each row lists that section's `web_search: true` IDs in catalog order; upstream's table lags its own data |

## Catalog files

| File | Shape | Entry key |
| --- | --- | --- |
| `models.json` | `{ "<section>": Model[] }` (`claude`, `codex-team`, `xai`, …) | `id` |
| `codex_client_models.json` | `{ "models": ClientModel[] }`, the Codex client template | `slug` |
| `devin_models.json` | `{ "devin": Model[] }` | `id` |

## Sync steps

Start from a clean working tree on `main`.

1. **Fetch upstream.** Add the remote only if `git remote -v` lacks it:
   ```sh
   git remote add upstream https://github.com/router-for-me/models
   git fetch upstream
   git log --oneline HEAD..upstream/main
   ```

2. **Mirror upstream.** Record upstream as a merge parent, then take its content:
   ```sh
   git merge -s ours --no-commit upstream/main
   git checkout upstream/main -- .
   ```
   Tracked files now equal upstream. Fork-only files such as this one are untouched.

3. **Re-apply the ledger**, item by item:
   - **Overlay entries**: copy each object verbatim from `git show HEAD:<file>` and insert it after the entry that precedes it in `HEAD`.
   - **`web_search` fixes**: run `node scripts/validate-native-capabilities.mjs`. For each failure, make the named entry match the expected value, which means deleting the field when the expected value is `undefined`. Repeat until the validator passes. The rules behind it:
     - `codex-*` entries need the boolean `supports_search_tool` of the same slug in `codex_client_models.json`.
     - All other entries need an exact section/ID match in `native-capabilities-evidence.json` (official HTTPS source) or in `gemini-native-search-declaration.json`.
     - Absence means unknown.
   - **README table**: rebuild the `codex-*` rows from the final `models.json`.
   - **Ledger**: update this file. Drop items that upstream now covers, and add new fixes or missing models together with their reason.

4. **Write JSON byte-compatibly.** Rewrite only the files that carry overlay items, using `JSON.stringify(data, null, 2)` (Python: `json.dumps(data, indent=2, ensure_ascii=False)`), in UTF-8 with LF line endings. Keep upstream's final newline as it is: `models.json` and `codex_client_models.json` end with one, `devin_models.json` does not. This makes `git diff upstream/main` show only the overlay.

5. **Validate**:
   ```sh
   node scripts/validate-native-capabilities.mjs
   node --test scripts/validate-native-capabilities.test.mjs
   git diff upstream/main --stat
   ```
   The sync is done when both checks pass and every hunk of `git diff upstream/main` maps to a ledger row or to this file.

6. **Commit** the merge, for example `git commit -m "chore: sync with upstream"`. Afterwards, `git log HEAD..upstream/main` stays empty until upstream moves again.
