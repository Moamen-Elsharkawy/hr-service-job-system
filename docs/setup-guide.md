# Setup Guide — HR Service Job System (Phase 1)

Complete setup from zero to running system. Follow sections in order.

---

## Section 1: VPS Requirements

- Ubuntu 22.04 LTS
- 1 vCPU, 2 GB RAM minimum
- 20 GB disk
- Public IP address
- Recommended providers: Hetzner CX22 (~$4/mo) or DigitalOcean Basic ($6/mo)
- Open port 5678 in firewall (or use a reverse proxy on port 443)

---

## Section 2: Install n8n with Docker

SSH into the VPS, then run:

```bash
mkdir ~/n8n && cd ~/n8n
```

Create `docker-compose.yml`:

```yaml
version: "3.8"
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=YOUR_SERVER_IP_OR_DOMAIN
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - WEBHOOK_URL=http://YOUR_SERVER_IP_OR_DOMAIN:5678/
      - GENERIC_TIMEZONE=Africa/Cairo
      - TZ=Africa/Cairo
    volumes:
      - n8n_data:/home/node/.n8n
volumes:
  n8n_data:
```

> If you have a domain with SSL, change `http` to `https` and `N8N_PROTOCOL=https`.
> The `GENERIC_TIMEZONE=Africa/Cairo` setting makes cron expressions use Cairo time —
> the daily-digest workflow uses `0 20 * * *` (8 PM) assuming this is set.

Start n8n:

```bash
docker compose up -d
```

Open `http://YOUR_SERVER_IP:5678` in a browser and create your owner account.

---

## Section 3: Google OAuth Setup

**Step 1 — Create Google Cloud Project:**
1. Go to console.cloud.google.com
2. Click "Select a project" -> "New Project"
3. Name: `HR Service` -> Create

**Step 2 — Enable APIs:**
1. APIs & Services -> Enable APIs and Services
2. Search and enable: `Google Sheets API`
3. Search and enable: `Google Drive API`

**Step 3 — OAuth Consent Screen:**
1. APIs & Services -> OAuth consent screen
2. User Type: External -> Create
3. App name: `HR Service n8n`
4. User support email: your Gmail
5. Developer contact: your Gmail
6. Save and Continue through all steps
7. Add test user: your Gmail address

**Step 4 — Create Credentials:**
1. APIs & Services -> Credentials -> Create Credentials -> OAuth 2.0 Client ID
2. Application type: Web application
3. Name: `n8n HR Service`
4. Authorized redirect URIs: `http://YOUR_SERVER_IP:5678/rest/oauth2-credential/callback`
5. Create -> copy Client ID and Client Secret

**Step 5 — Add Credential in n8n:**
1. n8n -> Settings -> Credentials -> Add Credential
2. Search: `Google Sheets OAuth2 API`
3. **Name it exactly:** `Google Sheets (HR Service)`
4. Paste Client ID and Client Secret
5. Click "Sign in with Google" -> authorize with the Google account that owns the spreadsheet
6. Save

---

## Section 4: Telegram Bot Setup

**Step 1 — Create Bot:**
1. Open Telegram, search for `@BotFather`
2. Send: `/newbot`
3. Bot name: `HR Service Bot`
4. Username: `hrservice_YOURNAME_bot` (must be unique, end in `bot`)
5. Copy the bot token (format: `123456789:ABCdefGHI...`)

**Step 2 — Add Credential in n8n:**
1. n8n -> Settings -> Credentials -> Add Credential
2. Search: `Telegram API`
3. **Name it exactly:** `Telegram (HR Service Bot)`
4. Paste the bot token
5. Save

**Step 3 — Get Admin Chat ID:**
1. Open Telegram and send any message to your new bot
2. Open in browser: `https://api.telegram.org/bot<YOUR_TOKEN>/getUpdates`
3. Find `"chat":{"id": NUMBER}` in the response — that number is your chat ID
4. Copy it — you will need it when importing the daily-digest workflow

---

## Section 5: Create Google Form

Create the job submission form manually at forms.google.com.

1. New blank form
2. Title: `إضافة وظيفة جديدة - HR Service`
3. Add fields in this exact order:

| Field | Type | Required |
|---|---|---|
| اسم الشركة | Short answer | Yes |
| المسمى الوظيفي | Short answer | Yes |
| المجال | Dropdown | Yes |
| المدينة | Dropdown | Yes |
| نوع الشغل | Multiple choice | Yes |
| الراتب | Short answer | No |
| طريقة التواصل أو رابط التقديم | Short answer | Yes |
| وصف الوظيفة | Paragraph | No |

