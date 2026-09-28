# Inbox cleaning

This project contains scripts meant to clean your Gmail inbox.

## How to use

0. Set your Gmail email and password in `.env`, and run in background `0_server_decision_model.py`
1. Run `ingest_gmail_emails_to_db.py` to get all your Gmail emails into `emails.db` (this drops emails.db and builds it again)
2. Run `label_emails.py` to label all emails with the different labels, using JevK5 locally
3. Run `label_unique_newsletters.py` to get all unique newsletters from precedent `newsletters` labeled emails + label these emails with the label `newsletter:{{ fromEmail }}` + generates a `unique_newsletter.json`
4. Previous step generates a `unique_newsletters.json` with all newsletter sender emails with a true/false (all true by default). Pass to `false` any emails you want cleared out
5. Run `label_to_remove_emails.py` to add the label `TO_REMOVE` to emails based on precedent labels. This script does not actually delete any emails, it just adds a label
6. Run `remove_emails.py` to actually remove the emails with the label `TO_REMOVE`. This actually calls the Gmail API and removes selected emails. In the database, the label `TO_REMOVE` is replaced by `REMOVED`

## Email in database

### Schema

- ID: string (primary key)
- from_email: string
- gmail_email_id: string
- sent_at: date
- was_replied_to: boolean (default: false)
- labels: label[]

### Possible labels, with their associated rule

The rules are editable in the `2_label_emails.py` file.

- `newsletter`: Email is coming from a newsletter (clues: html email, wording...)
- `functionnal`: Email was useful once at the time (ex: account created, password resets, magic links...) + Email is older than 10 days
- `has_attachments`: Email contains any sort of attachments
- `payment_related`: Email is about some financial transaction (monthly subscription payment, order confirmed, invoice...)
- `conversational`: Email appears to be sent only to me, for me (meaning, not a marketing email, not an automated outbound email either). Seems there is a real person behind the email.
- `newsletter_category:{{ newsletter_category }}`: Applies if previously labelled with newsletter label (default: `newsletter_category:none`). Categories includes :
  - marketing
  - AI
  - sales
  - solo_business
  - automation
  - content_creation

And 3 "special" labels:

- `PROTECTED`: Email that cannot be removed by the system, its rules are set in the `2_label_emails.py`.
- `TO_REMOVE`: Email labelled by the system or manually to be removed in the next run of `5_remove_emails.py`
- `REMOVED`: Email that still exists in database but have been removed from Gmail inbox.

#### Rules for special labels (personal, can be changed)

- `PROTECTED` = `has_attachments` || `was_replied_to` || `sent_at < 10 days`
- `TO_REMOVE` = `!PROTECTED` && (
  (`newsletter` && `newsletter not in NEWSLETTERS_WHITELIST`)
  || `spam`
  || (`functionnal` && `sent_at > 10 days`)
)

## Tech

The project runs 100% locally.

The project uses a single SQLite3 Database to store emails, runs deterministic rules for was_replied_to and has_attachments, and uses the open-weight JevK5 model to run the system one labelling system.
Under the hood, the program runs llama.cpp to create a server on port 8080 that serves the JevK5 model.

## Benchmark

Inference has been tested on a Macbook Pro 2021 16GB RAM.

Average labelling time per email: ~2s

## App features

- See all emails and sort / filters through them by labels
- See + whitelist / blacklist newsletters
- Graphs dashboard based on email data
