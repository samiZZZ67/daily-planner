# Deployment & Operations Guide (`deploy.md`)

Comprehensive, production-ready deployment guide for the **Daily Telegram Software Engineering Micro-Lesson Bot** and the **Two-Stream Editorial News Scout**.

---

## 1. System & Workflow Architecture

The repository contains two distinct, complementary automated subsystems designed to run as serverless scheduled jobs:

```
                                  ┌────────────────────────┐
                                  │   GitHub Actions Cron  │
                                  │   or Host OS Crontab   │
                                  └───────────┬────────────┘
                                              │
                    ┌─────────────────────────┴────────────────────────┐
                    │                                                  │
                    ▼ (Daily at 06:00 UTC)                             ▼ (Every 12h: 05:00 & 17:00 UTC)
         ┌─────────────────────┐                            ┌─────────────────────┐
         │ Daily Lesson Bot    │                            │ Editorial Scout     │
         │ (bot.py)            │                            │ (curate.py)         │
         └──────────┬──────────┘                            └──────────┬──────────┘
                    │                                                  │
        1. Read state.json (topic cursor)                  1. Fetch 17 RSS/Atom feeds (48h window)
        2. Prompt Gemini 2.5 (3-key failover)              2. Filter out seen URLs (seen_articles.json)
        3. Validate Reference URLs (HEAD/GET)              3. Evaluate with Gemini 2.5 rubrics
        4. Generate SD3 Diagram (HF 3-token failover)      4. Validate links & compile review cards
        5. AST Format Markdown (telegramify-markdown)      5. Dispatch to Editor Private Chat
        6. Telegram: sendPhoto -> sendMessage              6. Atomically update seen_articles.json
        7. Advance & persist state.json
                    │                                                  │
                    ▼                                                  ▼
         [Public Telegram Channel]                         [Private Editor Chat / Group]
         - Standalone Concept Diagram                      - Stream 1 (Tech & AI) Drafts
         - Rich 4096-char Formatted Lesson                 - Stream 2 (Architecture & Craft) Drafts
```

### Subsystem 1: Daily Micro-Lesson Bot (`bot.py` -> `src/pipeline.py`)
- **Purpose**: Posts an intermediate-to-advanced software engineering micro-lesson with an architecture concept diagram daily.
- **Topic Progression**: Deterministic sequential traversal from `src/topics.py`. Safe circular wrap-around back to topic 0 after the last topic.
- **Fail-Fast Invariant**: `state.json` advances **only** after Telegram confirms successful delivery of both photo and text message. If any step fails, the job exits with code `1` and retries the same topic on the next execution.
- **Zero-Escaping Typography**: Compiles Markdown into Telegram `MessageEntity` objects via AST (`telegramify-markdown`), preventing syntax-breaking escaping errors with characters like `.`, `!`, `<`, `>`, and `&`.

### Subsystem 2: Editorial News Scout (`curate.py` -> `news_curator/main.py`)
- **Purpose**: Aggregates top-tier engineering blogs and tech newsletters across two broad streams:
  - **Stream 1: Big Tech, AI & Developer Ecosystem** (GitHub, Cloudflare, Stripe, Netflix, Meta, Changelog, Simon Willison, The New Stack, Pragmatic Engineer)
  - **Stream 2: Architecture, Systems & Software Craft** (InfoQ, AWS Architecture, Martin Fowler, ByteByteGo, Marc Brooker, Dan Luu, Phil Eaton, Antirez)
- **Recency & Deduplication**: Discards entries older than 48 hours and deduplicates against `news_curator/seen_articles.json` (auto-pruned after 30 days).
- **Human-in-the-Loop Review**: Dispatches curated review cards and ready-to-publish Markdown drafts to the editor's private chat so they can approve or forward them.

---

## 2. Credentials & Prerequisites Checklist

Collect these secrets before deploying:

