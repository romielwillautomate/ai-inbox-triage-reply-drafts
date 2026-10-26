# Setup: Gmail, Google Sheets, Claude, HubSpot, Slack and n8n

You need self-hosted n8n, the Google account of the shared inbox (Google Workspace or a personal Gmail account), a Google Sheet for the FAQ, an Anthropic API key, a HubSpot account and a Slack incoming webhook. Plan about 45 minutes, most of it in Google Cloud.

## 1. Google: Gmail API, Sheets API and a refresh token

The workflow talks to Gmail and Sheets with an OAuth access token. It gets a fresh one at the start of every run from a refresh token, so you only do this once.

### Turn on the APIs

1. Open [console.cloud.google.com](https://console.cloud.google.com), create a project (or pick one) for this workflow.
2. **APIs & Services → Library**: enable **Gmail API** and **Google Sheets API**.

### OAuth consent screen

1. **APIs & Services → OAuth consent screen** (called **Google Auth Platform** in newer consoles).
2. **User type**:
   - Google Workspace mailbox: choose **Internal**. Only people in your organization can use the app, and the refresh token does not expire after a week.
   - Personal Gmail account: choose **External** and add the inbox address as a **test user**. Google's rule: an app with an external user type in "Testing" status gets refresh tokens that **expire after 7 days**. You would have to repeat the refresh token step every week, or publish the app (Gmail scopes then need Google's verification). For a business inbox, use Workspace and Internal.
3. Add the scopes `https://www.googleapis.com/auth/gmail.modify` and `https://www.googleapis.com/auth/spreadsheets.readonly`.

About the Gmail scope: Google has no scope that allows creating drafts without also allowing sending. `gmail.modify` covers reading, labels and drafts, and it could send mail. This workflow never sends: no node calls `messages.send` or `drafts.send`, and the test checks that no request reaches them. Keep that in mind if you edit the workflow, and keep the client secret and refresh token as safe as a password.

### OAuth client

1. **APIs & Services → Credentials → Create credentials → OAuth client ID**.
2. Application type **Web application**. Under **Authorized redirect URIs** add `https://developers.google.com/oauthplayground`.
3. Copy the **Client ID** into `GOOGLE_CLIENT_ID` and the **Client secret** into `GOOGLE_CLIENT_SECRET`.

### Refresh token (OAuth 2.0 Playground)

