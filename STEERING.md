# Steering Document: AI Workflow Builder Template

## 1. Project Overview

**Name:** AI Workflow Builder Template
**License:** Apache 2.0
**Version:** 0.1.0

This is a full-stack, AI-driven visual workflow automation platform built as a deployable Vercel template. Users design workflows on a drag-and-drop canvas, configure real third-party integrations, execute runs on the server, generate TypeScript code from visual workflows, and monitor execution history with per-node logs.

The product ships as a one-click Vercel deployment (`vercel.new/workflow-builder`) with automatic Neon Postgres provisioning.

---

## 2. Tech Stack

| Layer | Technology | Version / Notes |
|---|---|---|
| Framework | Next.js (App Router) | 16.x, React 19 |
| Language | TypeScript | Strict mode, path alias `@/*` |
| Package Manager | pnpm | Enforced; never npm/yarn |
| Workflow Engine | Workflow DevKit (`workflow`) | `"use workflow"` directive, `withWorkflow` Next plugin |
| Canvas | React Flow (`@xyflow/react`) | 12.x, node-based visual editor |
| State Management | Jotai | Atomic state for workflow data, UI, execution |
| Styling | Tailwind CSS 4 + shadcn/ui (New York) | `tw-animate-css`, dark mode via `next-themes` |
| Database | PostgreSQL (Neon in prod) | Drizzle ORM for type-safe access |
| Authentication | Better Auth | Email/password, GitHub, Google, Vercel OAuth, anonymous users |
| AI | OpenAI (via Vercel AI SDK) | Workflow generation from natural language |
| Code Editor | Monaco Editor | In-browser TypeScript editing |
| Code Quality | Ultracite (lint + format) | `pnpm check` / `pnpm fix` |
| E2E Testing | Playwright | Tests in `e2e/` |
| Animations | Motion (Framer Motion) | UI transitions |
| Deployment | Vercel | Analytics, Speed Insights, OG image generation |

---

## 3. Architecture

### 3.1 Directory Layout

```
/
  app/               Next.js App Router (pages, layouts, API routes)
    api/             REST API route handlers (no server actions)
    workflows/       Workflow list and editor pages
  components/        React components
    workflow/         Canvas, nodes, toolbar, config panels, runs
    overlays/         Modal/drawer UI for settings, integrations, exports
    ui/              shadcn/ui primitives and custom inputs
    ai-elements/     AI-themed visual components for the canvas
    auth/            Authentication provider and dialog
    settings/        Account and integration management
  lib/               Core logic
    db/              Database connection and Drizzle schema
    auth.ts          Server-side Better Auth configuration
    auth-client.ts   Client-side auth hooks
    api-client.ts    Type-safe fetch wrapper for all API routes
    workflow-store.ts  Jotai atoms for workflow state
    workflow-executor.workflow.ts  Server-side workflow execution engine
    workflow-codegen.ts  Visual workflow to TypeScript code generation
    types/           Shared TypeScript types
    atoms/           Additional Jotai atoms (overlays, UI)
    ai-gateway/      Vercel AI Gateway integration
    integrations/    Integration helper utilities
  plugins/           Auto-discovered integration plugins
    registry.ts      Declarative action metadata, config fields, codegen hooks
    index.ts         Generated barrel file (auto-created by discover-plugins)
    <name>/          One directory per integration (ai-gateway, blob, clerk, etc.)
  drizzle/           Generated SQL migration files
  scripts/           Build and dev scripts
  e2e/               Playwright test specs
  public/            Static SVG assets
```

### 3.2 Data Flow

```
Browser (React Flow + Jotai)
  |
  |-- lib/api-client.ts (type-safe fetch)
  |
  v
App Router API Routes (app/api/...)
  |
  |-- Drizzle ORM
  |
  v
PostgreSQL (Neon)
```

Workflow execution:
1. Client POSTs to `/api/workflow/[workflowId]/execute`
2. Route handler creates a `workflow_executions` row
3. Invokes Workflow DevKit `start(executeWorkflow, [...])` on the server
4. Executor iterates graph nodes, calling plugin steps for each action
5. Per-node results are written to `workflow_execution_logs`
6. Client polls execution status and displays per-node outcomes

### 3.3 State Management

All client-side workflow state lives in **Jotai atoms** defined in `lib/workflow-store.ts`:

