# CanPlan 2.0

An accessible task-planning system for people with cognitive disabilities and the support people who help them.

CanPlan breaks everyday activities — cooking a meal, doing laundry, getting ready for an appointment — into guided, step-by-step tasks. A **primary user** works through those steps on a phone at their own pace, with media, audio, and a reduced-complexity interface. A **support person** authors the tasks, schedules them, and reviews progress on the primary user's behalf.

Built as a Northeastern **CS7980 capstone** project with client **CanAssist / CanPlan**. This repository is a consolidated snapshot of what were four separate project repositories.

---

## Contents

| Folder | What it is | Stack |
|---|---|---|
| [`mobile/`](mobile) | Mobile app — primary user and caregiver experience | Expo · React Native · TypeScript · TanStack Query · Amplify |
| [`backend/`](backend) | Serverless GraphQL API and infrastructure-as-code | AWS CDK · AppSync · Lambda · DynamoDB · Cognito · S3 · Bedrock |
| [`web-portal/`](web-portal) | Supporter / admin web client | React · Vite · TypeScript · React Router |
| [`design/`](design) | Design system, screen designs, accessibility guidelines | Figma Make · shadcn/ui · Tailwind |

## Roles

Authorization is built around four Cognito groups, and the domain model follows them closely:

| Role | Can do |
|---|---|
| **PrimaryUser** | Work through assigned tasks step by step; manage their own tasks and categories |
| **SupportPerson** | Select a primary user they're linked to and act on their behalf — author tasks, schedule them, generate progress reports |
| **OrganizationAdmin** | Manage members within a care organization |
| **SystemAdmin** | Full administrative surface — users, organizations, roles, deletions |

The link between a support person and a primary user is a first-class `SupportLink` record. Most task and category operations take an optional `userId`, authorized against that link, so a single code path serves both self-service and delegated use.

## Features

**For primary users**
- Step-by-step task execution with per-step timers, media, and completion tracking
- Calendar view of scheduled task occurrences
- **Simple Mode** — a reduced-complexity interface for users who find the standard view overwhelming
- Accessibility settings: audio/speech, interface density, notifications
- Categories with custom color palettes for organizing tasks

**For support people**
- "Managing {name}" delegated mode after selecting a linked primary user
- Author and edit task templates on a primary user's behalf
- Schedule recurring assignments into a primary user's calendar
- AI-drafted progress reports with a generate → preview → save flow, so nothing is persisted without review
- Private PDF/JSON report downloads via short-lived presigned URLs

**AI assistance**
- Task-step generation grounded in a Bedrock Knowledge Base, with per-step citations
- `createAiTask` turns a free-text request into a title plus ordered steps as a preview that persists nothing until accepted

## Architecture

```
┌──────────────┐     ┌──────────────┐
│  mobile/     │     │ web-portal/  │
│ primary user │     │  supporter   │
│  + caregiver │     │   + admin    │
└──────┬───────┘     └──────┬───────┘
       │  Cognito ID token  │
       └─────────┬──────────┘
                 ▼
        ┌─────────────────┐
        │ AppSync GraphQL │
        └────────┬────────┘
                 ▼
   ┌─────────────────────────────┐
   │ Domain Lambdas              │
   │ users · categories · tasks  │
   │ assignments · media         │
   │ reports · admin · ai        │
   └──────┬───────────────┬──────┘
          ▼               ▼
   ┌────────────┐  ┌──────────────────┐
   │ DynamoDB   │  │ Bedrock Knowledge│
   │ single     │  │ Base (RAG)       │
   │ table      │  │ + S3 Vectors     │
   └────────────┘  └──────────────────┘
```

Clients never reach a database directly. Every request is a Cognito-authenticated GraphQL call to AppSync, which routes to a domain Lambda that owns its slice of the model.

## Design decisions

**Single-table DynamoDB.** Every entity — profiles, support links, organizations, categories, tasks, steps, assignments, instances, media, reports — lives in one table (`CanPlanTasks-<env>`) on a composite `PK`/`SK`. Key conventions are centralized in [`backend/src/shared/keys.ts`](backend/src/shared/keys.ts).

**Templates and execution are separate.** A `Task` is a reusable template with ordered steps and its own lifecycle status. What a person actually does is a `TaskAssignment` (the schedule rule) producing `TaskInstance` rows (the occurrences). Completion lives on instance steps, never on the template — so editing a template doesn't rewrite history.

**Lazily-materialized instances.** Recurring schedules are stored as rules rather than pre-expanded rows. Concrete instances are created on read, which keeps the calendar cheap no matter how far out a schedule runs.

**Grounded AI with an explicit fallback contract.** Step generation retrieves from the Knowledge Base first. A `groundingMode` input decides whether a query with no relevant corpus passage fails outright with `NOT_FOUND` — without ever calling the generation model — or falls back to ungrounded generation. Every response reports which path produced it via a `source` field, and grounded steps carry citations. Callers can tell generated content from retrieved content.

**Domain Lambdas, not per-field functions.** Each Lambda backs many GraphQL fields and routes internally on `event.info.fieldName`, which keeps cold starts and deploy surface manageable.

**Defense in depth on authorization.** Group restrictions are declared as AppSync auth directives *and* re-checked inside the Lambda, so a schema change can't silently widen access.

**One-directional data flow on the client.** Screens never call the network. They use feature hooks (TanStack Query) → a feature API facade → a shared typed GraphQL client, so caching, retries, and the data source can change without touching rendering.

## Getting started

Each project has its own README with full setup instructions:

- [`backend/README.md`](backend/README.md) — CDK deploy, GraphQL schema, complete API reference
- [`mobile/README.md`](mobile/README.md) — Expo dev server; iOS builds need Xcode 16.1+
- [`web-portal/README.md`](web-portal/README.md) — Vite dev server

Quick start for the mobile app:

```bash
cd mobile
npm install
cp .env.example .env.local   # point EXPO_PUBLIC_GRAPHQL_URL at your AppSync endpoint
npm start
```

All three read configuration from environment files. Copy the `.env.example` in each folder and fill in values from your own CDK deploy outputs. No credentials are committed to this repository, and the AWS environment this was developed against is not publicly reachable.
