# Email Automation & Reminder System

A beginner-friendly portfolio project for managing scheduled, personalized
email reminders from CSV files. The command-line program defaults to dry-run;
live email is only available with the explicit `--send` option.

**Problem:** Manually tracking follow-ups makes it easy to miss due dates,
repeat work, or send an incorrectly personalized message.

**Simple explanation:** Keep contacts and reminders in CSV files, write message
templates, and preview the personalized result before deciding whether to use
the protected live-send option.

**Technical overview:** A Python service loads and validates CSV records,
evaluates one-off or recurring local-time schedules, renders text templates,
and produces report rows. A local FastAPI API reuses that service for previews
and manages reminder CSV rows. A Next.js dashboard calls the API.

## Documentation

- [Project guide](docs/PROJECT_GUIDE.md) - architecture, setup, security, and
  GitHub portfolio guidance.
- [Proof plan](docs/PROOF_PLAN.md) - a six-day manual evidence plan and
  screenshot checklist. No screenshots have been captured yet.
- [Interview prep](docs/INTERVIEW_PREP.md) - concise project pitches and
  interview questions with answers.

The instructions document the current files and manifests; commands have not
been runtime-verified as part of this documentation update.

## Quick setup

Use Python 3.10 or newer. From the project root, create and activate a virtual
environment, then install the packages used by the API and optional `.env`
support.

**Windows PowerShell**

```powershell
cd "C:\Email Automation & Reminder System"
py -3 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
```

**macOS**

```bash
cd "/path/to/Email Automation & Reminder System"
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

With the environment active, preview due reminders with `python main.py`.
See the [project guide](docs/PROJECT_GUIDE.md) for API, dashboard, live-send,
and GitHub setup instructions.

## Data formats

### `data/contacts.csv`

Required columns:

```csv
email,name,team
alex.jones@example.com,Alex Jones,Engineering
```

### `data/reminders.csv`

Required columns:

```csv
reminder_id,recipient_email,subject,template_file,send_at,repeat_rule,enabled
R001,alex.jones@example.com,Project Check-in Reminder,project_checkin.txt,2026-12-01 09:00,none,true
```

Set `enabled` to `true`, `1`, `yes`, or `y` to preview a reminder. Other values are skipped.

## Templates

Templates are text files inside `templates/`. Use fields such as `{name}`, `{team}`, and `{subject}`.

```text
Hello {name},

{subject}
```

Template values come from the matching contact and reminder row.

## Python modules

The command-line app is split into reusable files under `src/`:

- `config.py` - project-root file paths and CSV column names.
- `data_loader.py` - CSV loading, required-column checks, and enabled values.
- `scheduler.py` - local date/time parsing and due-occurrence calculations.
- `emailer.py` - SMTP settings validation and secure message delivery.
- `reporting.py` - application logger, delivery state, and CSV report.
- `template_engine.py` - template-field validation and personalization.
- `service.py` - one reminder-processing cycle, without CLI parsing.
- `cli.py` - command-line options and the optional worker loop.

The root `main.py` only launches `src.cli.main()`. Importing these modules does
not parse arguments, start the worker, or send email.

The existing FastAPI preview endpoint calls the same service for a dry-run and
avoids writing a CLI report:

```python
from src.service import process_reminder_cycle

result = process_reminder_cycle(send=False, save_report=False)
preview_rows = result.rows
```

The service returns the cycle rows and status flags. The API always calls it
with `send=False`; neither the API nor the dashboard can send email.

## Local API

The API provides:

- `GET /health` - returns `{"status": "ok"}`.
- `GET /reminders` - returns reminder rows from `data/reminders.csv`.
- `POST /reminders` - creates a reminder, validates its fields, and assigns a
  unique `reminder_id`.
- `PATCH /reminders/{reminder_id}` - updates provided reminder fields. Send
  `{"enabled": false}` to cancel a reminder; its CSV row and delivery history
  are preserved.
- `POST /preview` - runs a dry-run preview and returns its result rows. Send an
  optional `as_of` value using `YYYY-MM-DD HH:MM`; without it, the API uses the
  current local time. The preview does not send email or write a report.

Reminder creation and updates validate email, subject, template filename,
schedule time, repeat rule, and enabled state. Template names must refer to
existing `.txt` files inside `templates/`. Exact duplicate reminder content
(recipient, subject, template, schedule, and repeat rule) returns `409`;
missing reminder IDs return `404`, and invalid data returns `422`. CSV changes
are written atomically.

Install the requirements if needed, then start the API locally from the project
root:

```powershell
python -m pip install -r requirements.txt
uvicorn src.api:app --host 127.0.0.1 --port 8000
```

On macOS, use the same commands in a terminal after activating `.venv`.

For example, a preview request body can be `{"as_of": "2026-12-03 10:00"}`.
The API has no send or worker endpoint and does not load SMTP settings.
For local dashboard development, FastAPI permits only these browser origins:
`http://localhost:3000` and `http://127.0.0.1:3000`, with GET/POST/PATCH
methods and no credentials.

## Next.js dashboard

Start the API in one terminal as described above. In a second terminal, set up
the dashboard from the project root:

