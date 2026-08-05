# Helio Solar — Appointment Generation System

## What This Project Is
A full-stack appointment management system for Helio Solar's 1099 appointment setter team.
Setters submit qualified leads via a mobile form. The lead is sent to a pool of sales reps via
SMS/email. The first rep to claim it gets the appointment. Setters can log in to a portal to
track appointment status through the full pipeline.

## Tech Stack
- **Backend**: Python FastAPI on Render (same pattern as `aurora-zoho-sync`)
- **Frontend**: Mobile-first HTML/JS (same pattern as `helio-project-monitor`)
- **SMS**: Twilio (already active on the Helio account)
- **CRM**: Zoho (Leads → Deals → Installs)
- **File storage**: Utility bill photos attached to Zoho Lead records

## System Architecture

### Flow
1. Setter submits form (customer info + qualification answers + utility bill photo)
2. Backend creates a Zoho **Lead** record with all info + attachment
3. Backend sends SMS (Twilio) + email to all available reps simultaneously
4. First rep to click claim link → backend locks the appointment to that rep, others get "already claimed"
5. Backend auto-converts Lead → **Deal** in Zoho and assigns to claiming rep
6. If no rep claims within timeout → admin alert (SMS + email)
7. Rep conducts appointment → logs outcome in Zoho (Sat / No Show)
8. If sat → logs Win / Loss in Zoho Deal stage
9. If sold → Deal moves to **Install** module in Zoho
10. Setter portal shows appointment status pulled live from Zoho

### Claim Timeout Logic
- Fixed window: configurable (start with 15 min, adjust based on team behavior)
- Hard deadline: 2 hours before appointment time (whichever is sooner)
- On timeout: SMS + email to admin (Harry)

### Setter Portal
- Login-based (setters have accounts, each sees only their own appointments)
- Shows all submitted appointments with current status
- Status states: Pending Claim → Claimed → Sat / No Show → Sold / Not Sold / Rescheduled
- Once sold: shows Install module stage from Zoho

## Zoho Modules
- **Leads**: created on form submission, stores all setter data + utility bill
- **Deals**: auto-converted from Lead when rep claims appointment
- **Installs**: existing module, setter portal reads stage from here once sold
- Rep assignment lives on the Deal record

### Key Zoho Fields (Installs module)
- `Commissions_Paid` — integer %, values 0/80/100 (percentage of commission paid)
- `Commission_Paid_Amount` — actual dollar amount paid
- `Commissions_Fully_Paid` — boolean

## Connections & Credentials

### Zoho CRM API
- Credentials stored in environment variables on Render
- Existing working code pattern in `aurora-zoho-sync/main.py` (Google Drive)
- Uses OAuth2 client credentials flow
- Modules: Leads, Deals, Contacts (see existing main.py for auth pattern)

### Aurora Solar API (for reference — may not be needed in this project)
- Base URL: `https://api.aurorasolar.com`
- API Key: `sk_prod_4ff27baddd6bfbc1b2d02167`
- Tenant ID: `25effab7-3d19-42d5-9fef-fbf53ac71ab8`
- Auth: `Authorization: Bearer {API_KEY}`

### Twilio (SMS)
- Already active on the Helio account and integrated with Zoho
- Credentials in Zoho account / Render env vars
- Used for: rep claim notifications, admin fallback alerts

### Google Sheets Service Account (for reference)
- Service account file: `/Users/harrydohnert/.claude/credentials/gsa_helio.json`
- Client email: `zoho-calendar-bridge@install-calendar-access.iam.gserviceaccount.com`

### Render
- Existing server: `https://aurora-zoho-sync.onrender.com`
- This project gets its own Render service (new repo → new service)
- Pattern: FastAPI app in `main.py`, deployed via GitHub push

### GitHub
- Org: `hdohnert-helio`
- Existing repos: `helio-project-monitor`, `aurora-zoho-sync`
- New repo for this project to be created under `hdohnert-helio`

## Existing Systems (Do Not Confuse)
- **HTML Pipeline Dashboard**: `helio-project-monitor` repo → GitHub Pages
- **Google Sheets Cashflow**: Sheet ID `1ktCKriA4W97Cxy-bubTD2zSP8W1X8fP52BLXvElkp5g`
- **Render Server**: `aurora-zoho-sync` — commissions, cashflow, webhook handling
- **Commissions Sheet**: ID `1JUXFXJJFOpbzNAnl-UH1bN_HIoYvFQtbnuwSFYA8L5I`

## Form Fields (Setter Submission)
**Customer Info:**
- First Name, Last Name
- Address (street, city, state, zip)
- Phone, Email
- Appointment Date + Time

**Qualification Questions:**
1. Is this the homeowner? (Yes/No)
2. Are they on a low-income electricity plan? (Yes/No)
3. Is their monthly electric bill over $150? (Yes/No)

**Attachments:**
- Utility bill photo (required)

**Setter Info (auto from login):**
- Setter name, setter ID

**Notes:**
- Free text field

## Key Design Decisions
- Setter accounts stored in app database (not Zoho) — simple username/password or magic link
- Rep pool: starts with 3 reps, configurable
- Claim mechanism: unique URL per appointment, first click wins (backend locks with DB transaction)
- Utility bills: uploaded to backend, attached to Zoho Lead via API
- Status feedback to setter: pulled from Zoho in real time (no separate status DB)
