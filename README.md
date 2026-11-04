# Lead Capture, Qualification & Follow-up System

*An n8n automation by Khubaib · Built with n8n, Google Sheets and Gmail*

## The problem

Small agencies and service businesses lose leads for simple reasons: replies are slow, every lead is treated the same, follow-ups get forgotten, and lead data is scattered across inboxes and phones. A lead that waits hours for a reply usually goes to a competitor.

This system receives each lead instantly, checks and scores it, replies automatically, alerts the owner about hot leads, and keeps following up until the lead replies or the sequence ends.

**Target use case:** digital agencies (web development, e-commerce, marketing, SEO). Scoring rules and message templates are kept separate from the core logic, so the same system can be adapted to other niches such as real estate.

## What it does

1. Captures leads from a web form (an n8n Form Trigger; a Webhook can replace it without changing the rest)
2. Cleans and validates the data, and ignores duplicate leads
3. Scores each lead as Hot, Warm or Not Fit using transparent rules
4. Saves every lead to a Google Sheet that works as a mini CRM
5. Sends the lead a confirmation email that matches their category
6. Emails the owner immediately when a Hot lead arrives
7. Sends up to 3 follow-up emails on a schedule and stops as soon as the lead replies
8. Alerts the owner when any workflow fails, so no lead is lost silently

## Architecture

The project is made of three workflows. Their exported JSON files are in this repository.

| Workflow | File | Purpose | Trigger |
|---|---|---|---|
| Lead Capture and Qualification | `01-lead-capture-and-qualification.json` | Receive, validate, deduplicate, score, save, confirm, alert | Form submission |
| Lead Follow-up Engine | `02-lead-followup-engine.json` | Detect replies, send scheduled follow-ups, close finished leads | Every hour |
| Global Error Handler | `03-global-error-handler.json` | Email an alert when any workflow fails | Error Trigger |

## Workflow 1: Lead Capture and Qualification

![Lead Capture and Qualification workflow](screenshots/01-lead-capture-workflow.png)

**Flow:** Lead Form Trigger → Standardize Lead Fields → Validate Lead Data → Finding Existing Lead → Is Duplicate? → Score Lead → Save new lead → Route by Temperature → confirmation email → Mark Contacted

| Node | What it does |
|---|---|
| Lead Form Trigger | Form with dropdowns for service, budget and timeline, so scoring always receives clean values |
| Standardize Lead Fields | Maps form labels to standard field names, trims and lowercases the email, keeps the phone number as text |
| Validate Lead Data | Lets through only leads with name, email and phone. Incomplete leads are saved with status Invalid and a reason in the error_message column |
| Finding Existing Lead | Looks up the email in the sheet. Always Output Data is on, so a new lead still continues down the flow |
| Is Duplicate? | Existing email: logged in the Logs tab as duplicate_ignored, nothing else happens. New email: continues |
| Score Lead | Rule-based scoring (see the table below) |
| Save new lead | Appends the lead with score, temperature and status New |
| Route by Temperature | Switch node with three branches: Hot, Warm, Not Fit |
| Email Hot / Warm / Not Fit Lead | Category-specific confirmation email. Retries up to 3 times. If it still fails, the error output marks the lead as Email Failed with the reason |
| Alert Owner Hot Lead | Hot leads only: emails the owner the name, service, budget, timeline, score, phone and email |
| Mark Contacted / Mark Owner Notified | Update the sheet so contact and alert status are always recorded |

![Hot lead alert email](screenshots/05-alert-email.png)

### Scoring rules

Rules are used instead of AI because they are predictable, free, easy to debug and easy to explain to a client. Budget is read from the numbers inside the option text, so small wording or formatting changes in the form do not break scoring. If any value cannot be matched, a scoring_warning is written instead of silently scoring zero.

| Field | Option | Points |
|---|---|---|
| Budget | Under $500 | 0 |
| | $500-$999 | 1 |
| | $1000-$2499 | 3 |
| | $2500+ | 5 |
| Timeline | Just Exploring | 0 |
| | 1-3 Months | 1 |
| | Within 1 Month | 2 |
| | Within 2 Weeks | 3 |
| | ASAP | 4 |
| Service | Other | 0 |
| | SEO, Digital Marketing | 1 |
| | Website Development, E-commerce Website | 2 |
| Project details | More than 20 characters | 1 |

**Maximum score: 12.** Score of 8 or more is Hot, 4 to 7 is Warm, below 4 is Not Fit.

## Workflow 2: Lead Follow-up Engine

![Lead Follow-up Engine workflow](screenshots/02-followup-engine-workflow.png)

**Flow:** Schedule Trigger → Get Lead → Find Due Leads → Get Recent Inbox → Check Replies → Is replied? → (true) Mark Replied, or (false) Get Templates → Build Follow-ups → Send Follow-up → Mark Followed Up