1. Open [developers.google.com/oauthplayground](https://developers.google.com/oauthplayground).
2. Click the gear icon (top right), tick **Use your own OAuth credentials**, paste the client ID and secret.
3. In **Step 1**, type the two scopes into the box at the bottom, separated by a space:
   `https://www.googleapis.com/auth/gmail.modify https://www.googleapis.com/auth/spreadsheets.readonly`
4. Click **Authorize APIs** and sign in **as the shared inbox account** (for example hello@), not your own.
5. In **Step 2**, click **Exchange authorization code for tokens**. Copy the **Refresh token** into `GOOGLE_REFRESH_TOKEN`.
6. Set `GMAIL_ADDRESS` to that inbox address.

A refresh token also stops working when the account's password changes, when it is unused for six months, or when you revoke access in the Google account's security settings. The run then fails at "Stop: Google Auth Failed" with `invalid_grant`, and Slack gets the alert: repeat the Playground steps.

## 2. The FAQ sheet

1. Create a Google Sheet in the inbox account's Drive (or share it with that account as Viewer).
2. Name a tab **FAQ**. Row 1: `Question` and `Answer`. From row 2, one entry per row:

   | Question | Answer |
   |---|---|
   | What are your hours? | Monday to Saturday, 8 AM to 6 PM. We are closed on Sundays. |
   | How do I book or change a visit? | Book online at your-business.example.com/book, or reply with two dates and times that suit you. |

3. Copy the spreadsheet ID from its URL (`https://docs.google.com/spreadsheets/d/THIS_PART/edit`) into `FAQ_SHEET_ID`. Leave `FAQ_RANGE=FAQ!A2:B` unless your tab or columns differ (for a tab name with spaces use `'Inbox FAQ'!A2:B`).

The answers are the only facts Claude may use in a draft. A row without an answer is left out. Every run reads the sheet, so edits apply on the next run. Write answers the way you would want them quoted to a customer.

## 3. Claude

Create a key at **console.anthropic.com → API keys** and put it in `ANTHROPIC_API_KEY`. The default model is `claude-haiku-4-5`, the cheapest current model, which fits a step that runs for every email. `claude-sonnet-5` and `claude-opus-5` also work (`ANTHROPIC_MODEL`). The request forces one tool call with thinking turned off, so `claude-opus-5-5` and the Claude Fable and Mythos models do not work: they refuse forced tool calls or thinking turned off. The workflow refuses to start with them and says so.

## 4. HubSpot

1. In HubSpot, create a private app: **Development → Legacy apps → Create legacy app → Private** (older accounts: **Settings → Integrations → Private Apps → Create a private app**).
2. On the **Scopes** tab, add `crm.objects.contacts.read` and `crm.objects.contacts.write`. Notes attached to contacts use the contact scopes.
3. Create the app and copy the access token into `HUBSPOT_ACCESS_TOKEN`.
4. Optional: to have the latest email category on the contact record (for lists and filters), create a contact property: **Settings → Properties → Contact properties → Create property**, label "Last inbox category", field type **Single-line text**. Put its internal name (for example `last_inbox_category`) into `HUBSPOT_CATEGORY_PROPERTY`. Without it, the category is recorded in the note only.

What the workflow writes: it searches the contact by email. A new sender is created with email and, if the From header has one, first and last name. An existing contact keeps its data; only blank first or last names are filled, plus the category property when set. Every triaged email adds one note to the contact with the category, sentiment, urgency, subject, summary and whether a draft was saved. Spam and emails Claude could not triage create nothing in HubSpot.

## 5. Slack

1. Go to [api.slack.com/apps](https://api.slack.com/apps) → **Create New App → From scratch**, pick your workspace.
2. **Incoming Webhooks** → turn it on → **Add New Webhook to Workspace** → pick the channel your team watches (for example #inbox).
3. Copy the URL into `SLACK_WEBHOOK_URL`.

The same URL is used for escalations, the daily digest and the error alerts.

## 6. n8n

1. Add the variables from `.env.example` to your n8n environment and restart n8n. The nodes read them with `$env`, so self-hosted n8n needs `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`. (n8n Cloud does not allow `$env`. There, replace `$env.` with `$vars.` and create the same names under Variables.)
2. Set the time zone the digest should follow: the `GENERIC_TIMEZONE` environment variable, or **Workflow settings → Timezone** after the import. The digest runs at 5 PM in that zone.
3. Import `workflow/error-alert-slack.json` and **activate it**. n8n 2.x only runs an error workflow that is active; an inactive one is skipped without any alert.
4. Import `workflow/ai-inbox-triage-reply-drafts.json`. In its settings, choose the error alert workflow as its **Error Workflow**.
5. Activate the workflow. The inbox check runs every 5 minutes; change it in the **Schedule: Check Inbox** node. If you do, keep `RUN_TIME_LIMIT_MINUTES` (default 3) at least 2 minutes below the new interval: n8n starts a scheduled run even while the last one is still working, and two runs at once could draft the same email twice. The digest time is in **Schedule: Daily Digest**.

## 7. First run

1. For the first day, set `LOOKBACK_DAYS=1` and `MAX_PER_RUN=5`, so the first runs only touch a few recent emails.
2. From a personal address, send the inbox a question your FAQ answers, and one it does not.
3. Within 5 minutes:
   - Gmail: each email has the labels **AI Triage/Question** and **AI Triage/Draft ready**, and a draft reply sits in its thread. The second one also has **AI Triage/Needs person**.
   - Slack: an alert for the second email, with a link to the thread.
   - HubSpot: your address is a contact with two notes.
4. Read the drafts. If the tone or facts are off, improve the FAQ answers first.
5. Then raise `MAX_PER_RUN` back to 20 and `LOOKBACK_DAYS` to 2.

Test with the active schedule rather than the editor's **Execute workflow** button: n8n keeps workflow static data (the ledger of handled emails) only for production runs. A manual run does not draft twice either, because emails that already carry one of the workflow's labels are skipped, but it does not remember skipped or failed emails.

## 8. Gmail labels the workflow adds

With the default `LABEL_PREFIX` they appear under **AI Triage**. The workflow creates them on its first run.

| Label | Meaning |
|---|---|
| `AI Triage/Question`, `/Booking`, `/Billing`, `/Complaint`, `/Other` | Claude's category |
| `AI Triage/Draft ready` | A reply draft is in the thread. Review, edit and send it yourself |
| `AI Triage/Needs person` | A person should handle it: the Slack alert says why |
| `AI Triage/Spam` | Claude is sure it is spam. Nothing else was done, the email stays where it was |
| `AI Triage/Skipped` | No-reply sender, mailing list, automatic reply or the business's own mail |

The labels are also the workflow's backup memory: an email with one of them is never triaged again, even if the workflow is re-imported and its ledger is empty.

## Troubleshooting

Open the workflow's **Executions** list in n8n:

| You see | Likely cause |
|---|---|
| Run stopped at "Stop: Bad Settings" | A setting is missing or invalid. The message (and the Slack alert) lists every problem |
| "Stop: Google Auth Failed" with `invalid_grant` | The refresh token expired or was revoked (7 days for an External app in Testing status, a password change, or revoked access). Create a new one with the Playground |
| "Stop: Google Auth Failed" with `invalid_client` | `GOOGLE_CLIENT_ID` or `GOOGLE_CLIENT_SECRET` is wrong |
| "Stop: Gmail List Failed" with 403 | The refresh token was made without the `gmail.modify` scope, or for another account |
| "Stop: FAQ Failed" | `FAQ_SHEET_ID` or `FAQ_RANGE` is wrong, the sheet is not shared with the inbox account, or no row has both a question and an answer. Nothing is drafted until it works |
| A run failed but Slack stayed quiet | The error alert workflow is not active, is not chosen as the Error Workflow, or `SLACK_WEBHOOK_URL` is empty |
| "Stop: Run Needs Attention" | Read the message: it names each email and what is left to do. "will be tried again after" means the workflow retries by itself; "by hand" means a step did not happen |
| No drafts at all, runs end at "Any New Emails?" | Nothing new matched `GMAIL_QUERY` within `LOOKBACK_DAYS`, or every email was handled already |
| Some emails are only drafted on the next run ("reached RUN_TIME_LIMIT_MINUTES" in Record Failure) | The run used its time limit, so the rest of a backlog waits for the next run. This is normal after a quiet night or an outage |
| An email has no labels and no draft | It may be skipped on purpose (a newer message from the same sender in the thread, an existing draft, or a reply from the team). The run's "Record Skip" output says why |
| HubSpot errors about a property | `HUBSPOT_CATEGORY_PROPERTY` names a property that does not exist. Create it or leave the setting empty |
| The digest says the drafts count is unknown | Gmail did not return the drafts list during the digest run. The next digest tries again |
