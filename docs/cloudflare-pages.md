# Cloudflare Pages deployment

## What this repository needs

VibeOffice is a static site: it uses classic browser scripts, not a framework. There is no build command or dependency installation. Cloudflare Pages should publish `public/` as its output directory. Routes such as `/quire/`, `/ledger/`, `/lectern/` and `/updater/` are directories with their own `index.html`; this is not an SPA, so there is deliberately no catch-all fallback that turns missing files into the Start Center. The root `404.html` prevents Pages from treating the project as an SPA when a file or route is missing.

`wrangler.toml` names the Pages project and sets the output directory. `public/_redirects` adds the trailing slash for app and updater URLs without dropping query strings. `public/_headers` sets safe response headers, revalidation for unversioned assets, one-day caching for fonts/icons/previews, long-lived caching for versioned offline ZIPs, and MIME types for the manifest, source assets and Office formats. The app already supplies its own per-page Content Security Policy; the header file does not override it.

Office documents are opened from the user's computer and exported as browser downloads; they are not files served from Pages. The Office MIME mappings are provided for any such files intentionally hosted there in the future.

## One-time setup

1. In Cloudflare, select the account that will host Pages and create an API token with **Account / Cloudflare Pages / Edit** permission.
2. In the GitHub repository, add these Actions secrets under **Settings → Secrets and variables → Actions → New repository secret**:
   - `CLOUDFLARE_API_TOKEN`: the token above.
   - `CLOUDFLARE_ACCOUNT_ID`: the account ID shown in Cloudflare.
3. From the repository root, install Wrangler, authenticate, create the project, and publish once:

   ```sh
   npm install -g wrangler
   wrangler login
   wrangler pages project create vibeoffice
   wrangler pages deploy
   ```

`wrangler.toml` makes the last command use `public/`. This manual first deployment publishes the source tree as-is; the service worker remains unstamped and therefore does not cache offline files yet. The workflow below creates a versioned copy before each production deployment and enables the site's offline/update behavior.

## Custom domain

In the Cloudflare dashboard, open **Workers & Pages → vibeoffice → Custom domains → Set up a custom domain** and add:

```text
office.giong.ai
```

Complete the DNS instructions Cloudflare presents for the `giong.ai` zone and wait for the domain's status to become **Active** and its certificate to be issued. Keep the Pages project hostname available for preview and rollback checks. The site currently keeps `vibeoffice.work` as its canonical/search-engine URL and update-notice source; `office.giong.ai` is enabled as an additional working and installable origin.

## Automatic production deployment

`.github/workflows/cloudflare-pages.yml` deploys to the `main` production branch after each push to `main`. It can also be started manually from `main` in **Actions → Cloudflare Pages → Run workflow**. The job copies `public/` to a temporary directory, stamps the commit version and creates the offline ZIP with `tools/web-release.sh`, then deploys that directory using the two repository secrets. It never edits the checked-in site. Wrangler's output directory remains `public/` for the manual command.

After the first workflow run, check the Pages deployment URL and then `https://office.giong.ai/`. Verify the app routes and their slashless redirects (`/quire`, `/ledger`, `/lectern`, `/updater`), open each app, test a refresh on an app URL, check the installed PWA/offline notice, and confirm the security and content-type headers in the browser's Network panel.

## Deployment checklist

- [ ] `CLOUDFLARE_API_TOKEN` and `CLOUDFLARE_ACCOUNT_ID` are configured as GitHub Actions secrets.
- [ ] The `vibeoffice` Pages project exists and the `main` branch is its production branch.
- [ ] `office.giong.ai` is attached to the Pages project; DNS and HTTPS show active.
- [ ] The Cloudflare Pages workflow succeeds on the intended `main` commit.
- [ ] `/`, `/quire/`, `/ledger/`, `/lectern/`, and `/updater/` load directly and after refresh.
- [ ] Slashless app/updater paths redirect to their slash form; unknown paths remain real 404s rather than SPA content.
- [ ] The browser reports `text/html`, JavaScript/CSS, `application/manifest+json`, `font/woff2`, and expected Office MIME types.
- [ ] `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, and `Permissions-Policy` are present.
- [ ] The Start Center offers installation and reports offline caching after the versioned production deploy.

## Rollback

For an immediate rollback, open **Workers & Pages → vibeoffice → Deployments**, select the last known-good production deployment, and choose **Rollback to this deployment**. Verify the custom domain and all three app routes. The repository and branch remain unchanged; a subsequent successful push to `main` deploys the current source again.

If the failure is in source, revert the offending commit and push that revert to `main`; the workflow will publish it as a new deployment. Avoid deploying an old source copy with the normal workflow: versions and offline caches are designed to move forward.

## Troubleshooting

| Symptom | Check |
|---|---|
| Workflow cannot authenticate | Confirm both secret names exactly and that the token has Cloudflare Pages **Edit** permission for the account ID provided. |
| Project not found | Create `vibeoffice` in the same Cloudflare account as `CLOUDFLARE_ACCOUNT_ID`; Wrangler deploys to the project named in `wrangler.toml`. |
| Pages deploy says output directory is missing | Run Wrangler from the repository root and verify `public/` exists; `wrangler.toml` sets `pages_build_output_dir = "./public"`. |
| Custom domain is pending or unavailable | Check that `office.giong.ai` is attached to this Pages project, the DNS record matches the dashboard instructions, and the certificate has finished provisioning. |
| App URL returns 404 or refresh fails | Confirm the deployment contains each app's `index.html` and the root `public/_redirects`; Pages directory indexes serve the slash form directly. |
| New release looks stale | Check the deployment commit and response `Cache-Control`; reload after revalidation. The `sw.js` response is `no-store`, and offline app assets update by the versioned ZIP. |
| Install works but offline resources do not appear | Confirm the deployment came from the GitHub workflow, not the initial raw `wrangler pages deploy`; the workflow stamps a non-`dev` version and builds the offline ZIP. |
| A resource is blocked by CSP | Preserve each page's existing CSP. Do not add a stricter global CSP in `_headers` without testing every app and the service worker. |
