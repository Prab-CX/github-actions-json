# Send JSON Files to Coralogix via GitHub Actions

This repo demonstrates how to automatically send JSON files to Coralogix every time they are updated in your repository.

---

## How It Works

A GitHub Actions workflow runs on every push to `main` that modifies either JSON file (or can be triggered manually). It reads each file and POSTs it to the Coralogix log ingestion API, where it appears as a searchable log entry.

---

## Prerequisites

- A Coralogix account
- Your **Send-Your-Data API key** (found in Coralogix under **Account Management → API Keys → Send-Your-Data API key**)
- A GitHub repository

---

## Setup

### 1. Copy the workflow file

Add `.github/workflows/send-to-coralogix.yml` to your repository with the following content, adjusting `applicationName`, `subsystemName`, and the ingress domain to match your setup:

```yaml
name: Send JSON to Coralogix

on:
  push:
    branches: [main]
    paths:
      - "your-file.json"
  workflow_dispatch:

jobs:
  send-to-coralogix:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Send your-file.json
        run: |
          TEXT=$(jq -c '.' your-file.json)
          PAYLOAD=$(jq -n \
            --arg text "$TEXT" \
            '[{
              "applicationName": "github-actions",
              "subsystemName": "your-subsystem",
              "severity": 3,
              "text": $text
            }]')

          HTTP_STATUS=$(curl -s -o /tmp/cx_response.json -w "%{http_code}" \
            -X POST "https://ingress.<your-domain>/logs/v1/singles" \
            -H "Content-Type: application/json" \
            -H "Authorization: Bearer $CX_SEND_DATA_KEY" \
            -d "$PAYLOAD")

          echo "Response: $(cat /tmp/cx_response.json)"
          if [ "$HTTP_STATUS" != "200" ]; then
            echo "Failed with HTTP $HTTP_STATUS"
            exit 1
          fi
          echo "Sent successfully (HTTP $HTTP_STATUS)"
        env:
          CX_SEND_DATA_KEY: ${{ secrets.CX_SEND_DATA_KEY }}
```

### 2. Set the ingress domain

Replace `<your-domain>` in the endpoint URL with the one matching your Coralogix region:

| Region | Domain |
|--------|--------|
| EU1 | `coralogix.com` |
| EU2 | `eu2.coralogix.com` |
| US1 | `coralogix.us` |
| US2 | `cx498.coralogix.com` |
| AP1 | `coralogix.in` |
| AP2 | `coralogixsg.com` |

### 3. Add the secret to GitHub

In your repository go to **Settings → Secrets and variables → Actions → New repository secret** and add:

| Name | Value |
|------|-------|
| `CX_SEND_DATA_KEY` | Your Coralogix Send-Your-Data API key |

Your key can be found at **Account Management → API Keys → Send-Your-Data API key** in your Coralogix account. It begins with `cxtp_`.

### 4. Push to main

Once the workflow file and secret are in place, push any change to your JSON file on `main`. The workflow will trigger automatically. You can also run it on demand from the **Actions** tab using **Run workflow**.

---

## Verifying in Coralogix

After the workflow runs successfully, open **Explore → Logs** in your Coralogix account and filter by:

- `Application` = `github-actions`
- `Subsystem` = the subsystem name you set in the workflow

Your JSON file contents will appear as log entries with the full JSON payload visible in the log body.

---

## Severity Levels

The `severity` field in the payload maps to Coralogix severity levels:

| Value | Level |
|-------|-------|
| 1 | Debug |
| 2 | Verbose |
| 3 | Info |
| 4 | Warning |
| 5 | Error |
| 6 | Critical |

---

## Sending Multiple Files

Repeat the step block for each additional JSON file, changing the filename and `subsystemName` for each one. See the full example in this repo's [workflow file](.github/workflows/send-to-coralogix.yml).
