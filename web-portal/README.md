# CanPlan 2.0 — Supporter Web Portal

A web portal for **support people** and **organization admins** using CanPlan 2.0,
the cloud-based expansion of the CanPlan app. From the portal, supporters can see
the people they support, monitor task progress, and respond to help requests.

> ⚠️ **Initial setup — mocked data.** This repository is the *first* version of the
> web portal. It is intentionally limited to a start page and project scaffolding.
> **All data is currently mocked.** There is no real backend, authentication, or AWS
> connection yet. The fake API layer is designed to be swapped for real AWS AppSync
> GraphQL calls later with minimal changes to component code.

## Tech stack

- **React** + **TypeScript**
- **Vite** (dev server and build tooling)
- **React Router** for client-side routing
- **CSS Modules** + a small global stylesheet (design tokens)
- A **fake GraphQL API service** (`src/api/fakeGraphqlClient.ts`) backed by in-memory mock data
- Build output compatible with **AWS Amplify Hosting** CI/CD

## Project structure

```
.
├── amplify.yml              # AWS Amplify Hosting build config
├── index.html               # Vite HTML entry point
├── public/                  # Static assets (favicon, etc.)
├── src/
│   ├── api/                 # Data-access layer (fake GraphQL client for now)
│   │   └── fakeGraphqlClient.ts
│   ├── components/          # Reusable UI components (Header, cards, activity feed)
│   ├── data/                # In-memory mock data (NOT imported by components)
│   │   └── mockData.ts
│   ├── pages/               # Route-level pages (StartPage / dashboard)
│   ├── styles/              # Global styles and design tokens
│   ├── types/               # Shared TypeScript domain types
│   ├── App.tsx              # Route table + layout shell
│   └── main.tsx             # App bootstrap (Router + render)
└── README.md
```

### How data flows

Components **never** import mock data directly. They call async functions from
`src/api/fakeGraphqlClient.ts`, which simulate network latency and return data
shaped like the future AppSync GraphQL responses:

- `getDashboardSummary()` — counts for the dashboard cards
- `getAssignedUsers()` — users assigned to the current supporter
- `getRecentActivity(limit?)` — recent progress/activity events
- `getTasks()` — tasks for assigned users

When the real backend is ready, reimplement these functions using the Amplify
GraphQL client (e.g. `generateClient().graphql(...)`). As long as the function
signatures and return types stay the same, the UI does not need to change.

## Local setup

Requires **Node.js 18+** and npm.

```bash
# 1. Install dependencies
npm install

# 2. Start the local dev server (http://localhost:5173)
npm run dev

# 3. Type-check + build for production (outputs to ./dist)
npm run build

# 4. Preview the production build locally
npm run preview
```

| Script            | Description                                   |
| ----------------- | --------------------------------------------- |
| `npm run dev`     | Start the Vite dev server with hot reload     |
| `npm run build`   | Type-check (`tsc -b`) and build to `dist/`    |
| `npm run preview` | Serve the production build locally            |
| `npm run lint`    | Type-check only (`tsc --noEmit`)              |

## AWS Amplify Hosting deployment

This project is ready to connect to **AWS Amplify Hosting** with GitHub-based CI/CD.

1. Push this repository to GitHub.
2. In the AWS Amplify console, choose **Host web app** → **GitHub**, and authorize access.
3. Select this repository and the `main` branch.
4. Amplify auto-detects the build settings from [`amplify.yml`](./amplify.yml).
   No changes are needed — it runs `npm ci`, then `npm run build`, and publishes `dist/`.
5. Deploy. Every push to the connected branch triggers a new build and deploy.

### Build configuration (`amplify.yml`)

```yaml
version: 1
frontend:
  phases:
    preBuild:
      commands:
        - npm ci
    build:
      commands:
        - npm run build
  artifacts:
    baseDirectory: dist
    files:
      - '**/*'
  cache:
    paths:
      - node_modules/**/*
```

### SPA routing note

Because the portal uses client-side routing (React Router), add a **rewrite rule**
in the Amplify console so deep links resolve to `index.html`:

- **Source address:** `/<*>`
- **Target address:** `/index.html`
- **Type:** `200 (Rewrite)`

This is configured in **App settings → Rewrites and redirects** in the Amplify console.

## Security notes

- **No real authentication yet.** Amazon Cognito will be added in a later phase.
- **No AWS credentials or secrets are committed.** Do not hardcode secrets in this repo.
  Environment-specific values should come from Amplify environment variables.
- `.env*` files and generated AWS config files are git-ignored.

## Roadmap (future phases)

- Replace the fake API layer with AWS AppSync GraphQL (DynamoDB-backed).
- Add Amazon Cognito authentication and role-based access (supporter vs. admin).
- Add S3 for media/attachments and CloudWatch for monitoring.
- Expand pages: user detail, task management, help-request handling.
