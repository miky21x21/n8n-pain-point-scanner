# Setup guide

Tested on Windows 10/11 with Git Bash (MINGW64), Docker Desktop 4.x and n8n 2.x.

## 1. Prerequisites

1. **Virtualisation enabled** — Task Manager → Performance → CPU → *Virtualization: Enabled*.
2. **WSL2** — in an **Administrator** PowerShell:
   ```powershell
   dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
   dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
   ```
   Restart, then:
   ```powershell
   wsl --set-default-version 2
   wsl --update
   ```
3. **Docker Desktop** (AMD64 installer for Intel/AMD CPUs). Settings → General → *Start Docker Desktop when you sign in*.

## 2. Run n8n with persistent storage

```bash
MSYS_NO_PATHCONV=1 docker run -d --name n8n --restart unless-stopped \
  -p 5678:5678 -v n8n_data:/home/node/.n8n n8nio/n8n
```

**Verify the mount before doing anything else:**

```bash
docker inspect n8n --format '{{json .Mounts}}'
```

It must show `"Name":"n8n_data"` and `"Destination":"/home/node/.n8n"`. See [LESSONS-LEARNED.md](LESSONS-LEARNED.md#5-lost-the-whole-workflow-after-ctrlc) for why.

Open `http://localhost:5678` and create the owner account.

## 3. Google Sheet

Create a spreadsheet with two tabs:

- `Sheet1`: `title | link | posted | score | reason | problem | logged | status | action_taken | notes | follow_up_date`
- `All posts`: `title | link | posted | score | reason | problem | logged`

Optional: Data → Data validation on `Sheet1!H2:H` → Dropdown with the status values.

Copy the Sheet ID from its URL (`/spreadsheets/d/<THIS PART>/edit`).

## 4. Credentials

**Gemini** — create a free key at <https://aistudio.google.com/apikey>.

**Google OAuth client** (needed because self-hosted n8n can't use n8n's managed Google sign-in):

1. <https://console.cloud.google.com> → new project.
2. APIs & Services → Library → enable **Google Sheets API**, **Google Drive API**, and **Gmail API** (for alerts).
3. Google Auth Platform → create consent screen → Audience **External** → **Test users → add your own Gmail address**.
4. Clients → Create → **Web application**. Leave *Authorized JavaScript origins* empty. Under **Authorized redirect URIs** add:
   ```
   http://localhost:5678/rest/oauth2-credential/callback
   ```
5. Copy the Client ID and Secret into n8n's Google Sheets and Gmail OAuth2 credentials → *Sign in with Google* → Advanced → continue past the unverified-app warning.

## 5. Import the workflows

For each file in `workflows/`: n8n → Create Workflow → ⋯ → **Import from File**.

- In **Pain Point Scanner**: attach the Gemini credential to *Google Gemini Chat Model*; attach the Sheets credential to *Log leads* and *Log all posts*; replace `YOUR_GOOGLE_SHEET_ID` with your Sheet ID (or pick the sheet from the list).
- In **Scanner error alerts**: attach the Gmail credential and replace `you@example.com`.

## 6. Test, then publish

1. Run **Pain Point Scanner** once manually (~5 min; Gemini calls are batched 5 per 25 s to respect the free-tier limit).
2. Check both tabs.
3. **Publish** *Scanner error alerts* first, then in *Pain Point Scanner* → ⋯ → Settings → **Error workflow** → select it.
4. **Publish** *Pain Point Scanner*. Edits only go live after re-publishing.

## 7. Keep it running

- Windows → Power → sleep **Never** when plugged in.
- Docker Desktop must be running, which requires signing in to Windows after a reboot.

## 8. Backups

Workflow JSON: ⋯ → Download in each workflow.

Full volume (workflows, credentials, history):

```bash
mkdir -p /c/n8n-backups
MSYS_NO_PATHCONV=1 docker run --rm -v n8n_data:/data -v C:/n8n-backups:/backup \
  alpine tar czf /backup/n8n_data_$(date +%Y-%m-%d).tar.gz -C /data .
```

**Never run `docker volume rm n8n_data`** — that is the only command here that deletes your data.