```powershell
cd web
Copy-Item .env.local.example .env.local
npm install
npm run dev
```

On macOS, from the project root:

```bash
cd web
cp .env.local.example .env.local
npm install
npm run dev
```

Open `http://localhost:3000`. The dashboard reads
`NEXT_PUBLIC_API_BASE_URL` from `web/.env.local`, defaulting to
`http://127.0.0.1:8000`. This public variable is only an API URL; do not put
secrets in frontend environment variables. The dashboard can show API health
and reminders, create and edit reminder rows, cancel reminders without
deleting their CSV history, and request dry-run previews. It has no email-send
controls.

## Schedule rules

`send_at` must use `YYYY-MM-DD HH:MM` and is interpreted as local computer time.
The `repeat_rule` column accepts:

- `none` - one occurrence at `send_at`.
- `daily` - repeats every day at the same local time, starting on the `send_at` date.
- `weekly` - repeats every seven days at the same local time, starting at `send_at`.

Only the most recent scheduled occurrence at or before the selected time is
previewed for a recurring reminder. Earlier missed occurrences are not replayed
or listed separately. A reminder with a future next occurrence is reported as
`NOT_DUE`. Disabled reminders remain `SKIPPED`.

## Dry-run preview (default)

Without `--send`, the program does not connect to SMTP and does not send email.

```powershell
cd "C:\Email Automation & Reminder System"
python main.py
```

To preview at a chosen local date and time:

```powershell
python main.py --as-of "2026-12-03 10:00"
```

For a repeating dry-run worker that checks immediately and then every 60
seconds:

```powershell
python main.py --daemon --interval 60
```

The worker uses the actual local time for every cycle. Since dry-run does not
mark reminders as sent, it may preview the same due reminder again on a later
cycle. `--as-of` is only for a one-off dry-run and cannot be combined with
`--daemon`.

The preview is written to `outputs/email_report.csv`, and application messages are written to `logs/app.log`.

## SMTP setup and live sending

Install the declared dependency before using `.env` support:

```powershell
python -m pip install -r requirements.txt
```

Copy `.env.example` to `.env` and fill in the values for your SMTP provider:

```powershell
Copy-Item .env.example .env
```

The application reads `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`,
`SMTP_PASSWORD`, `SMTP_FROM_EMAIL`, and `SMTP_FROM_NAME`. Port `587` uses
STARTTLS; port `465` uses SSL. Keep `.env` private and never commit real
credentials.

Live sending requires the explicit option:

```powershell
python main.py --send
```

For a one-off live send, the program exits with a nonzero code if any reminder
has `FAILED` or `SENT_STATE_UNRECORDED` status.

Live mode always evaluates reminders against the current local time.
`--as-of` is available only in dry-run mode; combining it with `--send` is
rejected.

To run a live worker, specify both `--send` and `--daemon`:

```powershell
python main.py --send --daemon --interval 60
```

The first cycle runs immediately; later cycles start after the selected
interval. The interval must be a positive integer number of seconds and
defaults to 60. In live worker mode, failed reminders are not retried
automatically: the worker finishes its current cycle, logs the failure, then
stops with a nonzero exit code.

Stop either worker with **Ctrl+C**. It logs a shutdown message and exits.

**Before sending to real contacts, replace the sample recipient in
`data/reminders.csv` with an inbox you control and verify the message and
report there first.** The sample contact addresses are fictional and use
`example.com`.

Successfully accepted deliveries are recorded in
`outputs/delivery_state.json`, keyed by reminder ID and occurrence time. Later
runs skip an occurrence already marked `SENT`. Dry-run mode reads this state to
show those occurrences as `SKIPPED` but never modifies the file.

If SMTP may have accepted an email but the state file cannot be updated, the
report records `SENT_STATE_UNRECORDED` and the run stops. Manually reconcile
that delivery before retrying to avoid sending a duplicate.

Statuses include:

- `DRY_RUN_PREVIEW` - the email body was personalized and previewed safely.
- `SENT` - the SMTP server accepted the email.
- `SENT_STATE_UNRECORDED` - SMTP may have accepted the email, but its delivery
  state was not saved; reconcile manually before retrying.
- `SKIPPED` - the reminder is disabled, already sent, or already attempted in
  the current live cycle.
- `NOT_DUE` - its next scheduled occurrence is later than the selected time.
- `FAILED` - schedule, input, contact, template, or SMTP processing failed;
  check the report's `error` column.

The report's `occurrence_time` column shows the scheduled occurrence selected
for due reminders.

## GitHub portfolio

Suggested repository name: `email-automation-reminder-system`. The sample CSV
contacts are fictional and use `example.com`; they are safe portfolio examples.
Keep `.env`, `.venv/`, `web/.env.local`, Python caches, `logs/*.log`, and
generated report/state files out of Git. The root `.gitignore` includes these
rules. See [PROJECT_GUIDE.md](docs/PROJECT_GUIDE.md) for a safe outer-root
upload checklist, suggested repository description and topics, and commit
examples. Do not upload the nested project folder or the
`Email-Automation-Reminder-System.zip` archive.
