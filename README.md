# Visa Automation (n8n)

End-to-end WhatsApp visa application automation built on n8n. Users submit visa requests and documents via WhatsApp (Twilio), the workflow manages their state in Google Sheets, parses uploaded documents with LlamaIndex, analyses them with an LLM (Groq), and sends status updates back over WhatsApp.

## Main Workflow

**`workflows/final-visa-automation.json`** — the complete, production-ready workflow.

### How it works

1. **Inbound WhatsApp message** — A Twilio webhook receives the user's message and normalises the payload (phone number, name, text, media URL).
2. **User lookup** — Checks Google Sheets for an existing record matching the phone number.
   - New user → added to the sheet and sent a welcome/onboarding message.
   - Returning user → routed by their current application state (Switch node).
3. **Document submission** — When the user sends a document (image/PDF):
   - The file is downloaded via the media URL.
   - Uploaded to **LlamaIndex Cloud** for parsing.
   - The workflow polls until parsing status = `SUCCESS`.
   - The extracted markdown text is passed to a **Groq LLM** for analysis/classification.
   - Results are appended to Google Sheets.
4. **State transitions** — Google Sheets rows are updated as the application moves through states (e.g. `→ payment_pending`, `→ visa_available`).
5. **Status updates trigger** — A Google Sheets row-update trigger fires outbound WhatsApp messages to the applicant whenever their status changes.

### Nodes overview

| Node | Purpose |
|------|---------|
| Webhook | Receives inbound Twilio WhatsApp messages |
| Code in JavaScript | Normalises Twilio payload |
| Google Sheets (get) | Looks up existing applicant by phone |
| If / Switch | Routes by user existence and application state |
| Google Sheets (add/append) | Creates new users, logs document results, updates state |
| Twilio (×7) | Sends WhatsApp replies at each stage |
| HTTP Request (×4) | Downloads media, uploads to LlamaIndex, polls parse job, fetches markdown |
| Wait | Polling delay for LlamaIndex job completion |
| Basic LLM Chain + Groq | Analyses parsed document content |
| Google Sheets Trigger | Fires on row update to send status-change notifications |

## Other files
- `workflows/01_inbound_router.json` – standalone inbound router (placeholder credentials)
- `workflows/02_state_manager.json` – state management skeleton
- `workflows/03_document_handler.json` – document handling skeleton
- `workflows/04_status_updates.json` – outbound status updates skeleton
- `prompts/n8n-workflow-generator.md` – prompt guidance used to generate these workflows

## Run n8n locally
From this folder:

- `docker compose up -d`
- Open `http://localhost:5678`

## Import workflows
In n8n UI:
- Workflows → Import from File
- Select a JSON from `workflows/`

## Credentials
This repo uses placeholders. Create credentials in n8n for:
- Google Sheets OAuth2
- Twilio HTTP Basic Auth (Account SID as username, Auth Token as password)

## Google Sheets OAuth (n8n + ngrok)
If you see `Error 400: redirect_uri_mismatch`, your Google OAuth Client needs the exact n8n callback URL whitelisted.

### 1) Set your public base URL in n8n
If you access n8n via ngrok, n8n must know its public URL.

Set these environment variables for the n8n container (examples):
- `N8N_EDITOR_BASE_URL=https://<your-ngrok-domain>`
- `WEBHOOK_URL=https://<your-ngrok-domain>/`

### 2) Add the Authorized redirect URI in Google Cloud
In Google Cloud Console → APIs & Services → Credentials → your OAuth Client:
Add this to **Authorized redirect URIs** (must match exactly):

`https://<your-ngrok-domain>/rest/oauth2-credential/callback`

Notes:
- The scheme must match (`https` for ngrok).
- If your ngrok URL changes, you must update Google Cloud or use a static ngrok domain.

### 3) Recreate or reconnect the credential in n8n
After updating the redirect URI, retry the OAuth login in the n8n credential.



## LIMITS
- Google Sheets API costs: https://developers.google.com/workspace/sheets/api/limits

## Security
- Do not store OAuth client secrets in this repo.
- Rotate the client secret in Google Cloud Console if it was pasted/shared.



