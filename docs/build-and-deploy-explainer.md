# How This Application Is Built and Deployed

## What You're Building

This app has two pieces that work together:

- **Frontend** — a React app that runs in the user's browser (like a smart webpage)
- **Backend** — an Express server that runs on Azure, handling API requests and serving the frontend

Both are written in **TypeScript**, which is JavaScript with type checking added. Neither the browser nor Node.js can run TypeScript directly, so a **build step** compiles everything down to plain JavaScript first.

---

## The Repo Structure

```
miamaiers-reading-buddy/          ← root (the whole monorepo)
├── artifacts/
│   ├── reading-buddy/            ← the React frontend app
│   └── api-server/               ← the Express backend server
├── lib/
│   ├── db/                       ← database access (Drizzle + PostgreSQL)
│   ├── api-zod/                  ← shared data shapes used by both front & back
│   └── api-client-react/         ← React hooks that call the API
├── pnpm-workspace.yaml           ← tells pnpm all these folders are one workspace
└── .github/workflows/            ← GitHub Actions CI/CD pipeline
```

This layout is called a **monorepo** — multiple related packages living in one Git repository. The `lib/` packages are shared code consumed by both `artifacts/`.

---

## Step 1 — You Push Code to GitHub

When you run `git push` and your code lands on the `main` branch, GitHub sees the push and automatically triggers the workflow defined in `.github/workflows/main_reading-buddy.yml`.

---

## Step 2 — GitHub Actions Spins Up a Build Machine

GitHub provisions a fresh Ubuntu virtual machine (called a **runner**) in the cloud. It has nothing on it yet.

---

## Step 3 — Dependencies Are Installed

```yaml
- uses: pnpm/action-setup@v4 # installs pnpm itself
- run: pnpm install # installs all packages
```

**pnpm** is the package manager (like npm but faster). `pnpm install` reads `pnpm-workspace.yaml` and all the `package.json` files across the repo, then downloads every third-party library listed as a dependency into a `node_modules/` folder. This includes React, Express, Drizzle, Vite, esbuild — everything.

---

## Step 4 — The Build Runs

```yaml
- run: pnpm -r --if-present run build
```

The `-r` flag means "recursively run the `build` script in every package." This triggers two separate builds:

### 4a — Frontend Build (Vite)

Vite is the build tool for the React app. See the **What Does Vite Do?** section below for a full explanation. Its output lands in `artifacts/reading-buddy/dist/public/` — a handful of static files: `index.html`, some `.js` bundles, and `.css` files. These are what the browser will actually download and run.

### 4b — Backend Build (esbuild)

esbuild is the build tool for the Express server. It:

1. **Compiles TypeScript** — converts `.ts` files to JavaScript
2. **Bundles** — takes your code plus Express, Drizzle, Pino (logging), etc. and packs them into **one single file**: `artifacts/api-server/dist/index.mjs`

`.mjs` is just a JavaScript file that uses the modern ES Module format. A single bundled file is much easier to deploy than thousands of individual files.

---

## Step 5 — The Deployment Package Is Assembled

```bash
mkdir -p deploy/artifacts/api-server
mkdir -p deploy/artifacts/reading-buddy
cp -r artifacts/api-server/dist   deploy/artifacts/api-server/dist
cp -r artifacts/reading-buddy/dist deploy/artifacts/reading-buddy/dist
```

This copies only the compiled output (not source code) into a `deploy/` folder. That's all Azure needs — the raw TypeScript source isn't required at runtime.

The `deploy/` folder looks like:

```
deploy/
└── artifacts/
    ├── api-server/
    │   └── dist/
    │       └── index.mjs          ← the entire Express server, one file
    └── reading-buddy/
        └── dist/
            └── public/
                ├── index.html
                ├── assets/
                │   ├── index-abc123.js
                │   └── index-abc123.css
```

---

## Step 6 — The Package Is Uploaded to Azure

```yaml
- uses: azure/webapps-deploy@v3
  with:
    app-name: "miamaiers-reading-buddy-4"
    publish-profile: ${{ secrets.AZURE_APP_SERVICE_PUBLISH_PROFILE }}
```

GitHub uses the **publish profile** (a secret stored in your GitHub repo settings under Settings → Secrets and variables → Actions — never in code) to authenticate with Azure and deploy the `deploy/` folder to your Azure App Service.

---

## Step 7 — Azure Runs the App

Azure App Service (Node 22) starts your app by running:

```
node --enable-source-maps ./dist/index.mjs
```

This starts the Express server. Here's what the server does at runtime:

1. **API routes** — any request to `/api/...` is handled by your Express route handlers (your business logic)
2. **Static files** — in production, all other requests serve files from `reading-buddy/dist/public/`
3. **SPA fallback** — any URL that doesn't match a file (e.g. `/home`, `/profile`) returns `index.html`, letting the React app handle routing in the browser

So **one process** serves both the API and the frontend. The browser gets `index.html`, loads the JavaScript bundle, and then React takes over rendering the UI and making API calls back to `/api/...`.

---

## The Full Picture

```
You push to main
        ↓
GitHub Actions runner starts
        ↓
pnpm install (download all libraries)
        ↓
Vite builds frontend → static HTML/CSS/JS files
esbuild builds backend → single index.mjs file
        ↓
deploy/ folder assembled
        ↓
Uploaded to Azure App Service
        ↓
Azure runs: node index.mjs
        ↓
Express server starts on PORT 3000
  ├── /api/* → route handlers (your logic)
  └── everything else → serves React frontend files
        ↓
User's browser loads index.html
React app boots, talks back to /api/*
```

---

## What Does Vite Do?

Vite is the **build tool and development server** for the frontend code. It has two distinct jobs depending on the context.

### During Development (local coding)

Vite runs a fast local server (`pnpm run dev`) so you can see your app in a browser while you work. Its key feature is **instant hot reload** — when you save a file, the browser updates in under a second without a full page refresh. You don't have to manually rebuild anything while coding.

### During Build (for deployment)

Browsers cannot run TypeScript or JSX (React's HTML-in-JS syntax). Vite transforms your source code into something browsers understand:

1. **Compiles TypeScript** — strips all the type annotations, converting `.tsx`/`.ts` files to plain `.js`
2. **Compiles JSX** — converts React's HTML-like syntax (e.g. `<Button>Click me</Button>`) into regular JavaScript function calls
3. **Bundles** — takes hundreds of separate files (your code + React + all the UI component libraries) and combines them into a small number of optimized `.js` files
4. **Tree-shakes** — removes code you imported but never actually used, keeping bundle size small
5. **Processes CSS** — compiles Tailwind's utility classes and removes any you didn't use
6. **Fingerprints assets** — renames output files with a hash (e.g. `index-a3f9c2.js`) so browsers know when to re-download a file vs. use a cached copy

The output lands in `artifacts/reading-buddy/dist/public/` — a folder of plain static files (HTML, CSS, JS) that any browser can load directly.

### The Short Version

Vite bridges the gap between the modern developer-friendly code you write (TypeScript, JSX, Tailwind) and the plain HTML/CSS/JavaScript that browsers can actually run.
