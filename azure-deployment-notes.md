# Azure Deployment Notes

## Code Fixes Applied

### Bug 1 — Static file path mismatch
`artifacts/api-server/src/app.ts:38` was looking for frontend static files at
`../../reading-buddy/dist`, but Vite outputs to `../../reading-buddy/dist/public`.

**Fixed:** path updated to `../../reading-buddy/dist/public`.

### Bug 2 — Frontend build fails in CI
`artifacts/reading-buddy/vite.config.ts` throws if `PORT` or `BASE_PATH` are not set,
but the GitHub Actions workflow did not provide them. `BASE_PATH` also affects asset
URLs in the built output.

**Fixed:** added `PORT: "3000"` and `BASE_PATH: "/"` to the build step env in
`.github/workflows/main_miamaiers-reading-buddy-4.yml`.

---

## Azure App Service Configuration

In the Azure Portal, go to **App Service → Configuration → Application Settings** and add:

| Setting | Value | Notes |
|---|---|---|
| `NODE_ENV` | `production` | Enables static file serving |
| `DATABASE_URL` | `postgresql://...` | Your PostgreSQL connection string |
| `PORT` | *(leave unset)* | Azure sets this automatically to `8080` |

### Also verify

- **Stack**: Node.js **22** (App Service → Configuration → General Settings)
- GitHub secret `AZUREAPPSERVICE_PUBLISHPROFILE_5B3B2B98C25541AFBD981BF6716902D9` must
  exist in your repo under **Settings → Secrets and variables → Actions**

---

## Startup Command

The startup command is already set correctly in the workflow:

```
node artifacts/api-server/dist/index.mjs
```

---

## Database Schema (one-time setup)

The app does not auto-migrate on startup. Run this once against your production database
before first use:

```bash
cd lib/db
DATABASE_URL="<your-prod-db-url>" pnpm run push
```

---

## Deployment Flow

1. Push to `main`
2. GitHub Actions builds the app (Node.js 22, pnpm 10)
3. Built artifact is deployed to Azure App Service `miamaiers-reading-buddy-4`
4. Azure starts the server with `node artifacts/api-server/dist/index.mjs`
5. The server serves the React frontend from `artifacts/reading-buddy/dist/public`
   and the API from `/api/*`
