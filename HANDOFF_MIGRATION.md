# Invoice Application - Handoff Document
**Last Updated:** 2026-09-26

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Architecture](#architecture)
3. [Recent Changes](#recent-changes)
4. [Deployment Guide](#deployment-guide)
5. [Data Migration](#data-migration)
6. [Repository Structure](#repository-structure)
7. [Configuration](#configuration)
8. [Common Tasks](#common-tasks)
9. [Troubleshooting](#troubleshooting)
10. [Access & Credentials](#access--credentials)

---

## Project Overview

**Application Name:** GME Invoice Management System  
**Owner:** Gulmohammed Ansari (itransform007@gmail.com)  
**Purpose:** Multi-tenant invoice, quotation, and certificate management system

### Key Features
- Multi-tenant invoice generation with PDF export
- Quotations and credit/debit notes
- Certificate generation (Classic, Horizon, Modern styles)
- AMC (Annual Maintenance Contract) management
- Challan tracking
- Email delivery (Gmail SMTP)
- WhatsApp sharing
- Google Sheets integration (legacy)
- Google Firestore backend (current)

---

## Architecture

### Two-Repo Structure

#### 1. Frontend Repository
- **Repo:** https://github.com/gulmohammedansari/Justatest2
- **Branch:** `master`
- **Hosting:** GitHub Pages
- **URL:** https://gulmohammedansari.github.io/Justatest2/
- **Technology:** Single-page HTML application (vanilla JS)
- **File:** `index.html` (557KB - contains entire app)
- **Deployment:** Auto-deploys on push to master

#### 2. Backend Repository
- **Repo:** https://github.com/gulmohammedansari/gme-invoiceapp-backend
- **Branch:** `main`
- **Hosting:** Google Cloud Run
- **Region:** us-central1
- **Technology:** Node.js + Express
- **Deployment:** Manual via gcloud CLI

### Infrastructure

```
┌─────────────────────────────────────────────────────────┐
│  Frontend (GitHub Pages)                                │
│  https://gulmohammedansari.github.io/Justatest2/       │
│  - SPA with vanilla JS                                  │
│  - Communicates with backend via REST API               │
└────────────────┬────────────────────────────────────────┘
                 │
                 │ HTTPS
                 ▼
┌─────────────────────────────────────────────────────────┐
│  Backend (Google Cloud Run)                             │
│  Region: us-central1                                    │
│  - Node.js + Express                                    │
│  - PDF generation (Puppeteer + Chromium)                │
│  - Email delivery (Gmail SMTP)                          │
└────────┬────────────────────────┬───────────────────────┘
         │                        │
         │                        │
         ▼                        ▼
┌────────────────────┐   ┌───────────────────────┐
│  Google Firestore  │   │  Google Cloud Storage │
│  (Primary DB)      │   │  (Files/PDFs)         │
│  - Tenants         │   │  Bucket:              │
│  - Users           │   │  gme-invoiceapp-      │
│  - Invoices        │   │  prod-files           │
│  - Certificates    │   └───────────────────────┘
│  - Items, etc.     │
└────────────────────┘
```

---

## Recent Changes

### 2026-09-26: Certificate Title Made Bold + Website/Email Fields Added

**Certificate Title Bold:**
- Classic style: font-weight 400 → 700 (bold)
- Horizon style: font-weight 400 → 700 (bold)
- Modern style: font-weight 400 → 700 (bold)
- Applied to both frontend and backend PDF generation

**Commits:**
- Frontend: `b414888` "Make certificate title text bold across all styles"
- Backend: `6f484cb` "Make certificate title text bold in PDF generation"

**Website & Email Fields:**
- Added Website and Email fields to Business Profile settings (frontend)
- Made all Business Profile fields editable by tenants
- Backend now allows tenants to update their own company profile fields
- Fields: name, address, gstin, phone, state, website, email

**Commits:**
- Frontend: `f5858fc` "Add website and email fields to tenant Business Profile settings"
- Backend: `5538556` "Allow tenants to edit their own company profile fields including website and email"

### 2026-09-26: Certificate Font Size Increases

**Frontend Changes:**
- Classic certificate title: 32px → 42px → 50px
- Horizon certificate title: 36px → 46px → 50px
- Modern certificate badge: 16px → 22px → 32px
- Font family: UnifrakturMaguntia with fallbacks

**Backend Changes:**
- Synced certificate PDF generation fonts to match frontend
- Updated `src/lib/certificateHtml.js` with 50px font sizes

**Commits:**
- Frontend: `de795fd` "Increase certificate title font size to 50px"
- Backend: `317fa0a` "Increase certificate title font size to 50px"

### 2026-09-25: Certificate Font Enhancement
- Added UnifrakturMaguntia Gothic Blackletter font
- Bundled font files in backend repository
- Updated all three certificate templates

---

## Deployment Guide

### Frontend Deployment

Frontend deploys **automatically** when you push to GitHub:

```bash
cd "F:\Claude Project\InvoiceProject_OnPrem_Server - Copy"
git add index.html
git commit -m "Your commit message"
git push origin master
```

**Verification:**  
Visit https://gulmohammedansari.github.io/Justatest2/ (changes appear in ~30 seconds)

---

### Backend Deployment

Backend requires **manual deployment** to Google Cloud Run:

#### Prerequisites
- Google Cloud Shell access (or gcloud CLI installed locally)
- Project: `gme-invoiceapp-prod`
- Region: `us-central1`

#### Deployment Steps

**Option 1: From Google Cloud Shell (Recommended)**

```bash
# Navigate to backend directory
cd ~/gme-invoiceapp-backend

# Pull latest changes from GitHub
git fetch origin main
git pull origin main

# Verify latest commit
git log --oneline -1

# Deploy to Cloud Run
gcloud run deploy gme-invoiceapp-backend \
  --source . \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \
  --min-instances 0 \
  --max-instances 10 \
  --memory 512Mi \
  --cpu 1 \
  --timeout 60s \
  --set-secrets=SUPER_ADMIN_SECRET=SUPER_ADMIN_SECRET:latest,GMAIL_USER=GMAIL_USER:latest,GMAIL_APP_PASSWORD=GMAIL_APP_PASSWORD:latest \
  --set-env-vars=NODE_ENV=production,GCS_BUCKET_NAME=gme-invoiceapp-prod-files,ALLOWED_ORIGIN=https://gulmohammedansari.github.io
```

**Deployment time:** ~2-3 minutes

**Verification:**
1. Wait for "Service deployed successfully" message
2. Test at https://gulmohammedansari.github.io/Justatest2/
3. Try generating a certificate or invoice
4. Check that PDFs generate correctly

---

## Data Migration

### Overview
Migrate data from Google Sheets (legacy) to Google Firestore (current).

### Migration Script Location
- **Path:** `backend/scripts/migrate/migrate.js`
- **Entry point:** Orchestrates the entire migration
- **Dependencies:**
  - `scripts/migrate/dateUtils.js` - Date conversion
  - `scripts/migrate/transforms.js` - Row transforms for each sheet type
  - `scripts/migrate/sheetsClient.js` - Google Sheets API wrapper

### Prerequisites

1. **Authentication:**
   ```bash
   gcloud auth application-default login
   ```

2. **Service Account Permissions:**
   - Firestore Database User (write access)
   - Sheets Viewer (read access on all spreadsheets)

3. **Spreadsheet Sharing:**
   - Control spreadsheet shared with service account as Viewer
   - All tenant spreadsheets shared with service account as Viewer

### Control Spreadsheet
**ID:** `15Bsg1lYxHb4vOJ4DKa6cW32Q0dHSB6pzPAO7VC8ZVB0`

Contains:
- **Tenants** sheet (tenant metadata)
- **Users** sheet (user accounts)

### Migration Commands

**1. Dry Run (Preview only, no writes):**
```bash
cd ~/gme-invoiceapp-backend
CONTROL_SPREADSHEET_ID=15Bsg1lYxHb4vOJ4DKa6cW32Q0dHSB6pzPAO7VC8ZVB0 npm run migrate:dry-run
```

**2. Single Tenant Test (Real write, limited scope):**
```bash
CONTROL_SPREADSHEET_ID=15Bsg1lYxHb4vOJ4DKa6cW32Q0dHSB6pzPAO7VC8ZVB0 npm run migrate -- --tenant=TENANT_ID
```

**3. Full Migration (All tenants):**
```bash
CONTROL_SPREADSHEET_ID=15Bsg1lYxHb4vOJ4DKa6cW32Q0dHSB6pzPAO7VC8ZVB0 npm run migrate
```

### What Gets Migrated

**From Control Spreadsheet:**
- Tenants → `/tenants/{tenantId}`
- Users → `/users/{userId}`

**From Each Tenant's Spreadsheet (8 sheets):**
- Invoices → `/tenantData/{tenantId}/invoices/{docId}`
- Buyers → `/tenantData/{tenantId}/buyers/{docId}`
- Items → `/tenantData/{tenantId}/items/{docId}`
- Challans → `/tenantData/{tenantId}/challans/{docId}`
- Quotations → `/tenantData/{tenantId}/quotations/{docId}`
- Credit/Debit Notes → `/tenantData/{tenantId}/creditDebitNotes/{docId}`
- AMC → `/tenantData/{tenantId}/amc/{docId}`
- Certificates → `/tenantData/{tenantId}/certificates/{docId}`

**NOT Migrated:**
- Sessions sheet (short-lived, 4-hour TTL, recreated on login)

### Migration Behavior

- **Idempotent:** Safe to run multiple times (overwrites with same data)
- **Batched:** Writes in chunks of 450 ops (under Firestore's 500 limit)
- **Error tolerant:** Skips bad rows, logs errors, continues
- **Deterministic:** Same doc IDs on every run

### Post-Migration

1. **Verify data in Firestore Console:**
   - https://console.firebase.google.com/project/gme-invoiceapp-prod/firestore

2. **Test login and data access:**
   - Login to https://gulmohammedansari.github.io/Justatest2/
   - Verify invoices, certificates, items load correctly

3. **Optional: Keep Sheets as backup**
   - Sheets remain read-only
   - Backend no longer writes to Sheets after migration

---

## Repository Structure

### Frontend Repository (`Justatest2`)

```
.
├── index.html              # Main application file (557KB)
├── super-admin.html        # Super admin interface
├── index-failover-*.html   # Failover modes
├── font-test.html          # Font testing page
├── certificate-font-test.html
├── Code/                   # Additional scripts
├── backend/                # Local backend copy (not deployed)
└── screenshots/
```

### Backend Repository (`gme-invoiceapp-backend`)

```
backend/
├── src/
│   ├── index.js                    # Express app entry point
│   ├── routes/
│   │   ├── certificates.js         # Certificate CRUD + PDF
│   │   ├── invoices.js             # Invoice CRUD + PDF
│   │   ├── tenants.js              # Tenant management
│   │   ├── users.js                # User auth
│   │   └── ...
│   └── lib/
│       ├── certificateHtml.js      # Certificate PDF templates
│       ├── pdf.js                  # Puppeteer PDF generation
│       ├── email.js                # Gmail SMTP
│       └── firestore.js            # Firestore client
├── scripts/
│   ├── migrate/
│   │   ├── migrate.js              # Main migration script
│   │   ├── transforms.js           # Row transformations
│   │   ├── dateUtils.js            # Date conversion
│   │   └── sheetsClient.js         # Sheets API wrapper
│   └── check-storage.sh            # GCS storage checker
├── fonts/
│   └── UnifrakturMaguntia-Regular.ttf  # Bundled font
├── Dockerfile                      # Cloud Run container
├── package.json
├── DEPLOYMENT.md
├── DEPLOY_NOW.txt                  # Quick deploy reference
└── README.md
```

---

## Configuration

### Environment Variables (Backend)

Set via `--set-env-vars` in Cloud Run deploy:

| Variable | Value | Description |
|----------|-------|-------------|
| `NODE_ENV` | `production` | Runtime environment |
| `GCS_BUCKET_NAME` | `gme-invoiceapp-prod-files` | Cloud Storage bucket for PDFs |
| `ALLOWED_ORIGIN` | `https://gulmohammedansari.github.io` | CORS allowed origin |

### Secrets (Backend)

Set via `--set-secrets` in Cloud Run deploy:

| Secret Name | Secret Manager Key | Description |
|-------------|-------------------|-------------|
| `SUPER_ADMIN_SECRET` | `SUPER_ADMIN_SECRET:latest` | Super admin access token |
| `GMAIL_USER` | `GMAIL_USER:latest` | Gmail SMTP username |
| `GMAIL_APP_PASSWORD` | `GMAIL_APP_PASSWORD:latest` | Gmail app password |

**View/Edit Secrets:**  
https://console.cloud.google.com/security/secret-manager?project=gme-invoiceapp-prod

### Frontend Configuration

Hardcoded in `index.html`:

```javascript
var API_BASE = 'https://gme-invoiceapp-backend-[hash]-uc.a.run.app';
var FRONTEND_BASE = 'https://gulmohammedansari.github.io/Justatest2';
```

---

## Common Tasks

### Update Certificate Fonts

**Frontend (`index.html`):**
1. Find `.cert-title`, `.hz-title`, `.mc-badge-title`
2. Update `font-size` values
3. Commit and push to `master`

**Backend (`src/lib/certificateHtml.js`):**
1. Find the same CSS classes in each template function
2. Update `font-size` values
3. Commit and push to `main`
4. Deploy to Cloud Run

### Add a New Certificate Style

1. **Backend:** Add new template function in `src/lib/certificateHtml.js`
2. **Frontend:** Add matching template in `index.html` certificate section
3. Update certificate format selector dropdown
4. Deploy both frontend and backend

### Update Email Templates

**Location:** `backend/src/lib/email.js`

Templates:
- `sendInvoiceEmail()`
- `sendCertificateEmail()`
- `sendQuotationEmail()`

After changes:
1. Commit to backend repo
2. Deploy to Cloud Run
3. Test email delivery

### Add New Firestore Collection

1. Define schema in code
2. Update `backend/src/routes/` with CRUD endpoints
3. Update frontend to call new endpoints
4. If migrating from Sheets: update `scripts/migrate/transforms.js`

---

## Troubleshooting

### Frontend Issues

**Problem:** Changes not appearing on GitHub Pages  
**Solution:**
- Hard refresh: Ctrl + Shift + R (Windows) / Cmd + Shift + R (Mac)
- Check GitHub Actions for deployment status
- Wait 30-60 seconds after push

**Problem:** CORS errors in browser console  
**Solution:**
- Verify backend `ALLOWED_ORIGIN` env var matches frontend URL
- Redeploy backend if changed

### Backend Issues

**Problem:** Cloud Run deployment fails  
**Solution:**
- Check logs: `gcloud run services logs read gme-invoiceapp-backend --region us-central1`
- Verify Docker build: `docker build -t test .` locally
- Check Dockerfile syntax
- Ensure all secrets exist in Secret Manager

**Problem:** PDF generation fails  
**Solution:**
- Check Puppeteer/Chromium installation in Dockerfile
- Verify fonts are bundled correctly
- Check Cloud Run memory (512Mi minimum)
- Review error logs for missing dependencies

**Problem:** Email delivery fails  
**Solution:**
- Verify Gmail App Password in Secret Manager
- Check GMAIL_USER secret is correct email
- Ensure Gmail account has "Less secure app access" or uses App Password
- Review SMTP connection logs

### Migration Issues

**Problem:** `ENOENT: no such file or directory, open '/home/.../package.json'`  
**Solution:**
- Navigate to correct directory: `cd ~/gme-invoiceapp-backend`

**Problem:** Authentication failed  
**Solution:**
- Run: `gcloud auth application-default login`
- Or set `GOOGLE_APPLICATION_CREDENTIALS` env var

**Problem:** Permission denied on Sheets  
**Solution:**
- Share spreadsheets with service account email as Viewer
- Check service account has Sheets API enabled

**Problem:** Firestore permission denied  
**Solution:**
- Grant service account "Firestore Database User" role
- Check IAM permissions in GCP Console

---

## Access & Credentials

### GitHub Repositories

**Frontend:**
- URL: https://github.com/gulmohammedansari/Justatest2
- Access: Owner

**Backend:**
- URL: https://github.com/gulmohammedansari/gme-invoiceapp-backend
- Access: Owner

### Google Cloud Platform

**Project:** `gme-invoiceapp-prod`  
**Console:** https://console.cloud.google.com/home/dashboard?project=gme-invoiceapp-prod

**Key Services:**
- Cloud Run: https://console.cloud.google.com/run?project=gme-invoiceapp-prod
- Firestore: https://console.firebase.google.com/project/gme-invoiceapp-prod/firestore
- Cloud Storage: https://console.cloud.google.com/storage/browser?project=gme-invoiceapp-prod
- Secret Manager: https://console.cloud.google.com/security/secret-manager?project=gme-invoiceapp-prod

**Owner:** itransform007@gmail.com

### Application URLs

**Frontend:** https://gulmohammedansari.github.io/Justatest2/  
**Backend API:** Visible in Cloud Run console (auto-generated URL)

---

## Maintenance & Monitoring

### Regular Tasks

1. **Monitor Cloud Run costs:**
   - Check billing dashboard monthly
   - Review instance auto-scaling settings

2. **Review Cloud Storage usage:**
   - Run: `bash backend/scripts/check-storage.sh`
   - Clean up old PDFs if needed

3. **Update dependencies:**
   - Backend: `npm audit` and `npm update`
   - Check for Node.js LTS updates

4. **Backup Firestore data:**
   - Use Firestore export feature
   - Schedule: weekly or before major changes

### Logs

**Backend logs:**
```bash
gcloud run services logs read gme-invoiceapp-backend --region us-central1 --limit 100
```

**Real-time logs:**
```bash
gcloud run services logs tail gme-invoiceapp-backend --region us-central1
```

---

## Support & Contact

**Developer:** Gulmohammed Ansari  
**Email:** itransform007@gmail.com  
**GitHub:** @gulmohammedansari

---

## Version History

| Date | Version | Changes |
|------|---------|---------|
| 2026-09-26 | 1.3 | Certificate font size increased to 50px |
| 2026-09-25 | 1.2 | UnifrakturMaguntia font added |
| 2026-09-11 | 1.1 | Deployment guide updated |
| 2026-09-06 | 1.0 | Initial handoff document |

---

## Quick Reference Card

### Deploy Frontend
```bash
git add index.html
git commit -m "Message"
git push origin master
```

### Deploy Backend
```bash
cd ~/gme-invoiceapp-backend
git pull origin main
gcloud run deploy gme-invoiceapp-backend --source . --region us-central1 \
  --allow-unauthenticated --min-instances 0 --max-instances 10 \
  --memory 512Mi --cpu 1 --timeout 60s \
  --set-secrets=SUPER_ADMIN_SECRET=SUPER_ADMIN_SECRET:latest,GMAIL_USER=GMAIL_USER:latest,GMAIL_APP_PASSWORD=GMAIL_APP_PASSWORD:latest \
  --set-env-vars=NODE_ENV=production,GCS_BUCKET_NAME=gme-invoiceapp-prod-files,ALLOWED_ORIGIN=https://gulmohammedansari.github.io
```

### Migrate Data
```bash
cd ~/gme-invoiceapp-backend
CONTROL_SPREADSHEET_ID=15Bsg1lYxHb4vOJ4DKa6cW32Q0dHSB6pzPAO7VC8ZVB0 npm run migrate:dry-run
CONTROL_SPREADSHEET_ID=15Bsg1lYxHb4vOJ4DKa6cW32Q0dHSB6pzPAO7VC8ZVB0 npm run migrate
```

---

**End of Handoff Document**