- **Workflow data:** `nodesAtom`, `edgesAtom`, `currentWorkflowIdAtom`, `currentWorkflowNameAtom`
- **Selection:** `selectedNodeAtom`, `selectedEdgeAtom`
- **Execution UI:** `isExecutingAtom`, `executionLogsAtom`, `selectedExecutionIdAtom`
- **Edit state:** `hasUnsavedChangesAtom`, undo/redo history via `historyAtom`/`futureAtom`
- **UI:** panel visibility, sidebar state, overlay atoms

Autosave is debounced (1s for typing, immediate for structural changes like add/delete/connect).

### 3.4 Persistent Canvas

The root layout mounts a `<PersistentCanvas />` component that survives route transitions. Page content overlays the canvas with `pointer-events-none` and a `z-10` layer. This allows the workflow graph to remain rendered while navigating between views.

---

## 4. Plugin System

### 4.1 Structure

Each plugin lives in `plugins/<name>/` and exports:
- An `index.ts` with metadata: plugin name, icon, integration type, and an array of **actions** (each action defines config fields, output fields, and an `execute` function or step file reference)
- Step files (`steps/<action>.ts`) containing the actual execution logic
- Optionally, React components for custom UI

### 4.2 Registry

`plugins/registry.ts` defines the shared types: `ActionConfigField`, `ActionOutputField`, `PluginDefinition`, and exports a `PLUGIN_REGISTRY` map plus helper lookup functions. Actions declare:
- **Config fields:** Typed inputs (template-input, template-textarea, text, number, select, schema-builder) with conditional rendering (`showWhen`)
- **Output fields:** Named fields returned on success
- **Codegen hooks:** Custom code generation for the TypeScript export

### 4.3 Discovery

Running `pnpm discover-plugins` (executed automatically on `dev` and `build`) scans the `plugins/` directory, generates `plugins/index.ts` (barrel imports), and creates step importer mappings used by the workflow executor at runtime.

### 4.4 Available Plugins (14)

AI Gateway, Blob, Clerk, fal.ai, Firecrawl, GitHub, Linear, Perplexity, Resend, Slack, Stripe, Superagent, v0, Webflow

### 4.5 Plugin Rules

- Steps must use native `fetch` -- no SDK client libraries as dependencies
- Standardized output: `{ success: true, data: {...} }` or `{ success: false, error: { message: "..." } }`
- Output fields reference keys without `data.` prefix

---

## 5. API Surface

All backend communication uses REST API routes -- **no Next.js server actions**. The client consumes routes through `lib/api-client.ts` which exposes typed namespaces.

### 5.1 Route Map

| Namespace | Route | Methods | Purpose |
|---|---|---|---|
| Auth | `/api/auth/[...all]` | * | Better Auth catch-all handler |
| Workflows | `/api/workflows` | GET | List user workflows |
| | `/api/workflows/create` | POST | Create new workflow |
| | `/api/workflows/current` | GET | Get current/latest workflow |
| | `/api/workflows/[workflowId]` | GET, PUT, DELETE | CRUD single workflow |
| | `/api/workflows/[workflowId]/code` | GET | Generate TypeScript code |
| | `/api/workflows/[workflowId]/duplicate` | POST | Duplicate workflow |
| | `/api/workflows/[workflowId]/download` | GET | Download workflow as file |
| | `/api/workflows/[workflowId]/webhook` | POST | Trigger via webhook |
| | `/api/workflows/[workflowId]/executions` | GET | Execution history |
| Execution | `/api/workflow/[workflowId]/execute` | POST | Run a workflow |
| | `/api/workflows/executions/[executionId]/logs` | GET | Execution logs |
| | `/api/workflows/executions/[executionId]/status` | GET | Execution status |
| AI | `/api/ai/generate` | POST | Generate workflow from prompt |
| Integrations | `/api/integrations` | GET, POST | List / create integrations |
| | `/api/integrations/test` | POST | Test an integration config |
| | `/api/integrations/[integrationId]` | PUT, DELETE | Update / delete integration |
| | `/api/integrations/[integrationId]/test` | POST | Test existing integration |
| API Keys | `/api/api-keys` | GET, POST | Manage webhook API keys |
| | `/api/api-keys/[keyId]` | DELETE | Delete API key |
| User | `/api/user` | GET, PUT | Get / update user profile |
| AI Gateway | `/api/ai-gateway/consent` | GET, POST | AI Gateway consent flow |
| | `/api/ai-gateway/status` | GET | AI Gateway status |
| | `/api/ai-gateway/teams` | GET | AI Gateway teams |

### 5.2 Client API Usage