### A. Google Gemini API (3-Key Failover Pool)
The bots implement multi-key failover rotation. If key #1 hits quota or rate limits, requests automatically fail over to key #2 and #3.
1. Go to [Google AI Studio](https://aistudio.google.com/).
2. Create or obtain 3 separate API keys: `API1_KEY`, `API2_KEY`, `API3_KEY`.
3. Recommended Model: `gemini-2.5-flash` (or `gemini-1.5-flash`).

### B. Hugging Face Inference API (3-Token Failover Pool)
Used by `src/image_generator.py` to synthesize conceptual architecture diagrams with `stabilityai/stable-diffusion-3-medium-diffusers`.
1. Go to [Hugging Face Settings -> Access Tokens](https://huggingface.co/settings/tokens).
2. Generate 3 **Read** tokens: `HF_IMAGE_TOKEN_1`, `HF_IMAGE_TOKEN_2`, `HF_IMAGE_TOKEN_3`.

### C. Telegram Bots & Chat IDs
You can use a single bot or two separate bots:
1. Open Telegram and talk to **@BotFather**.
2. Send `/newbot` to create your **Public Channel Bot** -> save `TELEGRAM_BOT_TOKEN`.
3. (Optional) Send `/newbot` to create a dedicated **Editorial Review Bot** -> save `REVIEW_BOT_TOKEN` (or reuse `TELEGRAM_BOT_TOKEN`).
4. **Get Channel Chat ID**:
   - Add your bot as an **Administrator** to your channel/group with **"Post Messages"** permission.
   - Post any text message into the channel.
   - Run `python get_chat_id.py` or query `https://api.telegram.org/bot<TOKEN>/getUpdates` to locate the numeric chat ID (e.g. `-1001234567890`). This is `TELEGRAM_CHAT_ID`.
5. **Get Review Chat ID**:
   - Open a chat with your Review Bot, click **Start** (or send `/start`).
   - Find your Telegram User ID using `@userinfobot` or check `/getUpdates`. This is `REVIEW_CHAT_ID`.

---

## 3. Configuration Reference (`.env`)

Create a `.env` file from `.env.example`:

| Environment Variable | Required | Description | Example / Default |
| :--- | :---: | :--- | :--- |
| `API1_KEY` | **Yes** | Primary Google Gemini API Key | `AIzaSy...` |
| `API2_KEY` | **Yes** | Secondary Gemini Failover Key | `AIzaSy...` |
| `API3_KEY` | **Yes** | Tertiary Gemini Failover Key | `AIzaSy...` |
| `GEMINI_MODEL_NAME` | **Yes** | Gemini model identifier | `gemini-2.5-flash` |
| `HF_IMAGE_TOKEN_1` | **Yes** | Primary Hugging Face Read Token | `hf_...` |
| `HF_IMAGE_TOKEN_2` | **Yes** | Secondary Hugging Face Token | `hf_...` |
| `HF_IMAGE_TOKEN_3` | **Yes** | Tertiary Hugging Face Token | `hf_...` |
| `TELEGRAM_BOT_TOKEN` | **Yes** | Telegram Bot Token for channel | `123456789:ABC...` |
| `TELEGRAM_CHAT_ID` | **Yes** | Destination channel or group ID | `-1001234567890` |
| `REVIEW_BOT_TOKEN` | *Scout* | Bot Token for editor review cards | `123456789:XYZ...` |
| `REVIEW_CHAT_ID` | *Scout* | Editor's private user ID or chat ID | `987654321` |
| `RECENCY_WINDOW_HOURS` | No | RSS article age limit in hours | `48` |
| `MAX_STORIES_PER_STREAM` | No | Top stories selected per stream | `5` |

---

## 4. Primary Deployment: GitHub Actions (Recommended, 100% Free)

This is the recommended serverless approach with zero hosting maintenance. GitHub Actions runs the cron triggers and commits updated state files back to your repository.

### Step 1: Push Code to GitHub
Ensure your repository is initialized and pushed to GitHub:
```bash
git init
git add .
git commit -m "feat: initial commit of telegram bot"
git remote add origin https://github.com/<your-username>/<your-repo>.git
git branch -M main
git push -u origin main
```

### Step 2: Configure GitHub Repository Secrets
In your GitHub repo, navigate to **Settings** -> **Secrets and variables** -> **Actions** -> click **New repository secret**:
Add each of the following secrets:
- `API1_KEY`
- `API2_KEY`
- `API3_KEY`
- `GEMINI_MODEL_NAME` (Value: `gemini-2.5-flash`)
- `HF_IMAGE_TOKEN_1`
- `HF_IMAGE_TOKEN_2`
- `HF_IMAGE_TOKEN_3`
- `TELEGRAM_BOT_TOKEN`
- `TELEGRAM_CHAT_ID`
- `REVIEW_BOT_TOKEN`
- `REVIEW_CHAT_ID`

### Step 3: Grant Workflow Write Permissions (Crucial for State Persistence)
To allow GitHub Actions to commit `state.json` and `seen_articles.json` back to your repository:
1. Go to **Settings** -> **Actions** -> **General**.
2. Scroll to **Workflow permissions**.
3. Select **Read and write permissions**.
4. Check **Allow GitHub Actions to create and approve pull requests**.
5. Click **Save**.

### Step 4: Verify Workflows
The repository includes two pre-configured workflows under `.github/workflows/`:
- `.github/workflows/daily-lesson.yml`: Runs daily at `06:00 UTC` to publish the micro-lesson.
- `.github/workflows/editorial-curator.yml`: Runs twice daily at `05:00 UTC` and `17:00 UTC` to scout news.

Both workflows feature `workflow_dispatch`, allowing you to trigger them manually at any time from the **Actions** tab in GitHub.

---

## 5. Alternative Deployment: Linux VPS / Server (Cron + Systemd)

If you prefer hosting on an Ubuntu/Debian VPS, AWS EC2, DigitalOcean Droplet, or Raspberry Pi:

### Step 1: Clone and Set Up Virtual Environment
```bash
git clone https://github.com/<your-username>/<your-repo>.git /opt/daily-telegram-bot
cd /opt/daily-telegram-bot/DailyTelegramNewsBot

python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
```

### Step 2: Configure Environment
```bash
cp .env.example .env
nano .env  # Populate your production secrets
```

### Step 3: Run Diagnostic Pre-flight Checks
```bash
# 1. Run unit test suite
python -m unittest discover -s tests

# 2. Check all 17 RSS feeds
python curate.py --check-feeds

# 3. Test Telegram review bot connection
python curate.py --test-bot
```

### Step 4: Configure Crontab
Edit the crontab for your user (`crontab -e`):
```cron
# --------------------------------------------------------------------------
# Daily Telegram News Bot & Editorial Scout Schedule
# --------------------------------------------------------------------------
SHELL=/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# Run Daily Micro-Lesson Bot every morning at 09:00 local (06:00 UTC)
0 6 * * * cd /opt/daily-telegram-bot/DailyTelegramNewsBot && .venv/bin/python bot.py >> /var/log/daily_lesson.log 2>&1

# Run Editorial News Scout twice a day at 08:00 and 20:00 (05:00 & 17:00 UTC)
0 5,17 * * * cd /opt/daily-telegram-bot/DailyTelegramNewsBot && .venv/bin/python curate.py >> /var/log/editorial_curator.log 2>&1
```

---

## 6. Alternative Deployment: Docker

You can containerize the bot with automated scheduling using **Supercronic** (a lightweight cron runner built for containers).

### `Dockerfile`
```dockerfile
FROM python:3.11-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

WORKDIR /app

# Install curl for supercronic installation
RUN apt-get update && apt-get install -y --no-install-recommends curl ca-certificates \
    && curl -fsSLO "https://github.com/aptible/supercronic/releases/download/v0.2.29/supercronic-linux-amd64" \
    && chmod +x "supercronic-linux-amd64" \
    && mv "supercronic-linux-amd64" "/usr/local/bin/supercronic" \
    && apt-get purge -y curl && apt-get autoremove -y && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# Setup crontab inside container
RUN echo "0 6 * * * cd /app && python bot.py" > /app/crontab && \
    echo "0 5,17 * * * cd /app && python curate.py" >> /app/crontab

CMD ["/usr/local/bin/supercronic", "/app/crontab"]
```

### `docker-compose.yml`
```yaml
version: '3.8'

services:
  telegram-bot:
    build:
      context: ./DailyTelegramNewsBot
      dockerfile: Dockerfile
    restart: unless-stopped
    env_file:
      - ./DailyTelegramNewsBot/.env
    volumes:
      # Mount state files to host so topic progression & seen history persist across container restarts
      - ./DailyTelegramNewsBot/state.json:/app/state.json
      - ./DailyTelegramNewsBot/news_curator/seen_articles.json:/app/news_curator/seen_articles.json
```

Run with:
```bash
docker compose up -d
docker compose logs -f
```

---

## 7. Diagnostics & Operational Verification

### CLI Diagnostic Commands
The bot includes built-in diagnostic and testing flags:

```bash
# 1. Probe all 17 RSS feeds across both streams and print status
python curate.py --check-feeds

# 2. Test Telegram review bot connection and send verification message
python curate.py --test-bot

# 3. Discover Telegram chat IDs from recent bot interactions
python get_chat_id.py

# 4. Run automated test suite
python -m unittest discover -s tests
```

### Single Execution Test Run
To trigger a live post manually without waiting for cron:
```bash
# Trigger the daily lesson immediately
python bot.py

# Trigger the editorial news scout immediately
python curate.py
```

---

## 8. Failure Modes, Invariants & Troubleshooting

| Issue / Symptom | Root Cause | Automated Recovery / Solution |
| :--- | :--- | :--- |
| **`429 Rate Limit` on Gemini** | Free tier RPM limit reached | Bot automatically catches exception, logs warning, and switches to next key in pool (`API1_KEY` -> `API2_KEY` -> `API3_KEY`). |
| **`503 Model Warming Up` on Hugging Face** | Cold start on SD3 Medium inference router | Bot reads server `estimated_time`, sleeps up to 30s, and retries. If persistent, rotates tokens. |
| **Telegram `chat not found`** | Bot was not started by user or not added to channel | - For private chat: Send `/start` to `@YourBotUsername`.<br>- For channel: Add bot as Administrator with "Post Messages" permission. |
| **Telegram Character Limits** | Captions on `sendPhoto` are capped at 1,024 chars | Architecture uses **Two-Stage Delivery**: clean image sent via `sendPhoto`, full 4,096-character formatted lesson sent via `sendMessage`. |
| **Markdown Parsing Error** | Special characters (`_`, `*`, `[`, `]`, `(`, `)`, `~`, `` ` ``, `>`, `#`, `+`, `-`, `=`, `|`, `{`, `}`, `.`, `!`) | Zero manual string escaping is used; `telegramify-markdown` compiles directly to Telegram AST `MessageEntity` objects with pre-computed UTF-16 byte offsets. |
| **Corrupted Code Blocks (`\n` shown literally)** | Gemini JSON mode double-escapes control characters | Handled automatically by `_repair_lesson_fields` in `src/lesson_generator.py` and `news_curator/editorial_evaluator.py`. |
| **Dead Reference Links (404)** | LLM hallucinated an outdated link | `src/url_validator.py` probes candidate URLs via `requests.head` (and fallback `stream=True requests.get`) and only publishes verified live URLs. |

---

## 9. Content Customization

### Adding or Editing Micro-Lesson Topics
All lesson topics are defined in [DailyTelegramNewsBot/src/topics.py](file:///c:/Users/yosee/Documents/daily%20plan/DailyTelegramNewsBot/src/topics.py) in `SEED_TOPICS`.
- You can freely append, reorder, or edit topics.
- Thanks to `src/topic_scheduler.py`, the index wraps around modulo `len(SEED_TOPICS)`, preventing index out-of-range crashes even if the topic list shrinks.

### Adding or Removing RSS Feeds
Feeds are registered in [DailyTelegramNewsBot/news_curator/sources.py](file:///c:/Users/yosee/Documents/daily%20plan/DailyTelegramNewsBot/news_curator/sources.py):
- Add a new `FeedSource(name="...", feed_url="...", site_url="...", stream_id=1)` to `STREAM_1_SOURCES` or `STREAM_2_SOURCES`.
- Run `python curate.py --check-feeds` to verify that the new feed responds with HTTP 200 and valid entries.
