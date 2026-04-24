# Deployment Fixes — How We Got It Running

This document records every change made to get the application successfully deployed
to Azure App Service. Use this as a checklist if you ever need to set up deployment
from scratch.

---

## Fix 1 — Rewrote the GitHub Actions Workflow

**File:** `.github/workflows/main_reading-buddy-2.yml`

Azure's auto-generated workflow template uses `npm install` and `npm run build`.
This project requires `pnpm`. The entire install/build section was replaced.

### What the broken template had:
```yaml
- name: npm install, build, and test
  run: |
    npm install
    npm run build --if-present
    npm run test --if-present
```

### What it was replaced with:

**a) Install pnpm BEFORE Node.js setup.**
pnpm must be installed first. If Node.js setup runs before pnpm exists, it falls
back to npm — which this project's preinstall guard script rejects with an error.

```yaml
- name: Install pnpm
  uses: pnpm/action-setup@v4
  with:
    version: 10
    run_install: false   # prevents pnpm from auto-running install here
```

**b) Set up Node.js AFTER pnpm, with pnpm caching enabled.**
```yaml
- name: Set up Node.js
  uses: actions/setup-node@v4
  with:
    node-version: '22'
    cache: 'pnpm'
```

**c) Install dependencies with pnpm.**
```yaml
- name: Install dependencies
  run: pnpm install
```

**d) Build with pnpm, passing required environment variables.**
`NODE_ENV=production` optimizes the output. `PORT` and `BASE_PATH` are required
by the Vite config or it throws an error during the frontend build.
```yaml
- name: Build
  run: pnpm -r --if-present run build
  env:
    NODE_ENV: production
    PORT: "3000"
    BASE_PATH: "/"
```

**e) Copy only built output into a clean deploy/ folder.**
The original template uploaded the entire repo (`.`). Azure only needs the compiled
files. Uploading source code wastes time and space.
```yaml
- name: Prepare deployment package
  run: |
    mkdir -p deploy/artifacts/api-server
    mkdir -p deploy/artifacts/reading-buddy
    cp -r artifacts/api-server/dist deploy/artifacts/api-server/dist
    cp -r artifacts/reading-buddy/dist deploy/artifacts/reading-buddy/dist

- name: Upload artifact for deployment job
  uses: actions/upload-artifact@v4
  with:
    name: node-app
    path: deploy/        # upload only the built files, not the whole repo
```

---

## Fix 2 — Express v5 Wildcard Route

**File:** `artifacts/api-server/src/app.ts`

The server crashed on startup with:
```
PathError [TypeError]: Missing parameter name at index 1: *
```

**Cause:** This project uses Express v5, which upgraded the `path-to-regexp`
library. In Express v4, a bare `*` was a valid wildcard for catch-all routes.
In Express v5 it is not — wildcards must be named.

**Change:**
```typescript
// Before (Express v4 syntax — crashes on Express v5)
app.get("*", (_req, res) => {

// After (Express v5 syntax)
app.get("/{*path}", (_req, res) => {
```

This is the SPA fallback route — it catches any URL that doesn't match an API
route and returns `index.html`, letting the React app handle routing in the browser.

---

## Fix 3 — Azure Portal: Startup Command

**Where:** Azure Portal → App Service → Configuration → General settings → Startup Command

Azure does not know how to start this app by default (it looks for a root-level
`server.js` or `app.js`, which don't exist here). Without this setting, Azure
shows a generic "Your web app is running and waiting for your content" placeholder.

**Set the Startup Command to:**
```
node artifacts/api-server/dist/index.mjs
```

> Note: The `startup-command` parameter in the GitHub Actions workflow does NOT
> work with publish-profile authentication or Windows App Service plans. It must
> be set manually in the Azure Portal.

---

## Fix 4 — Azure Portal: Application Settings

**Where:** Azure Portal → App Service → Configuration → Application settings

Two environment variables must be set for the app to work correctly:

| Setting | Value | Why it's needed |
|---|---|---|
| `NODE_ENV` | `production` | Tells Express to serve the React frontend's static files. Without this, the server runs in development mode and the frontend is never served. |
| `DATABASE_URL` | `postgresql://...` | The PostgreSQL connection string. The server crashes on startup without a valid database connection. |

> `PORT` does **not** need to be set here. Azure App Service sets it automatically.

---

## Fix 5 — GitHub Secret

**Where:** GitHub repo → Settings → Secrets and variables → Actions

The workflow authenticates with Azure using a publish profile. This credential
must be stored as a GitHub secret — never in the code.

**Steps to set it up:**
1. In the Azure Portal, open your App Service
2. Click **Download publish profile** in the Overview toolbar
3. Open the downloaded `.xml` file and copy all the contents
4. In GitHub, go to Settings → Secrets and variables → Actions
5. Create a secret named exactly: `AZUREAPPSERVICE_PUBLISHPROFILE_3370F419EA4E4C149976872D74731CFD`
6. Paste the publish profile contents as the value

> If you ever recreate the Azure App Service or reset the publish profile,
> you must download a new profile and update this secret.

---

## Summary Checklist

Use this when setting up deployment from scratch:

- [ ] Workflow file uses `pnpm/action-setup@v4` **before** `actions/setup-node@v4`
- [ ] Workflow build step has `NODE_ENV`, `PORT`, and `BASE_PATH` env vars
- [ ] Workflow uploads only the `deploy/` folder, not the whole repo
- [ ] `app.ts` SPA fallback route uses `"/{*path}"` not `"*"`
- [ ] Azure App Service startup command set to `node artifacts/api-server/dist/index.mjs`
- [ ] Azure Application Setting `NODE_ENV` = `production`
- [ ] Azure Application Setting `DATABASE_URL` = your PostgreSQL connection string
- [ ] GitHub secret `AZUREAPPSERVICE_PUBLISHPROFILE_3370F419EA4E4C149976872D74731CFD` contains a valid publish profile
