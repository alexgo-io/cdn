# ALEX CDN

Static files served at `cdn.alexgo.io`, `cdn.tesseralab.co` and `cdn.tessera.pe`
(the same files on each), including the
T-token metadata (`tessera/t-*.json`) and icons that the tokens' on-chain
metadata points to.

## How it deploys

The files live in the Cloudflare R2 bucket `cdn` (account `alexgo-io`), attached
to all three hostnames as custom domains (since 2026-11-22; Vercel before). `.github/workflows/r2-sync.yml` keeps the bucket in step with `main`:

- **Merge to `main`:** `rclone sync` uploads new and changed files and deletes
  files removed from the repo. A file's URL is its path in the repo, e.g.
  `tessera/t-spacex.json` → `https://cdn.tesseralab.co/tessera/t-spacex.json`.
- **Pull request:** the same sync as a dry run. The job log lists what merging
  would upload and delete.
- A sync deletes at most 25 files. A bigger clean-up fails until the limit in
  the workflow is raised in the same PR.
- Files are served with `Cache-Control: public, max-age=300`, so an edit is live
  within about five minutes.
- Not uploaded: `.github/`, this README, `.gitignore`, `.DS_Store`, `Thumbs.db`.
- rclone is a pinned release checked by SHA-256 (the workflow's `Install rclone`
  step). Any error fails the run: there is no silent second attempt.

Keep files under R2's 5 GiB single-upload limit. Large media (video) belongs
somewhere else.

## One-time Cloudflare setup

1. **Bucket.** R2 → Create bucket (e.g. `cdn`). Leave the r2.dev public URL off
   except to test.
2. **API token.** R2 → Manage API tokens → Create: *Object Read & Write*,
   limited to that bucket. It gives an Access Key ID and a Secret Access Key.
3. **GitHub.** In this repository's Settings → Secrets and variables → Actions:
   - secrets `R2_ACCESS_KEY_ID` and `R2_SECRET_ACCESS_KEY`
   - variables `R2_ACCOUNT_ID` (the Cloudflare account ID) and `R2_BUCKET`

   The workflow is skipped until `R2_BUCKET` is set.
4. **First upload.** Actions → Sync to R2 → Run workflow (on `main`).
5. **Connect each hostname** (`cdn.tesseralab.co`, `cdn.tessera.pe`,
   `cdn.alexgo.io`): delete any existing `cdn` record in the zone's DNS, then
   in the bucket's Settings → Custom Domains → Connect Domain, or with wrangler:

   ```
   wrangler r2 bucket domain add cdn --domain cdn.tesseralab.co --zone-id <zone id> --min-tls 1.2
   ```

   Cloudflare creates the proxied record; wait until ownership and SSL both
   read `active` (`wrangler r2 bucket domain list cdn`).
6. **CORS.** Browser wallets fetch the metadata JSON from other origins. A CORS
   policy on the bucket covers every hostname:

   ```
   echo '{"rules":[{"allowed":{"origins":["*"],"methods":["GET","HEAD"],"headers":["*"]},"maxAgeSeconds":3600}]}' > cors.json
   wrangler r2 bucket cors set cdn --file cors.json
   ```

   Responses then carry `Access-Control-Allow-Origin: *`, with or without an
   `Origin` header (checked 2026-11-22; Cloudflare does not cache these
   hostnames, `cf-cache-status: DYNAMIC`).
7. **Remove the Vercel project** once the hostnames serve from R2, so it
   cannot answer for them again.

Check after cut-over:

```
curl -sI https://cdn.tesseralab.co/tessera/t-spacex.json   # 200, application/json, access-control-allow-origin: *
curl -sI https://cdn.tessera.pe/tessera/tokenicon_T-SpaceX.svg   # 200, image/svg+xml
curl -sI https://cdn.alexgo.io/logos/ALEX-BANANA_Token.png   # 200
```
