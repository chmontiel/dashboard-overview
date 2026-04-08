# Grafana Dashboard Automation

Automates creating client folders and deploying dashboards to Grafana via the API.

## How It Works

The base dashboard (`Client-Overview.json`) uses **library panels** so that any panel changes propagate to all client dashboards automatically. These scripts copy the base dashboard, customize it for a specific client (title, subscription, hidden variables), and upload it to the correct folder in Grafana.

## Directory Structure

```
dashboards/
  templates/              ← Source of truth (never modified by scripts)
    Client-Overview.json
  output/                 ← Auto-generated client copies (gitignored)
  payloads/               ← API request bodies for debugging (gitignored)
tools/
  env.base.ps1            ← Environment variables (edit this)
  config.ps1              ← Builds config from env vars
  create-folder.ps1       ← Creates client subfolder in Grafana
  update_dashboard.py     ← Copies + customizes dashboard JSON
  send-dashboard.ps1      ← Uploads dashboard to Grafana
  run.ps1                 ← Runs everything in order
```

## Prerequisites

- **PowerShell** (Windows PowerShell or PowerShell 7)
- **Python 3** (for `update_dashboard.py`)
- A **"Clients" folder** must already exist in Grafana — the scripts create subfolders inside it
- A **Grafana service account token** with at least Editor privileges

## Setup

### 1. Place your base dashboard

Put your `Client-Overview.json` in the `dashboards/templates/` folder. This file is never modified by the scripts.

### 2. Edit environment variables

Open `tools/env.base.ps1` and fill in your values:

```powershell
$env:Grafana_Url     = "https://your-instance.grafana.azure.com"
$env:Token           = "glsa_your_service_account_token"
$env:Client_Name     = "AcmeCorp"
$env:Subscription    = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
$env:Namespace       = "default"
$env:Dashboards_Dir  = "C:\Users\YourName\project\kql-grafna\dashboards"
```

| Variable | Description |
|---|---|
| `Grafana_Url` | Base URL of your Grafana instance (no trailing slash) |
| `Token` | Service account token with Editor+ privileges |
| `Client_Name` | Display name of the client (e.g. "AcmeCorp", "AzBlue") |
| `Subscription` | The client's Azure subscription ID |
| `Namespace` | Grafana namespace, usually "default" |
| `Dashboards_Dir` | Full path to the `dashboards/` directory on your machine |

### 3. Run

```powershell
cd tools/
.\run.ps1
```

## What Each Step Does

The script runs 5 steps in order:

1. **Load env vars** — reads `env.base.ps1`
2. **Build config** — derives folder names, file paths, and dashboard title from the env vars
3. **Create folder** — finds the "Clients" parent folder in Grafana, then creates a subfolder named after the client (or skips if it already exists)
4. **Update dashboard** — copies `Client-Overview.json` from `templates/`, sets the title to `{ClientName}-Overview`, hides the subscription variable, and locks it to the client's subscription ID
5. **Send dashboard** — uploads the modified dashboard into the client's folder. If the dashboard already exists, it prompts you to overwrite (y/n)

## Deploying to a New Client

Just change two values in `env.base.ps1` and re-run:

```powershell
$env:Client_Name     = "NewClientName"
$env:Subscription    = "new-subscription-id-here"
```

Then:

```powershell
.\run.ps1
```

Everything else is derived automatically.

## What Changes in the Dashboard Copy

The scripts modify these fields in the copy (the original is never touched):

| Field | Value |
|---|---|
| `title` | `{ClientName}-Overview` |
| `description` | `{ClientName}-Overview` |
| `uid` | Removed (Grafana assigns one) |
| `id` | `null` |
| `version` | `0` |
| `sub` variable `current.text` | Client name |
| `sub` variable `current.value` | Subscription ID |
| `sub` variable `hide` | `2` (fully hidden) |
| `sub` variable `query.subscription` | Subscription ID |

## Grafana Folder Structure

After running for multiple clients, your Grafana looks like:

```
Clients/
  AcmeCorp/
    AcmeCorp-Overview
  AzBlue/
    AzBlue-Overview
  Contoso/
    Contoso-Overview
```

## Troubleshooting

**"Parent folder 'Clients' not found"** — Create a folder called "Clients" in Grafana manually before running.

**"bad request data" on upload** — Check the payload file in `dashboards/payloads/` to see what was sent. Common causes: leftover `uid` field, encoding issues, or wrong `folderUid`.

**"update_dashboard.py failed"** — Make sure `update_dashboard.py` is saved in the `tools/` folder and Python 3 is installed and on your PATH.

**Dashboard exists prompt** — If a dashboard with the same title already exists, the script asks you to overwrite. Enter `y` to replace it or `n` to skip.
