# Lessons learned

Every real problem hit while building this, what caused it, and how it was fixed. Kept as a troubleshooting log.

## 1. Docker: "Virtualization support not detected"
- **Symptom:** Docker Desktop refused to start, even though Task Manager showed *Virtualization: Enabled*.
- **Cause:** BIOS virtualisation was on, but the Windows features WSL and Virtual Machine Platform were off.
- **Fix:** enabled both with `dism.exe` in an **elevated** PowerShell (a non-admin shell returns *Error 740*), restarted, then `wsl --update`.

## 2. Reddit blocks and rate limits
- **`.json` endpoint → 403 "blocked by network security".** Switched to the public **RSS** feed (`/new/.rss`), which was accepted.
- **RSS → 429 Too Many Requests** after many test runs, and when both feeds fired at the same moment.
- **Fixes:** a 20 s Wait before the second feed; *Retry On Fail* on both RSS nodes; **pinning RSS output while testing** so test runs don't hit Reddit at all; waiting 30–60 min for the block to clear instead of retrying (each retry resets it).

## 3. Gemini errors
| Error | Cause | Fix |
|---|---|---|
| `503 high demand` | `gemini-flash-latest` alias is shared by everyone | Pinned a specific Flash-Lite model |
| `400 response modalities (TEXT) not supported` | Picked a `-tts` (text-to-speech) model from the list | Avoid models with `tts`, `image`, `audio`, `live`, `embedding` in the name |
| `429 quota exceeded, limit: 15` | Free tier = 15 requests/min | Batch Processing: 5 items per batch, 25 s between batches (~12/min) |

Note: n8n caps *Wait Between Tries* at 5000 ms, so batching — not retries — is what keeps you under the limit.

## 4. n8n wiring mistakes
- **Prompt read `{{ $json.chatInput }}` → undefined.** The Basic LLM Chain defaults to a Chat Trigger source; switched *Source for Prompt* to **Define below**.
- **Added the Auto-fixing Output Parser** instead of the **Structured Output Parser** (they sit next to each other in the list).
- **LLM output lost the title/link.** Re-attached them in an Edit Fields node with `$('Merge').item.json.title` (n8n's paired-item lookup).
- **Filter: "'1' is a string but was expecting a number".** Set the `score` field type to **Number** in Edit Fields.
- **A node got named `$('Merge1')`** — the expression was typed into the node *name*. Node names are plain labels.
- **Stray quote marks in the sheet.** Invisible tab characters before `{{` in expression boxes; Sheets quotes any cell containing a tab. Removed them and added `.trim()`.
- **Blank rows in the sheet.** The Sheets node had columns but no values mapped (`"value": {}` in the export).
- **Append-or-Update needed a match column** and would have overwritten manual notes. Switched to **Append** + a *previous executions* Remove Duplicates node.
- **Dedupe placed after the Filter** only protected the leads tab and still re-scored old posts. Moved it **directly after Merge**, before the LLM.

## 5. Lost the whole workflow after Ctrl+C
- **Cause:** in Git Bash, `-v ~/.n8n:/home/node/.n8n` was silently rewritten by MSYS path conversion. `docker inspect` showed:
  ```
  "Source":"C:\\Users\\This PC\\.n8n;C","Destination":"\\Program Files\\Git\\home\\node\\.n8n"
  ```
  Nothing was mounted at `/home/node/.n8n`, so all data lived inside the container — and the container was started with `--rm`, so stopping it deleted everything.
- **Fix:** a **named volume** plus `MSYS_NO_PATHCONV=1`:
  ```bash
  MSYS_NO_PATHCONV=1 docker run -d --name n8n --restart unless-stopped \
    -p 5678:5678 -v n8n_data:/home/node/.n8n n8nio/n8n
  ```
- **Recovered** by importing the exported workflow JSON and recreating two credentials (~10 min).
- **Lesson:** verify persistence with `docker inspect` *before* trusting a container with work, and keep exported JSON backups.

## 6. Google OAuth on self-hosted n8n
| Error | Fix |
|---|---|
| `Insufficient parameters for OAuth2 callback` | Self-hosted n8n needs your own Google Cloud OAuth client |
| `Invalid Origin: URIs must not contain a path` | The callback URL goes in **Authorized redirect URIs**, not *JavaScript origins* |
| `Error 403: access_denied` (app not verified) | Add your Gmail as a **Test user**; keep the app in Testing mode |
| `Gmail API has not been used in project …` | Enable the Gmail API in the same project, wait a few minutes |

## 7. n8n 2.x differences from older tutorials
- No *Active* toggle — workflows are **Published**, and edits need re-publishing.
- Autosave replaces the Save button in the editor.
- An **error workflow must itself be published** before it can be selected in another workflow's settings.
- Error Trigger only fires on production (scheduled) runs, not manual executions.

## 8. Prompt quality
The first prompt scored whether *software could help with the topic*, so hiring posts, research write-ups and idea-validation posts scored 7–8. Rewriting it to score **the poster's own current pain**, with explicit reject categories, dropped those to 1–3 while real owner-operator posts stayed at 8–10.