| Node | What it does |
|---|---|
| Schedule Trigger | Runs every hour |
| Get Lead | Reads all rows from the lead tab |
| Find Due Leads | Keeps only leads whose status is Contacted, who have not replied, whose follow-up time has passed and who have received fewer than 3 follow-ups. Text is trimmed and lowercased, so Yes, yes and extra spaces do not matter |
| Get Recent Inbox | Reads recent inbox messages once per run |
| Check Replies | Marks each due lead as replied or not by looking for their email among the senders |
| Is replied? | Splits leads into replied and not replied |
| Mark Replied | Sets reply status to Yes and lead status to Replied, so follow-ups stop |
| Get Templates | Reads the follow-up messages from the Templates tab |
| Build Follow-ups | Picks the right template for each lead, fills in the name and service, and calculates the next follow-up time. On the last stage the lead is set to Closed. Invalid template data raises an error instead of failing silently |
| Send Follow-up | Sends the email, with retries |
| Mark Followed Up | Updates the follow-up count, last contact time, next follow-up time and status. This prevents the same lead from being emailed again in the next run |

### Follow-up sequence

| Stage | When | After it |
|---|---|---|
| First follow-up | Day 1 after the confirmation email | Next in 2 days |
| Second follow-up | Day 3 | Next in 4 days |
| Third (last) follow-up | Day 7 | Lead status becomes Closed |

If the lead replies at any point, the sequence stops and the lead is marked Replied. Not Fit leads are never followed up.

Messages are written in the Templates tab of the sheet with {{name}} and {{service}} placeholders. The business can edit them without touching the workflow.

## Workflow 3: Global Error Handler

![Global Error Handler workflow](screenshots/03-error-handler-workflow.png)

An Error Trigger workflow connected to the other workflows through their Error Workflow setting. When any node fails after its retries, it emails the owner the workflow name, the failed node, the error message, the execution ID and a link to the execution.

## Google Sheet structure

![Lead tab of the Google Sheet](screenshots/04-sheet-lead-tab.png)

| Tab | Purpose |
|---|---|
| lead | One row per lead: contact details, score, temperature, status, follow-up count and times, owner alert time, email status, error message |
| Templates | Follow-up messages: stage, subject, body, days_until_next |
| Logs | Events such as ignored duplicates: timestamp, email, event, details |

**Lead status lifecycle:** New → Contacted → Replied or Closed. Other statuses: Invalid (incomplete data) and Email Failed (confirmation email could not be sent).

## Reliability and error handling

| Risk | How it is handled |
|---|---|
| Duplicate lead | Email lookup before saving; ignored and logged |
| Incomplete lead | Saved as Invalid with a reason, never lost |
| Email API fails | Retry 3 times, then lead marked Email Failed with the error text |
| Any other node fails | Global Error Handler emails the owner |
| Same follow-up sent twice | Sheet is updated right after each send, and due leads are re-checked on every run |
| Lead replied | Detected before sending, follow-ups stop |
| Form option text changes | Budget scored from numbers; unmatched values produce a warning |
| Messy sheet values | Status and reply values are trimmed and compared case-insensitively |

## Test results

| Test | Expected | Result |
|---|---|---|
| New valid lead | Row saved with score, temperature and status; confirmation email sent | Pass |
| Same email submitted again | No new row, entry in Logs, no email | Pass |
| Leads of different quality | Correct Hot, Warm and Not Fit classification | Pass |
| Hot lead | Owner alert email and alert time saved | Pass |
| Deliberate failure in a node | Error alert email with workflow and node name | Pass |
| Due lead in the follow-up engine | Stage 1 email sent, count and next date updated | Pass |
| Engine run again immediately | No lead selected, no duplicate email | Pass |
| Lead who replied | No follow-up, status Replied | Pass |
| Third follow-up | Last template sent, status Closed, no next date | Pass |
| Reply status written as yes instead of Yes | Lead still excluded from follow-ups | Pass |

## Setup

1. Create a Google Sheet with three tabs named lead, Templates and Logs, and add the column headers described above.
2. Import the three workflow JSON files from this repository into n8n.
3. Connect Google Sheets and Gmail credentials in n8n using OAuth.
4. Select your own sheet in every Google Sheets node and set your owner email in the alert nodes.
5. In each workflow's settings, set your timezone and set the Error Workflow to Global Error Handler.
6. Publish the workflows and share the form's production URL.

## Limitations

- Reply detection checks whether the lead's email address sent any message in the recent inbox. It does not check that the message answers the follow-up, and it only looks at a limited number of recent messages.
- Gmail has daily sending limits. A business email service is better for large volumes.
- Failure alerts are sent through Gmail, so an expired Gmail connection would also stop alerts. Keep the Google OAuth app in production and check the Executions tab regularly.
- The demo runs on a local n8n instance. For 24/7 operation, deploy n8n on a VPS or n8n Cloud.

## Problems solved while building

- **New leads stopped at the duplicate check:** a lookup with no match returns nothing, which ends the flow. Fixed with Always Output Data and by reading lead data from the earlier node.
- **Budget scored as zero:** dropdown text did not match the scoring text exactly. Fixed by reading the numbers inside the option and adding a warning for unmatched values.
- **Replied leads still followed up:** a value written as yes did not match Yes. Fixed with trimmed, lowercase comparison.
- **Lead silently skipped:** a stray character in a date made the row unreadable. Dates are now read defensively.
- **Google authorization expired:** caused by an OAuth app in testing mode. Fixed by moving it to production.

## Future improvements

- WhatsApp or Telegram alerts for the owner
- Send follow-ups only during working hours
- Stop the sequence when a lead asks to unsubscribe
- AI reading of the free-text project description, with rules as a fallback
- Daily summary report of new, replied and closed leads

## Demo

Demo video: coming soon.