```typescript
import { api } from "@/lib/api-client";

await api.workflow.create({ name, nodes, edges });
await api.workflow.update(workflowId, { nodes, edges });
await api.ai.generate(prompt);
await api.integration.list();
```

---

## 6. Database Schema

PostgreSQL via Drizzle ORM. Schema defined in `lib/db/schema.ts`.

| Table | Purpose | Key Fields |
|---|---|---|
| `users` | User accounts (Better Auth) | id, name, email, isAnonymous |
| `sessions` | Active sessions | id, token, userId, expiresAt |
| `accounts` | OAuth / credential accounts | userId, providerId, accessToken |
| `verifications` | Email verification tokens | identifier, value, expiresAt |
| `workflows` | Workflow definitions | id, name, userId, nodes (JSONB), edges (JSONB), visibility |
| `integrations` | User credential storage | userId, type, config (JSONB), isManaged |
| `workflow_executions` | Execution run records | workflowId, userId, status, input, output, duration |
| `workflow_execution_logs` | Per-node execution logs | executionId, nodeId, status, input, output, duration |
| `api_keys` | Webhook authentication keys | userId, keyHash, keyPrefix |

Migration workflow:
1. Edit `lib/db/schema.ts`
2. Run `pnpm db:generate` (auto-generates SQL)
3. Run `pnpm db:push` (applies to database)

Never write migration SQL manually.

---

## 7. Authentication

Better Auth handles all authentication with the following strategies:

- **Anonymous users:** Auto-created on first visit, can be upgraded when linking a real account
- **Email/password:** Standard credential flow
- **GitHub OAuth:** Enabled when `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` are set
- **Google OAuth:** Enabled when `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` are set
- **Vercel OAuth:** Enabled when `VERCEL_CLIENT_ID` is set (used for AI Gateway integration)

Server config lives in `lib/auth.ts`; client hooks in `lib/auth-client.ts`. The root layout wraps the app in `<AuthProvider>`.

---

## 8. Workflow Execution Engine

The executor lives in `lib/workflow-executor.workflow.ts` and uses the `"use workflow"` directive from Workflow DevKit.

Execution flow:
1. Resolves the workflow graph into an ordered list of steps
2. For each node, determines the step type (trigger, HTTP request, database query, condition, or plugin action)
3. Resolves template variables (`{{NodeLabel.field}}`) from previous step outputs
4. Calls the appropriate step function (built-in or dynamically imported from plugins via `getStepImporter`)
5. Records per-step results to `workflow_execution_logs`
6. Returns aggregated execution result

---

## 9. Code Generation

The system can convert visual workflows into standalone TypeScript files using the `"use workflow"` directive.

Key files:
- `lib/workflow-codegen.ts` -- Main codegen orchestrator
- `lib/workflow-codegen-sdk.ts` -- SDK-based code generation
- `lib/workflow-codegen-shared.ts` -- Shared utilities
- `lib/codegen-templates/` -- Template files for generated code

Plugins can provide custom codegen hooks via the registry to control how their actions appear in generated code.

---

## 10. UI Component Architecture

### 10.1 Component Categories

- **Workflow components** (`components/workflow/`): Canvas, nodes (trigger, action, add), toolbar, config panels, runs viewer, context menu
- **Overlay system** (`components/overlays/`): Modals and drawers for settings, integrations, API keys, export, public sharing, confirmations
- **UI primitives** (`components/ui/`): shadcn/ui components plus custom template-aware inputs (`template-badge-input`, `template-badge-textarea`)
- **AI elements** (`components/ai-elements/`): Visual flourishes for the canvas (shimmer effects, custom edges, prompts)

### 10.2 Node Types

- **Trigger node:** Entry point (webhook, schedule, manual, database event). Cannot be deleted.
- **Action node:** Configurable step (plugin actions, HTTP request, condition, database query)
- **Add node:** Placeholder "+" button for appending new steps

### 10.3 Overlay System

The app uses a centralized overlay provider (`components/overlays/overlay-provider.tsx`) managed through Jotai atoms. Overlays include: settings, integrations management, API keys, export workflow, make public, AI Gateway consent, workflow issues, alerts, confirmations.

---

## 11. Build, Deploy, and Scripts

### 11.1 NPM Scripts

