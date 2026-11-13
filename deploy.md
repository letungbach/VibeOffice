DEPLOY.md
Deploy VibeOffice to Cloudflare Pages
This guide deploys VibeOffice to:

https://office.letstravel.vn

using:

Cloudflare Pages
Wrangler CLI
GitHub repository
Existing Cloudflare-managed DNS zone: letstravel.vn
Architecture
GitHub
   │
   ▼
Cloudflare Pages
   │
   ▼
office.letstravel.vn

No VPS
No Docker
No Backend
No Database
No Server Maintenance
VibeOffice is a static browser application.

All document processing happens on the client device.

Cloudflare only serves HTML, CSS, JavaScript, fonts and images.

Prerequisites
Already completed:

Cloudflare account
letstravel.vn managed by Cloudflare
Wrangler authenticated
Verify:

wrangler whoami
Expected:

You are logged in as ...
Install Wrangler
If not already installed:

npm install -g wrangler
Verify:

wrangler --version
Create Cloudflare Pages Project
Run once:

wrangler pages project create vibeoffice
Project name:

vibeoffice
Repository Structure
Expected:

VibeOffice/
├── public/
├── docs/
├── desktop/
├── tests/
├── README.md
├── LICENSE
└── ...
No build step is required.

The application is plain HTML and JavaScript.

Create wrangler.toml
Create:

name = "vibeoffice"

compatibility_date = "2026-11-13"
Create _headers
Create:

/*
  X-Frame-Options: SAMEORIGIN
  X-Content-Type-Options: nosniff
  Referrer-Policy: strict-origin-when-cross-origin

/*.js*  Cache-Control: public, max-age=3*536000, immutable*
/*.css
  Cache-Control: public, max-age=31536000, immutable

/*.png
  Cache-Control: public, max-age=31536000, immutable

/*.svg
  Cache-Control: public, max-age=31536000, immutable

/*.woff2
  Cache-Control:*public, max-age=31536000, immutabl*
Save*as:

public/_headers
*f deployment root is not public,*place the file at the effective Pages output root.

Create _redirects
Only required if direct URL refresh fails.

File:

/* /index.html 200
Save as:

public/_redirects
Do not use this file if Quire, Ledger and Lectern are already separate physical folders.

First Deployment
From repository root:

wrangler pages deploy . \
  --project-name vibeoffice
If the application entrypoint is inside public:

wrangler pages deploy ./public \
  --project-name vibeoffice
Verify Deployment
Cloudflare returns a URL similar to:

https://xxxxxxxx.vibeoffice.pages.dev
Open:

https://xxxxxxxx.vibeoffice.pages.dev
Verify:

Start Center loads
Quire loads
Ledger loads
Lectern loads
Open local file works
Save file works
PWA install still works
Attach Custom Domain
Open:

Cloudflare Dashboard
→ Workers & Pages
→ vibeoffice
→ Custom Domains
Add:

office.letstravel.vn
Cloudflare automatically creates the DNS record.

Verify DNS
Check:

nslookup office.letstravel.vn
or

dig office.letstravel.vn
Expected:

Cloudflare-managed hostname
Automatic GitHub Deployments
Create:

.github/workflows/cloudflare-pages.yml
Contents:

name: Deploy to Cloudflare Pages

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Deploy
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          command: pages deploy . --project-name=vibeoffice
GitHub Secrets
Repository Settings

Settings
→ Secrets and Variables
→ Actions
Create:

CLOUDFLARE_API_TOKEN
and

CLOUDFLARE_ACCOUNT_ID
Get account id from:

Cloudflare Dashboard
→ Right sidebar
→ Account ID
Rollback
List deployments:

wrangler pages deployment list \
  --project-name vibeoffice
Redeploy a previous commit:

git checkout <commit>
wrangler pages deploy . \
  --project-name vibeoffice
Troubleshooting
Blank Page
Check browser console:

F12
→ Console
Look for:

404
CORS
Module Loading Errors
CSS Missing
Verify deployment root:

.
vs

./public
404 Refresh Error
Add:

/* /index.html 200
to:

public/_redirects
Domain Not Working
Verify:

Workers & Pages
→ vibeoffice
→ Custom Domains
contains:

office.letstravel.vn
SSL Pending
Wait a few minutes for Cloudflare certificate provisioning.

Production Deployment Command
For future releases:

git pull

wrangler pages deploy . \
  --project-name vibeoffice
Production URL:

https://office.letstravel.vn