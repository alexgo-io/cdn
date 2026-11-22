# ALEX CDN

Static files served at `cdn.alexgo.io` and `cdn.tesseralab.co`, including the
T-token metadata (`tessera/t-*.json`) and icons that the tokens' on-chain
metadata points to.

## How it deploys

The files live in a Cloudflare R2 bucket, attached to both hostnames as custom
domains. `.github/workflows/r2-sync.yml` keeps the bucket in step with `main`:

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
5. **Cut over each hostname** (`cdn.alexgo.io`, `cdn.tesseralab.co`): delete
   its CNAME to Vercel in the zone's DNS, then in the bucket's Settings →
   Custom Domains → Connect Domain. Cloudflare creates the proxied record.
6. **CORS.** Browser wallets fetch the metadata JSON from other origins. In
   each zone: Rules → Transform Rules → Response Header Transform, for requests
   to the hostname, set `Access-Control-Allow-Origin: *`. A response-header rule
   adds it to every response, cached or not.
7. **Remove the Vercel project** once both hostnames serve from R2, so it
   cannot answer for them again.

Check after cut-over:

```
curl -sI https://cdn.tesseralab.co/tessera/t-spacex.json   # 200, application/json, access-control-allow-origin: *
curl -sI https://cdn.alexgo.io/logos/ALEX-BANANA_Token.png   # 200
```