| Script | Description |
|---|---|
| `pnpm dev` | Discover plugins, then start Next.js dev server |
| `pnpm build` | Run prod migrations, discover plugins, then Next.js build |
| `pnpm start` | Start production server |
| `pnpm type-check` | TypeScript compiler check (`tsc --noEmit`) |
| `pnpm check` | Lint check via Ultracite |
| `pnpm fix` | Auto-fix lint and format via Ultracite |
| `pnpm db:generate` | Generate Drizzle migration SQL from schema changes |
| `pnpm db:migrate` | Run pending migrations |
| `pnpm db:push` | Push schema directly to database |
| `pnpm db:studio` | Open Drizzle Studio GUI |
| `pnpm discover-plugins` | Scan and register plugins (auto-generates index) |
| `pnpm create-plugin` | Scaffold a new plugin interactively |
| `pnpm test:e2e` | Run Playwright E2E tests |
| `pnpm test:e2e:ui` | Run Playwright tests with UI mode |

### 11.2 Deployment

- **Vercel:** One-click deploy via `vercel.new/workflow-builder`
- **Neon Postgres:** Auto-provisioned via `vercel-template.json` integration
- **Production migrations:** `scripts/migrate-prod.ts` runs `pnpm db:migrate` only in Vercel production environments
- **Next config:** Wrapped with `withWorkflow` from Workflow DevKit

### 11.3 Environment Variables

| Variable | Required | Purpose |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `BETTER_AUTH_SECRET` | Yes | Auth encryption secret |
| `BETTER_AUTH_URL` | Yes | Base URL for auth callbacks |
| `AI_GATEWAY_API_KEY` | Yes | OpenAI API key for AI generation |
| `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` | No | Enable GitHub OAuth |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | No | Enable Google OAuth |
| `VERCEL_CLIENT_ID` | No | Enable Vercel OAuth / AI Gateway |

---

## 12. Testing

- **E2E only:** Playwright tests in `e2e/workflow.spec.ts`
- **Test suite:** "Workflow Editor" -- verifies canvas visibility, drag-to-create, action search, HTTP request selection, search filtering
- **Web server:** Tests start `pnpm dev` on `localhost:3000` (reuses existing unless in CI)
- **No unit test framework** currently configured (no Jest/Vitest)

---

## 13. Coding Conventions

These are enforced rules from `AGENTS.md`:

1. **Package manager:** Always pnpm
2. **No server actions:** Use API routes exclusively; consume via `api` from `@/lib/api-client`
3. **shadcn/ui components:** Use existing primitives; add new ones with `pnpm dlx shadcn@latest add <component>`
4. **No native dialogs:** Use shadcn AlertDialog, Dialog, or Sonner toasts
5. **No barrel files:** Do not create re-export index files
6. **Jotai hooks:** Use the correct hook (`useAtom`, `useAtomValue`, `useSetAtom`) based on read/write needs
7. **Plugin fetch-only:** No SDK dependencies in plugin steps; use native `fetch`
8. **Standardized step output:** `{ success, data }` or `{ success, error }` format
9. **Database migrations:** Always auto-generated via `pnpm db:generate`; never write SQL manually
10. **Remove unused code:** No underscore-prefixed unused variables; clean imports
11. **No emojis** in code or documentation
12. **Code quality gates:** Run `pnpm type-check` and `pnpm fix` before completing work

---

## 14. Key Risks and Considerations

- **JSONB columns:** Workflow `nodes` and `edges` are stored as untyped JSONB. Validation happens at the application level, not the database level.
- **Anonymous users:** The system creates anonymous accounts automatically. Orphaned anonymous data may accumulate without a cleanup strategy.
- **Plugin security surface:** While SDKs are avoided in plugins for supply chain safety, the plugins still make authenticated `fetch` calls to third-party APIs using user-stored credentials.
- **No unit tests:** Test coverage is limited to E2E. Core logic (executor, codegen, state management) lacks unit test protection.
- **README accuracy:** Some endpoint paths in `README.md` differ from actual implementation (e.g., `generate-workflow` vs `generate`, `generate-code` vs `code`). Use the implemented routes as source of truth.
- **Autosave race conditions:** Rapid edits with debounced saves could theoretically lose intermediate state if the tab closes during the delay window.

---

## 15. Getting Started (Developer Quick Reference)

```bash
# Clone and install
pnpm install

# Set up environment
cp .env.example .env.local  # then fill in DATABASE_URL, BETTER_AUTH_SECRET, etc.

# Initialize database
pnpm db:push

# Start development
pnpm dev

# Before committing
pnpm type-check && pnpm fix
```

Open `http://localhost:3000` to access the workflow builder.
