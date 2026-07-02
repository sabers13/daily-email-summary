# imap-gpt-digest

A daily email digest for university (or any IMAP) mailboxes. It fetches your
unread mail, uses a GPT model to triage it into urgency buckets and extract a
dated action list, and pushes the result to Telegram. Runs unattended and for
free on GitHub Actions — no server required.

It was built for a mailbox that receives a lot of German university mail and
job-application replies, so triage is tuned to surface deadlines, exam
registration windows, and interview invitations. Everything in the digest is
written in English, translating non-English subjects.

## What you get

Every morning, a Telegram message like:

```
📬 Thu 02 Jul — 7 new email(s)

🔴 URGENT
• Prüfungsamt — Exam registration closes tomorrow
   ↳ Register for the Machine Learning exam by 03 Jul

🟡 THIS WEEK
• Career Service — Working-student role at ACME
   ↳ Reply requested by Friday

🔵 FYI
• Library — Opening hours during the holidays

📅 ACTION LIST
• 2026-07-03: Register for Machine Learning exam
```

## How it works

1. **Fetch** — connects to your IMAP server (TLS, port 993 by default) and
   pulls unread mail from the last `LOOKBACK_HOURS` (default 24). Mail is read
   with `BODY.PEEK` on a read-only mailbox, so **nothing is ever marked read**
   on the server. RFC 2047 headers (umlauts etc.) are decoded, threads are
   de-duplicated, and the newest mail is kept if there's a backlog.
2. **Triage** — sends sender/subject/preview (never your credentials) to the
   OpenAI API and gets back structured JSON, which is rendered into the digest
   in Python. Email contents are treated as untrusted data — the prompt never
   follows instructions found inside an email.
3. **Notify** — posts to Telegram, splitting long digests to stay under
   Telegram's message-size cap. Every network step retries with backoff, and
   failures send you a notice instead of failing silently.

## Setup

### 1. Fork / clone this repo

Use it as a template or fork it into your own account.

### 2. Create a Telegram bot

1. Message [@BotFather](https://t.me/BotFather), send `/newbot`, and follow the
   prompts. Copy the **bot token** it gives you.
2. Open your new bot and press **Start** (the bot can't message you until you
   do).
3. Get your numeric chat id — message [@userinfobot](https://t.me/userinfobot),
   or open `https://api.telegram.org/bot<TOKEN>/getUpdates` after messaging
   your bot and read `chat.id`.

### 3. Add your secrets and the IMAP host

In the repo: **Settings → Secrets and variables → Actions**.

Add these **secrets**:

| Secret | Value |
|---|---|
| `TU_USER` | Your IMAP login username (for many universities this is *not* your email address) |
| `TU_PASS` | Your IMAP password |
| `OPENAI_API_KEY` | An OpenAI API key from platform.openai.com |
| `TELEGRAM_TOKEN` | Your bot token from BotFather |
| `TELEGRAM_CHAT_ID` | Your numeric chat id |

Add this **variable** (the Variables tab — a hostname isn't a secret):

| Variable | Value |
|---|---|
| `IMAP_SERVER` | Your provider's IMAP host, e.g. `imap.example.edu` |

With the [GitHub CLI](https://cli.github.com/):

```bash
gh secret   set TU_USER          # you'll be prompted to paste the value
gh secret   set TU_PASS
gh secret   set OPENAI_API_KEY
gh secret   set TELEGRAM_TOKEN
gh secret   set TELEGRAM_CHAT_ID
gh variable set IMAP_SERVER --body "imap.example.edu"
```

### 4. Run it

- **Manually:** Actions tab → *Daily UniMail Digest* → **Run workflow**, or
  `gh workflow run daily-digest.yml`.
- **On schedule:** it runs every morning at **07:00 Europe/Berlin**. GitHub
  cron is UTC-only and ignores daylight saving, so the workflow schedules both
  candidate UTC hours and a gate step runs the digest only at the right local
  hour — exactly one fires per day. Adjust the crons in
  [`.github/workflows/daily-digest.yml`](.github/workflows/daily-digest.yml)
  for your timezone.

> GitHub disables scheduled workflows in a repo after 60 days of no activity
> (it emails you first). Any commit, or one click in the Actions tab,
> re-enables it.

## Configuration

All optional, set as repo **variables** (or env vars locally):

| Variable | Default | Meaning |
|---|---|---|
| `IMAP_PORT` | `993` | IMAP TLS port |
| `LOOKBACK_HOURS` | `24` | How far back mail counts as "new" |
| `NOTIFY_MODE` | `always` | `always` = every digest buzzes your phone; `urgent` = digest is always sent but silent unless the URGENT bucket is non-empty |

The GPT model is set by the `MODEL` constant near the top of
[`check_unimail_gpt.py`](check_unimail_gpt.py); a cheaper model is noted in the
comment there. At daily-digest volume, API cost is on the order of a couple of
cents per month.

## Run locally

```bash
pip install "openai>=1.40" "requests>=2.31,<3"

export IMAP_SERVER="imap.example.edu"
export TU_USER="your-username"
export TU_PASS="your-password"
export OPENAI_API_KEY="sk-..."
export TELEGRAM_TOKEN="123456:ABC..."
export TELEGRAM_CHAT_ID="123456789"

python check_unimail_gpt.py
```

## Security notes

- No credentials live in the code or in git — they're read from environment
  variables, which on GitHub Actions come from encrypted repo secrets.
- The mailbox is opened read-only and mail is fetched with `BODY.PEEK`, so the
  tool never changes anything on your mail server.
- Only sender, subject, and a short body preview are sent to the OpenAI API.
- Email bodies are treated as untrusted input; the triage prompt will not act
  on instructions embedded in an email (prompt-injection resistance).

## License

MIT — see [LICENSE](LICENSE).