Dropdown options for **المجال**: `IT/تقنية`, `محاسبة ومالية`, `مبيعات`, `هندسة`, `تسويق`, `موارد بشرية`, `طب وصحة`, `تعليم`, `أخرى`

Dropdown options for **المدينة**: `القاهرة`, `الإسكندرية`, `الجيزة`, `المنصورة`, `أخرى`

Multiple choice options for **نوع الشغل**: `دوام كامل`, `دوام جزئي`, `عن بعد`, `فريلانس`

4. Responses tab -> click Sheets icon -> "Create a new spreadsheet"
5. Name the spreadsheet: `HR Service - الوظايف`
6. Click Create
7. The spreadsheet opens — note the Spreadsheet ID from the URL:
   `https://docs.google.com/spreadsheets/d/`**`YOUR_SPREADSHEET_ID`**`/edit`
8. In the spreadsheet, rename the sheet tab to: `استجابات الفورم`

---

## Section 6: Add "الوظايف المنظمة" Sheet

1. In the same spreadsheet, click `+` to add a new sheet tab
2. Rename it: `الوظايف المنظمة`
3. In row 1, add these headers in columns A through J:

```
التاريخ | اسم الشركة | المسمى الوظيفي | المجال | المدينة | نوع الشغل | الراتب | التواصل | الوصف | الحالة
```

Make sure the header row is row 1 and matches exactly — n8n writes by column header name.

---

## Section 7: Import Workflows

### Import job-form-processor.json

1. n8n -> Workflows -> click `+` -> Import from File
2. Select `workflows/job-form-processor.json`
3. Open the imported workflow
4. Find the node **"Watch Form Responses"** (Google Sheets Trigger):
   - Re-link credential to `Google Sheets (HR Service)`
   - Replace `YOUR_SPREADSHEET_ID` with your actual Spreadsheet ID
5. Find the node **"Read Organized Sheet"**:
   - Re-link credential to `Google Sheets (HR Service)`
   - Replace `YOUR_SPREADSHEET_ID` if needed
6. Find the node **"Append to Organized Sheet"**:
   - Re-link credential to `Google Sheets (HR Service)`
   - Replace `YOUR_SPREADSHEET_ID` if needed
7. Save -> toggle Active ON

### Import daily-digest.json

1. n8n -> Workflows -> Import from File
2. Select `workflows/daily-digest.json`
3. Open the imported workflow
4. Find **"Read Organized Sheet"** -> re-link credential + replace `YOUR_SPREADSHEET_ID`
5. Find **"Send Daily Digest"** (Telegram node):
   - Re-link credential to `Telegram (HR Service Bot)`
   - Replace `YOUR_ADMIN_CHAT_ID` with the numeric chat ID from Section 4 Step 3
6. Save -> toggle Active ON

---

## Section 8: Test the Full System

**Test 1 — Form to Sheet:**
1. Open the Google Form and submit a test job (e.g., company: "شركة تجريبية", title: "مطور")
2. Wait up to 5 minutes (trigger polls every 5 min)
3. Open the spreadsheet -> "الوظايف المنظمة" tab
4. A new row should appear with today's date and status=active

**Test 2 — Duplicate Detection:**
1. Submit the same form with identical company + title
2. Wait 5 minutes
3. Check "الوظايف المنظمة" — the row count should NOT increase

**Test 3 — Daily Digest:**
1. In n8n, open the daily-digest workflow
2. Click "Execute Workflow" (manual run)
3. Check your Telegram — the digest message should arrive in Arabic
4. If no message: open the execution log in n8n and check which node stopped

---

## Troubleshooting

**Trigger not firing:** Check that the workflow is Active (toggle is ON). Check n8n execution log for errors. Verify the credential is linked and the spreadsheet ID is correct.

**Arabic column headers not matching:** Open the n8n trigger node, run it manually once, and inspect the output. The JSON keys should match your form field labels exactly. If there is a mismatch (extra space, different character), fix the header in the sheet or update the expression in "Extract Fields".

**Telegram message not sending:** Verify the bot token and chat ID. Make sure you sent a message to the bot first (Telegram bots cannot initiate conversations with users who have never messaged them).

**Duplicate check not working:** Check that "الوظايف المنظمة" has data in it and headers are in row 1 with exact Arabic text.
