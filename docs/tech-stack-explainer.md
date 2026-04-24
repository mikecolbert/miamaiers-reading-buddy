# Tech Stack Explainer

---

## Languages

**TypeScript**
JavaScript with a type system bolted on. Instead of just writing `let name = "Mike"`, you can declare `let name: string = "Mike"` and the compiler will catch mistakes — like trying to pass a number where a name is expected — before the code ever runs. All the source files in this project are TypeScript (`.ts` and `.tsx`). Neither browsers nor Node.js can run TypeScript directly, so it always gets compiled to plain JavaScript first.

**JSX / TSX**
A special syntax used in React files that lets you write HTML-like tags directly inside JavaScript/TypeScript code. For example: `<Button onClick={save}>Save</Button>`. A browser has no idea what that means, so Vite compiles it to plain function calls before deployment.

---

## Frontend

**React**
A JavaScript library for building user interfaces. Instead of manually manipulating the webpage (finding elements, changing text, updating styles), you describe what the page *should look like* given the current data, and React figures out the minimal changes needed to get there. The entire `artifacts/reading-buddy/` folder is a React app.

**Vite**
The build tool and local development server for the React app. During development it gives you instant hot reload (save a file → browser updates in under a second). For deployment it compiles and bundles all the TypeScript, JSX, and CSS into plain files browsers can load. Output goes to `artifacts/reading-buddy/dist/public/`.

**Tailwind CSS**
A CSS framework where you style things by applying small utility classes directly in your HTML/JSX rather than writing separate CSS files. `className="text-lg font-bold text-blue-600"` — each class does one thing. Tailwind removes any classes you didn't use at build time, keeping the CSS file tiny.

**shadcn/ui + Radix UI**
A component library. Instead of building every button, dropdown, dialog, and date picker from scratch, you get pre-built, accessible components. Radix UI provides the behavior (keyboard navigation, focus management, accessibility), and shadcn/ui layers the visual styling on top using Tailwind.

**TanStack Query**
Manages the data-fetching lifecycle in React — loading states, caching responses, retrying on failure, keeping data fresh. When your app asks the backend for a list of books, TanStack Query handles the "show a spinner, fetch the data, cache it, show the result" cycle automatically.

**Wouter**
A lightweight client-side router. When a user clicks a link to `/my-books`, Wouter shows the My Books page without doing a full browser page reload. The URL changes, but only the relevant React components re-render.

**Zod**
A schema validation library. You define the shape of your data (e.g. "a book has a title that's a string and a page count that's a positive number") and Zod validates that real data matches that shape. Used on both the frontend (form validation) and backend (request validation).

**React Hook Form**
Manages HTML form state — tracking which fields are filled, when to show errors, handling submission. Works with Zod to validate form input.

**Framer Motion**
Handles animations — fade-ins, slide transitions, etc. — with a simple declarative API.

**Recharts**
A charting library for React. Used for displaying reading statistics as graphs.

---

## Backend

**Node.js**
A JavaScript runtime that lets JavaScript code run on a server (outside the browser). The Express server runs on Node.js. This project uses Node.js version 22.

**Express**
A minimal web framework for Node.js. It handles incoming HTTP requests and routes them to the right code. In this app, it serves API routes under `/api/*` and, in production, serves the React frontend's static files for all other requests.

**esbuild**
The build tool for the Express backend — the equivalent of Vite but for server-side code. It compiles TypeScript and bundles everything into a single file (`dist/index.mjs`) that Node.js can run directly.

**Pino**
A structured logging library. Instead of `console.log("request received")`, it outputs machine-readable JSON: `{"level":"info","method":"GET","url":"/api/books","statusCode":200}`. This makes logs searchable and useful in production monitoring tools.

---

## Database

**PostgreSQL**
A relational database — data stored in tables with rows and columns, connected by relationships. This is where the app's persistent data lives (books, reading sessions, goals, settings).

**Drizzle ORM**
A TypeScript library that lets you query PostgreSQL using TypeScript code instead of raw SQL strings. It also handles the schema — you define your tables in TypeScript, and Drizzle keeps the database in sync. "ORM" stands for Object-Relational Mapper.

**Drizzle Zod**
A bridge between Drizzle and Zod — it automatically generates Zod validators from your Drizzle table definitions, so your database schema and your validation logic stay in sync.

---

## Shared Libraries (the `lib/` folder)

**`lib/api-spec`** — The API contract, written as an OpenAPI YAML spec. Defines every endpoint, what it accepts, and what it returns. Acts as the single source of truth shared between frontend and backend.

**`lib/api-zod`** — Zod validators auto-generated from the API spec. Used by the backend to validate incoming requests.

**`lib/api-client-react`** — React hooks auto-generated from the API spec. The frontend uses these to make API calls (e.g. `useGetBooks()`) without having to hand-write fetch calls.

**`lib/db`** — The database schema and connection setup, shared wherever database access is needed.

---

## Monorepo & Package Management

**pnpm**
The package manager — like npm but faster and more disk-efficient. It manages downloading and linking all the third-party libraries the project depends on.

