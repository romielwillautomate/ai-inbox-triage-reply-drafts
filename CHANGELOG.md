# Changelog

## v1 (2026-10-26)

First build: an n8n workflow that triages a shared Gmail inbox with Claude, saves FAQ grounded reply drafts in each thread, logs senders in HubSpot, alerts a person in Slack and posts a daily digest. It comes with an error alert workflow and an end to end mock test (see [docs/test-report.md](docs/test-report.md) for the scenarios and checks).

### Inbox check (every 5 minutes)

- Settings are checked before any API call; a problem stops the run with the full list, including a Claude model that refuses the forced tool call.
- A fresh Google access token comes from the refresh token on every run (`POST oauth2.googleapis.com/token`, `grant_type=refresh_token`), so no n8n Google credential is needed.
- `messages.list` with `GMAIL_QUERY newer_than:<LOOKBACK_DAYS>d`. Message ids already in the ledger (workflow static data, kept for `LEDGER_DAYS`) and emails waiting for a retry are dropped; at most `MAX_PER_RUN` emails per run, oldest first.
- The Gmail labels (`AI Triage/...`) are looked up once per run and created on the first run, one at a time, parent first.
- The FAQ comes from Google Sheets (`values.get`); rows without an answer are left out. Without a FAQ nothing is drafted.
- Each email is read with `messages.get`. Mail from the business (`GMAIL_ADDRESS`, `OWN_ADDRESSES`), no-reply senders, mailing lists (`List-Id`, `List-Unsubscribe`, `Precedence`), automatic replies (`Auto-Submitted` and others) and emails that already carry a workflow label are skipped without an AI call.
- `threads.get` gives Claude the earlier messages and stops a second draft: no draft when the thread already has a newer draft or a reply from the team, and only the newest message gets one when a customer writes twice.
- Claude (`ANTHROPIC_MODEL`, default `claude-haiku-4-5`) gets the email inside `<email>` tags in the user turn and answers through one forced `record_triage` tool call with a strict schema: category, sentiment, urgency, needs a person, confidence, whether the FAQ answers it, FAQ entries used, summary and the draft.
- Code checks every field and decides. Spam that Claude is sure about is only labelled. Other emails get a draft (`drafts.create` with the thread id, `In-Reply-To`, `References` and the same subject, as Gmail's threading rules require), labels, a HubSpot contact and note, and a Slack alert when a person is needed: complaints, angry or urgent emails, low confidence, questions the FAQ does not answer, anything Claude flags or a cited FAQ entry that does not exist.
- A code check for phrases aimed at an AI system (prompt injection) sends the email to a person with no draft, whatever Claude answers.
- HubSpot: contact search by email, then create (email and names from the From header) or update (blank names and the optional `HUBSPOT_CATEGORY_PROPERTY` only). A 409 "already exists" on create uses the existing id. One note per triaged email, associated with type 202.
- Failures before the draft (Gmail read or thread, Claude, the draft itself) leave the email for a later run with backoff: `RETRY_BASE_MINUTES`, then twice as long each time, up to `MAX_TRIAGE_TRIES`; after that a person gets it. One Claude failure makes the rest of the run's emails wait for the next run without using a try. Failures after the draft are listed for a person. Any problem ends the run as failed, so the error workflow posts the whole list.
- The workflow never sends email: the only Gmail write calls are `drafts.create` and `messages.modify`, and the recipient comes from the email's headers in code, with line breaks stripped.

### Daily digest (5 PM)

- Reads the ledger and `drafts.list`, and posts to Slack: emails by category, spam, drafts saved and still waiting to be sent, escalations with links, skips with reasons, retries. If Gmail does not return the drafts list, the digest is still posted with the count marked unknown, and the run is flagged.

### Decisions

- **HubSpot: search, then create or update, instead of batch upsert by email.** HubSpot's contacts guide says partial upserts are not supported when email is the `idProperty`, and an upsert would overwrite names a person entered. Search plus create keeps existing data, and the documented search index delay is covered by reading the existing id from the 409 that HubSpot returns for a duplicate email.
- **Retries in two layers.** n8n retries each call inside the run with a fixed wait. Real backoff (growing waits) happens across runs, for the steps that are safe to repeat because nothing has been written yet.
- **`claude-haiku-4-5` by default.** The call runs for every email and only needs a classification and a short draft. Sonnet 5 and Opus 5 work too; the token counter accepted the same request on all three.

### Found and fixed while testing and reviewing

- **The label list ran once per new email.** n8n runs an HTTP Request node once for each incoming item, so a run with 26 new emails fired 26 parallel `labels.list` calls, and only 16 reached the mock. The failed ones' error items would have stopped the run at "Stop: Labels Failed" after the loop had already drafted everything (the schedule test caught it). "Gmail: List Labels" now runs once per run.
- **Labels were created in parallel**, so a child such as `AI Triage/Billing` could be created before its parent. They are now created one at a time, parent first, and the test checks the order.
- **An untested failure path.** The first test covered Gmail failing to save a draft but not Gmail failing to return an email. A scenario was added (run 4), and it also showed that an email waiting behind a Claude outage waits again when the outage outlasts the next run, without using one of its tries.
- **Digest error text.** When the paginated drafts list failed, n8n's error had no `message`, and the digest showed a raw JSON blob. It now shows the status code and Gmail's error.
- **Sender name.** Without `BUSINESS_NAME`, drafts would have come from "our team" and been signed that way. The From line is now the address alone and the signature "Kind regards".

### Fixed in a second review (October 26, 2026)

An adversarial review read every node, compared the mocks with the vendor docs again and added a test for each problem it found. Run against the first version of the workflow, the extended suite passed 63 of 78 checks; every failure traces back to one of the problems below.

- **Overlapping runs could draft the same email twice.** n8n starts a scheduled run even while the previous one is still working, and a run with `MAX_PER_RUN` near 100 (or a slow API) can take longer than 5 minutes. The second run would read the same unlabelled emails, skip the ones the first run had already labelled, catch up with it and then draft, note in HubSpot and alert on each email a second time; whichever run ended last would also overwrite the other's ledger. The README claimed the labels and the thread check prevented this; they do not, because both runs check before either has saved anything. (Found by reading the code; the test does not start two runs at once.) New setting `RUN_TIME_LIMIT_MINUTES` (default 3): after it, a run starts no more Claude calls and the rest wait for the next run without using a try (at least one email per run is always triaged). A test copy with a limit under a second checks it. The remaining edge (one email whose calls hang until their timeouts) is listed under limitations.
- **A From header with a list inside the angle brackets became two recipients.** `"Lena Ruiz" <lena.ruiz@example.com,list@other.example.net>` was read as one address, so the draft's `To` named both, against the README's promise that no text in an email can add a recipient. The address is now always one plain address (the first). The mock also missed this: it only checked the last address in `To`; it now refuses a `To` with more than one address.
- **A teammate's reply from their own address did not count as a team reply.** Only messages sent from the shared inbox did, so a customer email already answered by a colleague from an `OWN_ADDRESSES` address still got a draft. Now any later message from the inbox or from `OWN_ADDRESSES` counts, and such messages are marked "Us" in the history Claude reads.
- **The prompt injection check only read the subject and body.** A harmless follow-up in a thread whose earlier message tried an injection went to Claude with that text in the history and got a normal draft. The check now reads everything of the customer's that Claude sees (sender name, attachment names, earlier customer messages), and the fake closing tag is also neutralised when written with spaces (`< /email>`).
- **Draft headers broke RFC limits on long input.** A non ASCII subject became one encoded word of any length (156 characters for the test subject; RFC 2047 allows 75), and `References` was one line that passes RFC 5322's 998 character limit in a thread of about 13 messages. Subjects and names are now split into encoded words of at most 75 characters on folded lines, and `References` puts each message id on its own folded line. The mock now refuses longer header lines and encoded words.
- **The inbox list was 100 ids, and the README said a burst would be picked up later.** It would not: handled emails stay in the inbox and keep filling the newest 100, so older unhandled ones were never listed again. The list now asks for 500 (the API maximum) and the limitation is described as it is.
- **Docs.** The test report now says that the one recipient and no Cc/Bcc rules are the workflow's own rules checked by the mock, not Gmail's. The README lists the retry of `drafts.create` (a save that times out after Gmail stored it can leave two drafts) next to the same note for HubSpot.
- **Reviewed and kept:** n8n's in-run retries use a fixed wait (at most 5 seconds is all n8n offers), with growing waits only across runs for the steps that are safe to repeat. Both the README and the decisions above say so, and with the sequential loop no API's documented rate limit is close.
