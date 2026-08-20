# WhatsappWebhook — local testing and ngrok guide

This repository contains a minimal Express webhook for WhatsApp (app.js). Below are step-by-step instructions to run it locally and expose it publicly for WhatsApp webhook testing using ngrok, plus a GitHub Actions workflow you can trigger to produce a temporary public URL from GitHub Actions logs.

## Quick summary

- Locally: run `node app.js` with `VERIFY_TOKEN` set, then run `ngrok http 3000` to get a public HTTPS URL you can paste into the WhatsApp/Facebook webhook configuration.
- GitHub Actions: trigger the `Expose webhook with ngrok` workflow and read the workflow logs to get the temporary public URL.

## Local setup (recommended)

1. Clone the repo and install dependencies:

   ```bash
   git clone https://github.com/G21K/WhatsappWebhook.git
   cd WhatsappWebhook
   npm install
   ```

2. Set the verification token and start the app (example uses a token `my_secret_token`):

   ```bash
   export VERIFY_TOKEN=my_secret_token
   node app.js
   ```

   The app listens on port 3000 by default. You can set `PORT` to change it.

3. Install and run ngrok to expose port 3000 (https is required by WhatsApp):

   - If you don't have ngrok: https://ngrok.com/download
   - Run:

   ```bash
   ngrok http 3000
   ```

   Copy the HTTPS URL (example: `https://abcd1234.ngrok.io`).

4. Configure WhatsApp / Facebook webhook

   In the Facebook Developer dashboard for your app or WhatsApp business setup, for the webhook URL use:

   `https://<your-ngrok-host>/`  (include the trailing slash)

   For the Verify Token enter the same value as the `VERIFY_TOKEN` environment variable (e.g. `my_secret_token`). Facebook will perform a GET verification call to your webhook endpoint with query parameters `hub.mode`, `hub.challenge`, and `hub.verify_token`.

5. Manual verify test with curl

   - Simulate the verification GET that Facebook performs:

     ```bash
     curl "https://<your-ngrok-host>/?hub.mode=subscribe&hub.verify_token=my_secret_token&hub.challenge=12345"
     ```

     If the token matches `VERIFY_TOKEN`, your server should respond with `12345`.

   - Simulate sending a webhook POST event:

     ```bash
     curl -X POST -H "Content-Type: application/json" -d '{"object":"whatsapp_business_account","entry":[{"id":"WHATSAPP_BUSINESS_ID","changes":[{"value":{"messages":[{"from":"123456789","text":{"body":"hello"}}]}}]}]}' https://<your-ngrok-host>/
     ```

     The server prints the JSON to logs and returns 200 OK.

## GitHub Actions workflow (manual trigger)

I added a workflow `.github/workflows/expose-ngrok.yml` that you can trigger manually (workflow_dispatch). It will:

- Checkout the repo
- Install Node and dependencies
- Start `node app.js` in the background
- Install `ngrok` and start a tunnel
- Query the local ngrok API (http://127.0.0.1:4040/api/tunnels) to find the HTTPS public URL and print it to the job logs

Notes:
- The workflow uses a `verify_token` input (you should set the same value you'll use in your WhatsApp app config).
- The runner and the tunnel exist only while the workflow job runs; the URL is temporary and valid only while the workflow is running.

To run it:

1. In GitHub go to Actions -> Expose webhook with ngrok -> Run workflow
2. Enter `verify_token` input (e.g. `my_secret_token`) and run
3. Open the job logs and look for the line starting with `Public ngrok URL:` — copy that URL and paste it into your Facebook/WhatsApp webhook configuration.

## Security/warnings

- The ngrok URL exposes your local server to the internet. Only enable it when testing and do not expose production credentials.
- For production, deploy the webhook to a proper hosting provider with HTTPS and stable URL.