**pnpm workspaces**
A feature that lets multiple related packages (the `artifacts/` and `lib/` folders) live in one Git repository and share dependencies. This is the "mono" in monorepo — one repo, many packages.

---

## CI/CD & Hosting

**GitHub Actions**
Automated pipelines that run in the cloud whenever you push code. This project's pipeline installs dependencies, compiles everything, and ships the result to Azure — all without you doing anything manually after `git push`.

**Azure App Service**
Microsoft's cloud platform for hosting web apps. A single Node.js process runs there, serving both the API and the React frontend. Azure handles the server infrastructure (keeping it running, scaling, TLS certificates, etc.) so you don't have to manage a server yourself.

---

## pnpm Workspaces Deep Dive

### What Is a Workspace?

Normally, a project is one `package.json` with one set of code. A **workspace** is a way of saying: "this repository actually contains several distinct packages, and I want them to share dependencies and reference each other directly."

pnpm reads `pnpm-workspace.yaml` to know which folders are packages:

```yaml
packages:
  - artifacts/*
  - lib/*
  - scripts
```

This tells pnpm: every subfolder inside `artifacts/` and `lib/` is its own package. Each one has its own `package.json`, its own `src/` folder, and its own purpose.

### Why Split Into Multiple Packages At All?

Imagine you wrote the same TypeScript type for a "Book" object in three places — the frontend, the backend, and the API validator. Now you change the Book shape. You have to find and update all three. They drift out of sync. Bugs appear.

Workspaces solve this by letting you put shared code in one place and have everything else **import it like a library**. When one package references another:

```json
"dependencies": {
  "@workspace/db": "workspace:*"
}
```

The `workspace:*` means "use the local version of this package from this repo, not something downloaded from the internet." Changes to `lib/db` are immediately visible to `artifacts/api-server` without publishing anything.

---

## The `lib/` Folder — Shared Building Blocks

These are **libraries with no UI and no server** — pure logic and types that other packages consume.

```
lib/
├── api-spec/          ← the API contract (OpenAPI YAML)
├── api-zod/           ← validators generated from the spec
├── api-client-react/  ← React hooks generated from the spec
└── db/                ← database schema + connection
```

**`lib/api-spec`** — The single source of truth for the API. One YAML file defines every endpoint: what URL it lives at, what data it accepts, what it returns. Both the frontend and backend are generated from this file, so they can never disagree about the shape of the data.

**`lib/api-zod`** — Auto-generated Zod validators from the spec. The backend imports these to check that incoming requests have the right shape before touching the database.

**`lib/api-client-react`** — Auto-generated React hooks from the spec. The frontend imports these to call the API. Instead of writing a raw `fetch("/api/books")` call by hand, a component just calls `useGetBooks()` and gets back typed data.

**`lib/db`** — The database layer. Defines the PostgreSQL table schemas using Drizzle ORM, and exports the database connection. Any code that needs to read or write the database imports from here.

The key idea: `lib/` packages are consumed but never deployed on their own. They get bundled into the artifacts that actually run.

---

## The `artifacts/` Folder — The Things That Actually Run

These are the **deployable outputs** — the two runnable pieces of the application.

```
artifacts/
├── api-server/        ← the Express backend server
└── reading-buddy/     ← the React frontend app
```

**`artifacts/api-server`** — The Node.js/Express web server. It imports from `lib/db` to talk to the database, imports from `lib/api-zod` to validate requests, and defines the actual route handlers (what happens when someone calls `GET /api/books`). At build time, esbuild compiles it into a single `dist/index.mjs` file. This is what Azure runs.

**`artifacts/reading-buddy`** — The React app the user sees in their browser. It imports from `lib/api-client-react` to call the backend, and from `lib/db` types for shared data shapes. At build time, Vite compiles it into static HTML/CSS/JS files in `dist/public/`. These files are served by the Express server in production.

There's also a third artifact:

**`artifacts/mockup-sandbox`** — A separate Vite/React app used only during design and development. It's a playground for building and previewing UI components in isolation, without needing the backend to be running. It is **not** deployed to Azure.

---

## How They All Connect

```
lib/api-spec  ─────────────────────────────────────────┐
     │                                                   │
     ├─→ lib/api-zod  ──────→  artifacts/api-server     │
     │                              │                    │
     └─→ lib/api-client-react ──→  artifacts/reading-buddy
                                        │
lib/db  ──────────────────────→  artifacts/api-server
```

- The **API spec** is the contract. Everything else is derived from it.
- The **backend** uses `lib/db` to talk to the database, and `lib/api-zod` to validate what it receives.
- The **frontend** uses `lib/api-client-react` to talk to the backend.
- The **database library** is only used by the backend — the frontend never touches the database directly.

### The Practical Benefit

When you change something — say you add a `pageCount` field to a book — you change it in one place (`lib/api-spec`), regenerate the validators and client hooks, and TypeScript immediately tells you everywhere in the frontend and backend that needs updating. The monorepo structure makes the entire codebase act as one coherent system instead of three separate projects that have to stay manually synchronized.
