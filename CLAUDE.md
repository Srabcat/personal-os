# Personal OS — Project Brief & Conventions

> **What this file is.** This is the project's `CLAUDE.md`. Claude Code reads it automatically at the start of every session. It captures the decisions made before the codebase existed, the current state, the conventions to follow, and the immediate work plan. Keep it updated as the project evolves — when a decision changes, change it here.

---

## 1. What this is

A personal life OS — a single self-hosted web app that combines four modules into one place:

1. **People (Personal CRM).** Track friends, family, colleagues. Last-contact, life events, gift ideas, family relationships, freeform notes.
2. **Memory (File / Knowledge Vault).** Drop random files, links, and notes; find them later via tags + semantic search.
3. **Journal (Daily Notes).** Daily entries, mood, reflections, optional prompts. Linked to People when relevant.
4. **Todos (Eisenhower Prioritization).** Tasks with priority, due dates, and a 2x2 urgent/important grid view.

Single user (me) for v1. Multi-user-capable for future sharing (partner, coach).

## 2. Why this codebase

This repo is a fork of [pdovhomilja/nextcrm-app](https://github.com/pdovhomilja/nextcrm-app) — chosen after a three-way evaluation against [twentyhq/twenty](https://github.com/twentyhq/twenty) (CRM, Jotai+Linaria, paradigm we'd have to learn) and [marmelab/atomic-crm](https://github.com/marmelab/atomic-crm) (Vite + react-admin, would lock us into react-admin's resource model).

NextCRM won because (a) the stack is exactly what we want — Next.js 16, React 19, Prisma 7, shadcn 3.8, Tailwind 4, better-auth — (b) the `Documents` model with embeddings + chunks is essentially the memory module already built, and (c) iterating with Claude Code is fastest in vanilla Next.js + shadcn (no domain-specific framework like react-admin in the way).

We are **not** keeping NextCRM's product shape. It ships as a sales-CRM; we strip the sales-CRM out and reshape what remains into a personal OS. See Phase 1 below for the pruning plan.

## 3. Confirmed stack

These versions are what NextCRM ships with at the fork point. Pin them on first install; upgrade deliberately, not opportunistically.

- **Framework:** Next.js 16.2.x, React 19.2.x, TypeScript (strict)
- **UI:** shadcn 3.8.x (Radix + Tailwind 4.2.x), TipTap editor
- **DB:** PostgreSQL via Prisma 7.6 (with `@prisma/adapter-pg`)
- **Auth:** better-auth 1.5.x (NOT NextAuth/Auth.js — note this; the README in upstream is wrong)
- **Background jobs:** Inngest 4
- **Email:** Resend (transactional) + react-email (templates). IMAP sync via `imap` + `mailparser` is in upstream — we will likely replace this with Gmail OAuth later (see § 9).
- **Storage:** S3-compatible (AWS S3 or MinIO)
- **AI:** OpenAI + `@openai/agents` + Vercel `ai` SDK + `@modelcontextprotocol/sdk` — we are swapping to Anthropic Claude (see § 7).
- **Forms / validation:** react-hook-form + Zod 4
- **State:** jotai (already in deps; use it sparingly)
- **Tests:** Jest (unit) + Playwright (E2E)
- **Runtime:** Node 22+, pnpm 9+

## 4. Decisions on record (revisit if pain)

| Decision | Choice | Rationale |
|---|---|---|
| v1 scope | All four modules, sequenced by friction (People → Memory → Journal → Todos) | NextCRM gives us People + Memory cheaply; Journal + Todos need a small build. Better to exercise the full system early than ship a half-product. |
| Auth model | Keep better-auth multi-user as-is | Zero work today, future-proofs sharing. Single-user practically just means "I'm the only Sales/User row." |
| AI features | Keep embeddings, swap OpenAI → Anthropic for completions, defer agents + MCP server to v2 | Vector search across notes is the unlock; agents/MCP are over-scoped for v1. |
| Database | Keep Postgres + Prisma | NextCRM is wired to it; swapping to Supabase is a weekend of work for marginal benefit. |
| Hosting | TBD — local Docker for dev; production target Vercel + Neon (or self-hosted VPS) later | Decide before v1 ship. |
| Mobile / offline | Mobile-responsive in v1; PWA + offline in v2 | "Focus on feature and usability first" — user explicit ask. |
| Gmail / Slack | Defer integrations to v2 — add Gmail OAuth (Twenty pattern) and Slack webhook (via Inngest) post-v1 | Both are easy to add later; not on the critical path. |
| Browser extension (Streak-style Gmail sidebar) | v3 ambition — separate Manifest V3 project that calls our API | Highest leverage UX win, but separate scope. |

If any of these starts costing you more than expected, change the row and note when/why.

## 5. Day 0: setup

```bash
# 1. Fork upstream on GitHub first, then:
git clone git@github.com:<your-github-username>/personal-os.git
cd personal-os
git remote add upstream https://github.com/pdovhomilja/nextcrm-app.git
git fetch upstream

# 2. Pin to the commit you forked from (record it)
git rev-parse HEAD > .upstream-fork-point

# 3. Install
pnpm install

# 4. Env
cp .env.example .env
# fill in: DATABASE_URL, RESEND_API_KEY, AUTH_SECRET (generate via openssl rand -hex 32),
# ANTHROPIC_API_KEY (we're using Claude, not OpenAI — see § 7),
# S3 creds (or MinIO for local). Leave OPENAI_API_KEY for now if any old code paths still use it.

# 5. DB
# either use the docker-compose if it exists, or:
docker run -d --name personal-os-pg -p 5432:5432 -e POSTGRES_PASSWORD=dev postgres:16
pnpm prisma migrate dev
pnpm prisma db seed   # only if a seed exists; otherwise skip

# 6. Run
pnpm dev
```

If `pnpm install` complains about peer deps (Next 16 + React 19 are recent), use `pnpm install --strict-peer-dependencies=false` once and document it here.

## 6. Phase 1 — Brutal pruning pass

NextCRM ships a sales CRM. About half of it is dead weight for a personal OS. Delete aggressively now while the codebase is still upstream-shaped; pruning later is harder.

### Prisma models to delete (`prisma/schema.prisma`)

```
crm_Leads, crm_Lead_Sources, crm_Lead_Statuses, crm_Lead_Types
crm_Opportunities, crm_OpportunityLineItems
crm_Contracts, crm_ContractLineItems
crm_Targets, crm_TargetLists, crm_Target_Contact, CampaignToTargetLists
crm_campaigns, crm_campaign_steps, crm_campaign_templates, crm_campaign_sends
crm_Products, crm_ProductCategories, crm_AccountProducts
crm_Industry_Type, crm_Contact_Types, crm_Contact_Enrichment
crm_Embeddings_Leads, crm_Embeddings_Opportunities, crm_Embeddings_Accounts
Invoices, Invoice_LineItems, Invoice_Payments, Invoice_Attachments,
  Invoice_Activity, Invoice_TaxRates, Invoice_Series, Invoice_Settings
Currency, ExchangeRate
TodoList   # stub, replaced by Tasks
```

Keep these (we'll repurpose):

```
crm_Contacts          → rename to Person
crm_Accounts          → keep for now; may rename to Organization or delete entirely later
crm_Activities, crm_ActivityLinks   → keep, this is timeline
Documents, Documents_Types, crm_Document_Chunks, crm_Embeddings_Documents
Tasks, tasksComments, Boards, Sections
Email, EmailAccount, EmailEmbedding   → keep for v2 Gmail sync
Users, Session, Account, Verification, ApiKeys, ApiToken
crm_AuditLog, crm_SystemSettings
ImageUpload
```

### Routes to delete (`app/[locale]/(routes)/`)

```
crm/leads/
crm/opportunities/
crm/contracts/
crm/targets/
crm/campaigns/
crm/products/
invoices/
reports/
```

### Action folders to delete (`actions/`)

```
actions/crm/leads/
actions/crm/opportunities/
actions/crm/contracts/
actions/campaigns/
actions/invoices/
actions/reports/
```

### Inngest functions to delete (`inngest/functions/`)

```
campaigns/        # all 4 functions
invoices/         # if any
reports/          # scheduled report sends
emails/sync-account.ts   # IMAP sync — defer; we'll add Gmail OAuth instead in v2
```

### Packages to remove from `package.json`

```
primereact, @primer/* (legacy UI lib coexisting with shadcn)
mongodb (MongoDB driver — leftover from a Postgres migration)
moment, dayjs (keep date-fns only — pick one date lib)
bcryptjs (keep bcrypt only)
@openai/agents, openai   # if removing agents — see § 7
```

After pruning, run `pnpm install`, fix imports, run `pnpm build`. Expect 30-60 errors; resolve them mechanically. Do NOT skip this step — leftover imports will rot the codebase.

### Convention rename pass

In Prisma schema and throughout the app, rename:

```
crm_Contacts  → Person
crm_Accounts  → keep or delete; if keep, rename → Organization
crm_Activities → Activity
crm_ActivityLinks → ActivityLink
```

Keep model names PascalCase singular per Prisma convention. Do this before adding new models so the new schema reads coherently.

## 7. Phase 2 — Schema additions

After pruning. New Prisma models, in addition to the renamed Person and Document.

### People-side additions

```prisma
// Family graph — self-join on Person.
model FamilyRelation {
  id            String   @id @default(cuid())
  fromPersonId  String
  toPersonId    String
  type          FamilyRelationType  // PARENT | CHILD | SIBLING | SPOUSE | PARTNER | OTHER
  notes         String?
  createdAt     DateTime @default(now())

  fromPerson    Person   @relation("from", fields: [fromPersonId], references: [id])
  toPerson      Person   @relation("to",   fields: [toPersonId],   references: [id])

  @@unique([fromPersonId, toPersonId, type])
  @@index([fromPersonId])
  @@index([toPersonId])
}

// Life events — birthdays already on Person, this is for everything else.
model LifeEvent {
  id          String   @id @default(cuid())
  personId    String
  date        DateTime
  isRecurring Boolean  @default(false)   // true for birthday/anniversary
  type        String   // free-form: "birthday", "anniversary", "moved", "married", "kid born", etc.
  title       String
  notes       String?
  createdAt   DateTime @default(now())

  person      Person   @relation(fields: [personId], references: [id])

  @@index([personId])
  @@index([date])
}

// Gift ideas — per person.
model GiftIdea {
  id          String   @id @default(cuid())
  personId    String
  title       String
  notes       String?
  link        String?
  givenAt     DateTime?  // null = idea, set = already given
  createdAt   DateTime @default(now())

  person      Person   @relation(fields: [personId], references: [id])

  @@index([personId])
}
```

### Tags — replace `tags String[]` with a real Tags table (Atomic CRM pattern)

```prisma
model Tag {
  id          String   @id @default(cuid())
  name        String   @unique
  color       String?  // hex
  description String?
  createdAt   DateTime @default(now())

  people      PersonTag[]
  documents   DocumentTag[]
  journals    JournalEntryTag[]
  todos       TaskTag[]
}

model PersonTag {
  personId String
  tagId    String
  person   Person @relation(fields: [personId], references: [id])
  tag      Tag    @relation(fields: [tagId],    references: [id])
  @@id([personId, tagId])
}
// Repeat for DocumentTag, JournalEntryTag, TaskTag.
```

### Person — multi-email/phone JSONB (Atomic CRM pattern)

Drop the `email` + `personal_email` two-column hack. Replace with:

```prisma
model Person {
  // ... existing fields ...
  emails  Json?  // [{type: "personal" | "work" | "other", value: string, primary: boolean}]
  phones  Json?  // [{type: "mobile" | "home" | "work" | "other", value: string, primary: boolean}]
  // ... existing fields ...
}
```

Validate the shape with Zod at the application boundary. The flexibility is worth the schema looseness.

### Journal

```prisma
model JournalEntry {
  id           String   @id @default(cuid())
  date         DateTime @db.Date  // one canonical date — use this for daily-view
  title        String?
  content      Json     // TipTap doc
  contentText  String   @db.Text  // denormalized plain text for full-text + embeddings
  mood         Int?     // optional 1-5 scale
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  ownerId      String

  owner        User     @relation(fields: [ownerId], references: [id])
  people       JournalEntryPerson[]   // who's mentioned in this entry
  tags         JournalEntryTag[]
  attachments  Json?    // [{name, url, mime, size}]

  @@index([ownerId, date])
}

model JournalEntryPerson {
  journalEntryId String
  personId       String
  journalEntry   JournalEntry @relation(fields: [journalEntryId], references: [id])
  person         Person       @relation(fields: [personId],       references: [id])
  @@id([journalEntryId, personId])
}
```

### Todos — Eisenhower

NextCRM's `Tasks` already has `priority String`, `dueDateAt DateTime?`, and `Sections` (Kanban columns). Add:

```prisma
// Add to Tasks model:
quadrant   EisenhowerQuadrant?   // DO | DECIDE | DELEGATE | DELETE  -- urgent x important matrix
isUrgent   Boolean  @default(false)
isImportant Boolean @default(false)
```

The 2x2 view derives quadrant from `isUrgent` + `isImportant`. We store both for flexibility (e.g., a task can be neither urgent nor important = DELETE quadrant).

### Activity log as a database view (Atomic CRM pattern)

After all models exist, add a Postgres view that unions journal entries + completed tasks + new contacts + new gift ideas:

```sql
-- prisma/migrations/<timestamp>_activity_log_view/migration.sql
CREATE OR REPLACE VIEW activity_log AS
  SELECT 'journal'::text AS kind, id, "ownerId" AS owner_id, date AS occurred_at, COALESCE(title, contentText) AS summary
  FROM "JournalEntry"
  UNION ALL
  SELECT 'task_done'::text, id, "ownerId", "doneAt", title
  FROM "Tasks" WHERE "doneAt" IS NOT NULL
  UNION ALL
  SELECT 'person_added'::text, id, "ownerId", "createdAt", CONCAT(first_name, ' ', last_name)
  FROM "Person"
  -- add more sources as the system grows
  ORDER BY occurred_at DESC;
```

Expose in Prisma via raw query or `previewFeatures = ["views"]` (see Prisma docs).

## 8. Phase 3 — UI customization

### Routes to add (under `app/[locale]/(routes)/`)

```
people/                  # rename from crm/contacts/
  [id]/                  # person detail with tabs: profile, life events, gifts, journal mentions, activity
journal/
  [date]/                # daily entry: today's view, prev/next nav, calendar picker
  search/
memory/                  # rename/repurpose from documents/
  [id]/
  search/
todos/
  matrix/                # Eisenhower 2x2 grid view
  inbox/                 # unsorted incoming todos
dashboard/               # home: today's journal prompt, top todos, "you haven't talked to X in N days"
```

### Components to build (in addition to repurposing existing)

- **Cmd-K command palette** (Twenty pattern) — global, fuzzy-search across people / notes / journal / todos. Use `cmdk` package + shadcn Command primitive.
- **Right-side detail panel** (Twenty pattern) — clicking a row in any list slides in a detail panel without page nav. Use shadcn Sheet.
- **View-as-table-or-kanban-or-calendar** (Twenty pattern) — start with table + kanban for todos. Calendar comes later.
- **Daily-journal calendar widget** — heatmap of which days have entries.
- **Person card** — last-contact, upcoming life events, recent gift ideas, quick-add note.
- **Activity feed** — uses the `activity_log` view, paginated.

### Things NextCRM has that we're keeping

- TipTap editor — already wired, use for journal content and freeform notes.
- Inngest — already wired, use for: daily journal prompt reminders, "you haven't talked to X in 6 months" alerts, embedding regeneration on save.
- Better-auth — already wired with email-OTP. Don't touch unless we need a new auth method.
- Resend + react-email — for transactional email (auth OTPs, alerts).

## 9. AI plan (v1)

Keep it minimal. Two AI features in v1:

1. **Semantic search across memory and journal.** Reuse NextCRM's embedding pipeline (`crm_Document_Chunks` + `crm_Embeddings_Documents`). Generalize the chunking + embedding code to also embed `JournalEntry.contentText`. Single search endpoint that queries both.
2. **Person summary on demand.** A button on the Person detail page that calls Claude with all the person's notes, life events, recent journal mentions, and recent activities, returning a 200-word summary. Cache the result.

Implementation:

- Replace OpenAI calls with Anthropic Claude. The Vercel `ai` SDK supports both — swap the provider import:
  ```ts
  import { anthropic } from '@ai-sdk/anthropic';
  const model = anthropic('claude-sonnet-4-6');
  ```
- Keep embeddings on OpenAI for now (their text embeddings are cheaper and well-supported). Or switch to Voyage if you prefer a cleaner Anthropic-only stack — flag this decision.
- Rip out `@openai/agents` and the MCP server scaffolding — they're over-scoped for v1.

## 10. Patterns inherited from Twenty (read-only references)

These were not forkable (Twenty's stack is incompatible) but the *concepts* are worth borrowing as we build:

- **Object/Field metadata system** — Twenty makes Person, Note, Task, etc. all rows in an `ObjectMetadata` table with typed `FieldMetadata` rows. We are NOT replicating this in v1 (over-engineered for our needs), but if/when we want user-defined custom fields, this is the pattern to revisit. See `packages/twenty-server/src/engine/metadata-modules/` in upstream Twenty.
- **Workflow engine shape** — triggers (DB-event, cron, manual, webhook) × actions (create record, send email, AI, HTTP, delay, branch). Inngest covers v1 needs; revisit if we want a UI workflow builder.
- **Right-side detail panel + inline-editable cells** — copy the UX, build with shadcn primitives.
- **Polymorphic `*Target` pattern** (TaskTarget, NoteTarget) — a single Note/Task can attach to multiple object types. Use this if our notes start needing to attach to journal entries AND people AND todos simultaneously.

## 11. Patterns inherited from Atomic CRM (read-only references)

Same logic — Atomic CRM stack is incompatible (react-admin paradigm), but concepts are gold:

- **Multi-email/phone JSONB** — already baked into our schema above.
- **Separate Tags table** — already baked in.
- **Notes with `attachments jsonb[]`** — already baked into Journal schema.
- **Activity log as a database view** — already baked in.
- **Mobile component split** — pair every component with `<Component>.tsx` + `<Component>Mobile.tsx` rather than `useIsMobile` checks scattered through one file. Adopt when we touch mobile in v2.
- **CSV-import wizard** — defer to v2; pattern is "Imports table tracking job status + UI for column mapping."
- **Storybook stories alongside each component** — not in v1, but adopt when we hit our first visual regression. Cheap to add early.

## 12. Working conventions for Claude Code

When working in this codebase, prefer these patterns:

- **TypeScript strict.** Fix the legacy `target: "es5"` in `tsconfig.json` to `es2022` first thing — that's a leftover from the upstream config.
- **Server actions over API routes.** NextCRM mixes both styles in the same domain (we saw `actions/crm/contacts/...` and `app/api/crm/contacts/[id]/route.ts`). For new code, default to server actions; only use API routes when we genuinely need a public HTTP endpoint (e.g., webhooks).
- **Single date library: `date-fns`.** Remove uses of `moment` and `dayjs` as you encounter them.
- **One bcrypt: `bcrypt`.** Remove `bcryptjs` everywhere.
- **No primereact components.** All new UI uses shadcn. Migrate primereact usage as you touch the surrounding code.
- **Prisma migrations, not `db push`.** Every schema change generates a migration. Commit migrations with the same PR as the code change.
- **Conventional commits.** `feat:`, `fix:`, `chore:`, `refactor:`, `db:` for migrations.
- **One PR = one logical change.** Pruning, schema additions, UI changes — separate commits or PRs.
- **Test before claiming done.** `pnpm test` for unit, `pnpm test:e2e` for Playwright. New features need at least one test of the happy path.
- **Don't introduce new top-level dependencies casually.** If you reach for a new package, justify it in the commit message — and prefer extending what's already in `package.json`.
- **Don't mention this file in commits or code comments.** Just follow it.

## 13. Open decisions / TODOs

- [ ] Pick database hosting for production: Neon vs Supabase Postgres vs self-hosted.
- [ ] Pick app hosting for production: Vercel vs self-hosted VPS (Coolify, Dokploy).
- [ ] Decide on embeddings provider: OpenAI text-embedding-3-small vs Voyage AI vs Cohere.
- [ ] Decide whether to keep `crm_Accounts` (rename to Organization) or delete entirely.
- [ ] Decide v2 priorities: Gmail OAuth sync, Slack notifications, Chrome extension, mobile PWA, family-graph visualization. Pick the top 2 to scope after v1 ships.
- [ ] Set up CI (GitHub Actions): lint + typecheck + Jest unit + Prisma migrate check on PR.
- [ ] Set up automated dependency updates (Dependabot or Renovate).

## 14. Source references

- Forked from: [pdovhomilja/nextcrm-app](https://github.com/pdovhomilja/nextcrm-app)
- Pattern references (read-only):
  - [twentyhq/twenty](https://github.com/twentyhq/twenty) — `packages/twenty-server/src/engine/metadata-modules/`, `packages/twenty-front/src/modules/{command-menu,object-record,side-panel}`
  - [marmelab/atomic-crm](https://github.com/marmelab/atomic-crm) — `supabase/migrations/`, `src/components/atomic-crm/{contacts,notes,tasks}`
- This brief was generated from a research session that:
  1. Surveyed open-source personal-OS / personal-CRM projects.
  2. Compared Twenty, NextCRM, Atomic CRM, Monica, AppFlowy, AFFiNE on stack, adoption, maintenance, code quality.
  3. Cloned NextCRM, Atomic CRM, and Twenty for direct code inspection.
  4. Selected NextCRM as the fork target.

---

## How to use this file with Claude Code

This file lives at the repo root. Claude Code reads it automatically each session — no need to paste it in prompts.

When asking Claude Code to do work, you can reference sections by number ("§ 6 pruning pass," "§ 7.3 Tags model") and Claude will look it up. When a decision changes, ask Claude Code to update the row in § 4 with the new choice and a note on why.

Work order suggestion for the first week:
1. Day 1: Run § 5 setup. Get `pnpm dev` working. Commit pinned versions.
2. Day 2-3: Execute § 6 pruning pass. Single PR. `pnpm build` clean.
3. Day 4: Execute § 6 rename pass (`crm_Contacts` → `Person` etc.). Single migration.
4. Day 5-7: § 7 schema additions (FamilyRelation, LifeEvent, GiftIdea, Tag*, JournalEntry, Eisenhower fields). Single migration.
5. Week 2+: § 8 UI work, module by module
