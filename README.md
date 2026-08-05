# HR Hiring Pipeline — Animaster Studio

An n8n automation that runs an entire hiring pipeline from a Google Form submission to job offer — logging applicants to Google Sheets, notifying HR on Telegram, and letting HR score, schedule, and decide on candidates using simple Telegram text commands, with emails sent automatically at every stage.

## How it works

### 1. New application intake
1. A candidate submits a Google Form; the response lands in a connected Google Sheet.
2. **HR Hiring Pipeline - Trigger** fires on the new row.
3. **Edit Fields** builds a unique `applicant_id` (`APP-` + digits from the submission timestamp) and normalizes the form fields (name, email, phone, position, experience, portfolio, etc.).
4. **Check Email Duplicate** / **Check Duplicate Email** looks for an existing applicant with the same email.
   - **If duplicate:** HR gets a Telegram duplicate warning and the entry is skipped.
   - **If new:** the applicant is saved to the Google Sheet (**Save New Applicant**), sent a welcome email (**Email - Welcome**), and HR is notified on Telegram with the applicant's details and their `Applicant ID` (**Telegram - HR Notification**).

### 2. HR actions via Telegram
HR manages the entire pipeline by replying to the bot with one of three commands. A **Switch** node routes each message to the right flow, and an **Invalid Command** reply is sent (with format examples) if the message doesn't match.

| Command | Format | Example |
|---|---|---|
| Score a portfolio | `SCORE <applicant_id> <0-100>` | `SCORE APP-040720261439 85` |
| Schedule an interview | `INTERVIEW <applicant_id> <date> <time>` | `INTERVIEW APP-040720261439 2026-07-15 10:00AM` |
| Give final decision | `RESULT <applicant_id> <selected/rejected> [joining_date]` | `RESULT APP-040720261439 selected 2026-07-20` |

**Scoring (`SCORE`)**
- Parsed by **Parse SCORE Command**, then branched by **Score 70 or Above?**
- **≥ 70:** applicant status updated to *Shortlisted*, shortlisted email sent, HR notified on Telegram.
- **< 70:** applicant status updated to *Score Rejected*, rejection email sent, HR notified on Telegram.

**Interview scheduling (`INTERVIEW`)**
- Parsed by **Parse INTERVIEW Command**.
- Applicant status updated to *Interview Scheduled*, an interview invite email is sent with the date/time, and HR is notified on Telegram.

**Final decision (`RESULT`)**
- Parsed by **Parse RESULT Command**, then branched by **Decision Selected?**
- **Selected:** status updated to *Selected*, a job offer email is sent (with joining date), HR notified on Telegram.
- **Rejected:** status updated to *Interview Rejected*, a rejection email is sent, HR notified on Telegram.

Every branch looks up the applicant's current record in the sheet first (`Get Applicant - ...` nodes) before updating it (`Update Status - ...` nodes), so the Google Sheet is always the source of truth for applicant status.

## Workflow diagram

```
[Google Sheets Trigger] → Edit Fields → Check Email Duplicate → Is Duplicate?
      ├─ Yes → Telegram: Duplicate Warning
      └─ No  → Save New Applicant → Email: Welcome → Telegram: HR Notification

[Telegram Trigger] → Switch
      ├─ SCORE     → Parse SCORE Command     → Score 70 or Above?
      │                                            ├─ Yes → Get/Update: Shortlisted   → Email: Shortlisted        → Telegram: Shortlisted
      │                                            └─ No  → Get/Update: Score Rejected → Email: Score Rejection   → Telegram: Score Rejected
      ├─ INTERVIEW → Parse INTERVIEW Command  → Get/Update: Interview Scheduled → Email: Interview Invite → Telegram: Interview Scheduled
      ├─ RESULT    → Parse RESULT Command     → Decision Selected?
      │                                            ├─ Selected → Get/Update: Selected            → Email: Job Offer            → Telegram: Selected
      │                                            └─ Rejected → Get/Update: Interview Rejected   → Email: Interview Rejection  → Telegram: Interview Rejected
      └─ (none of the above) → Telegram: Invalid Command
```

## Applicant status flow

```
New → Shortlisted ─┐
   └→ Score Rejected│
                     ├→ Interview Scheduled → Selected (Job Offer)
                     └→                     → Interview Rejected
```

## Files in this repo

| File | Description |
|---|---|
| `HR_Hiring_Pipeline_-_Animaster_Studio.json` | n8n workflow export |
| `Hr_Hiring_Pipeline.png` | Full workflow diagram (n8n canvas view) |
| `HR_hiring_pipeline_reply.jpg` | Screenshot of the Telegram bot notifying HR of a new application, and its invalid-command help message |
| `Hr_reply.jpg` | Example job offer email sent by the workflow |

## Setup / requirements

- An [n8n](https://n8n.io/) instance (cloud or self-hosted)
- A Google Form set up to write responses into a Google Sheet with columns for Timestamp, Full Name, Email address, Phone Number, Position Applied For, Years of Experience, and Portfolio/GitHub URL
- Google Sheets credentials (OAuth2) in n8n, connected to that sheet
- Gmail credentials (OAuth2) in n8n, for sending welcome/status/offer emails
- A Telegram bot token (via [@BotFather](https://t.me/BotFather)) and HR's chat ID configured for notifications and commands

### To use it
1. Import `HR_Hiring_Pipeline_-_Animaster_Studio.json` into n8n.
2. Connect the Google Sheets, Gmail, and Telegram credentials.
3. Point the Google Sheets trigger at your applications sheet.
4. Activate the workflow.
5. New form submissions will automatically flow through intake; HR manages scoring, interviews, and decisions by messaging the Telegram bot with the `SCORE`, `INTERVIEW`, and `RESULT` commands shown above.
