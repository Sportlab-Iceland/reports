# SportLab K5 Platform

Internal tool: upload a Cosmed K5 `.xlsx` export, get an auto-generated
report (identical to the interactive reports built earlier — same
charts, same V-slope/ventilatory-equivalents VT1/VT2 detection, same
granularity sliders, same custom chart builder), and share a private
view link with the athlete/customer.

## What's included

- `app/xlsx_parser.py` — reads the COSMED Data + Results sheets, bins
  breath-by-breath data into 14 stages
- `app/detection.py` — the same V-slope + ventilatory-equivalents VT1/VT2
  algorithm validated against Almar Viðarsson's real file (exact HR
  match at VT1, within 3 bpm at VT2 vs. the device's own detection)
- `app/report_renderer.py` — substitutes real data into the proven report
  template; auto-fills "Trend vs. Previous Test" from database history
- `app/models.py` — Athlete / Report database tables
- `app/main.py` — the web app (login, upload, dashboard, athlete
  history, shareable report links)
- `templates/report_template.html` — the actual K5 report (unchanged
  from what's been built and tested throughout this project)
- `templates/{login,dashboard,upload,athlete,base}.html` — the simple
  internal admin UI

## How it works

1. Staff log in with a shared password (`STAFF_PASSWORD` env var)
2. Upload a `.xlsx` file on the Upload page
3. The app parses it, detects VT1/VT2, and generates the full report —
   this takes about 1–2 seconds
4. You land on the finished report immediately; it's also saved and
   listed on the Dashboard and the athlete's history page
5. Every report gets a permanent link: `yoursite.com/report/SL-2026-0042`
   — send that link directly to the athlete/customer. No login needed
   to view a report (the ID itself is the access token — someone
   would need the exact link to see it, and links aren't listable
   without staff login)
6. Upload a second file for the same athlete (matched by name) and the
   "Trend vs. Previous Test" table auto-fills from the first report's
   real stored numbers — the delta calculation you've seen in earlier
   reports is now driven by actual history, not manual entry

## Running locally

```bash
pip install -r requirements.txt
cd app
SECRET_KEY=$(openssl rand -hex 32) STAFF_PASSWORD=yourpassword uvicorn main:app --reload
```

Visit `http://localhost:8000`, log in with `yourpassword`.

## Deploying to Render.com (recommended — free tier works for this)

**1. Push this project to a GitHub repo** (private is fine).

**2. Create a PostgreSQL database on Render**
- Render dashboard → New → PostgreSQL
- Free tier is fine for an internal tool
- Once created, copy the "Internal Database URL" — you'll need it next

**3. Create a Web Service on Render**
- Render dashboard → New → Web Service → connect your GitHub repo
- Runtime: Python 3
- Build command: `pip install -r requirements.txt`
- Start command: `cd app && uvicorn main:app --host 0.0.0.0 --port $PORT`
- Add environment variables:
  - `DATABASE_URL` — paste the Internal Database URL from step 2
  - `STAFF_PASSWORD` — whatever password your team should use
  - `SECRET_KEY` — any long random string (e.g. run `openssl rand -hex 32` locally and paste the result)
- Deploy

**4. Done** — Render gives you a URL like `sportlab-k5.onrender.com`.
Share that with your team for uploads, and the `/report/SL-2026-XXXX`
links it generates with customers.

### Notes on the free tier
Render's free web services sleep after inactivity and take ~30–60s to
wake on the next request — fine for an internal tool used a few times
a day, less fine if customers are clicking report links throughout the
day and hitting a cold start. If that matters, upgrade to Render's
cheapest paid tier (~$7/mo) for an always-on instance.

## Extending this later

- **Customer self-service upload**: add a second, more restricted
  upload flow/login tier — the architecture already separates "staff
  who upload" from "anyone with a report link who views," so this is
  additive, not a rewrite
- **Editing before sharing**: reports are fully editable in the browser
  (every field is `contenteditable`) but edits aren't currently saved
  back to the database — add a "Save Changes" button that POSTs the
  edited HTML back to `/report/{id}` to persist manual edits
- **Athlete matching**: currently matches athletes by exact name string;
  for a larger roster, switch to an explicit athlete-ID lookup or a
  fuzzy-match confirmation step on upload
