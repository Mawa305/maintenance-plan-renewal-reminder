# Maintenance Plan Renewal Reminder

A daily n8n workflow for an HVAC business that reminds customers when their maintenance plan is about to expire. It checks the renewals table every morning, works out how many days are left, writes a short reminder email with OpenAI, and sends it through Gmail. Each customer gets one reminder per stage (30, 14, and 7 days), and failures go to Telegram.

## Architecture

```
Schedule Trigger (daily, 09:00)
   → Get many rows (Supabase: renewals)
   → Code in JavaScript (days left, reminder stage, action)
   → Switch
        ├── send    → Message a model (OpenAI) → Send a message (Gmail)
        │                ├── success → Update a row1 (Supabase: reminder_status = sent_<stage>)
        │                └── error   → Send a text message (Telegram)
        └── invalid → Update a row (Supabase: reminder_status = invalid)
```

## Features

* Runs every day at 09:00 Karachi time
* Works out days left until the plan end date and picks the stage: 30, 14, or 7 days
* Sends each stage only once. The `reminder_status` field stops repeat emails
* Marks customers with a bad email or date as `invalid` instead of sending anything
* AI writes a short reminder that matches the stage: gentle at 30 days, helpful at 14, polite but urgent at 7
* The prompt forbids invented prices or discounts
* Sent reminders are recorded with the date in `last_reminder_date`
* Failed emails are reported to Telegram with the customer name, stage, and error

## Tech Stack

* n8n 2.x (workflow automation)
* Supabase (renewals table)
* OpenAI (reminder text)
* Gmail API (customer emails)
* Telegram Bot API (failure alerts)

## Setup

1. Create a Supabase table named `renewals` with these columns:

```
id | customer_name | email | plan_type | plan_end_date | reminder_status | last_reminder_date
```

   Use `plan_end_date` in `YYYY-MM-DD` format. Leave `reminder_status` empty for new customers.

2. Import `maintenance_plan_renewal_reminder.json` into n8n.
3. Add credentials: Supabase, OpenAI, Gmail OAuth2, and Telegram. Assign them to the matching nodes.
4. In the **Message a model** node, replace `[PAYMENT_LINK]` in the prompt with your real renewal page link.
5. Replace `YOUR_TELEGRAM_CHAT_ID` in the Send a text message node with your chat ID.
6. Test with one or two rows that have a plan end date 30, 14, or 7 days from today. Check the email, then the `reminder_status` value.
7. Activate the workflow when the test works.

## Reminder Stages

| Days left | Stage | Status written after send |
|---|---|---|
| 0 to 7 | 7d | `sent_7d` |
| 8 to 14 | 14d | `sent_14d` |
| 15 to 30 | 30d | `sent_30d` |

A customer moves to the next stage automatically. If their status already matches the current stage, nothing is sent.

## Sample Row

```
customer_name: Ali Raza
email:         ali.test@example.com
plan_type:     Yearly AC Maintenance
plan_end_date: 2026-11-20
```

If today is 14 November 2026, this customer is at 6 days left, so the 7d reminder is sent.

## Project Notes

* Built as part of my AI Automation portfolio.
* The reminder text is written by the model, but the stage, the date, and the send decision are all set in code, so the model cannot change the schedule.
* Only the OpenAI message node uses `gpt-4o-mini`. Change its model in that node if you want another model.
