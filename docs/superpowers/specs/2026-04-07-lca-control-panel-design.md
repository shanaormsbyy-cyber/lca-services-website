# LCA Control Panel — Design Spec
**Date:** 2026-04-07
**Owner:** Shana (LCA Cleaning Services)
**Status:** Approved

---

## Overview

A private internal control panel for LCA Cleaning Services — a short-term rental cleaning business based in Hamilton/Waikato, NZ with operations in Wanaka. Replaces ad-hoc spreadsheets and messaging with a single source of truth for tasks, staff, and properties, plus an embedded AI assistant with live business context.

**Repo location:** `C:\Users\dener\lca-control-panel` (separate from the marketing site)
**Deployment:** Vercel
**Access:** Private — authenticated users only (Shana + small team)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 14 (App Router) |
| Database | Supabase (Postgres) |
| Auth | Clerk |
| Styling | Tailwind CSS |
| Drag-and-drop | @hello-pangea/dnd |
| AI | Anthropic Claude API (`claude-sonnet-4-5`) via `@anthropic-ai/sdk` |
| Markdown | react-markdown |
| Deployment | Vercel |

---

## Architecture

### Project Structure

```
/app
  /dashboard          → stats overview (default landing)
  /tasks              → kanban board
  /staff              → staff & schedules grid
  /properties         → property list
  /ai                 → AI assistant with persistent history
  /api
    /tasks            → CRUD endpoints
    /staff            → CRUD endpoints
    /properties       → CRUD endpoints
    /ai/chat          → streaming AI endpoint
  /sign-in            → Clerk sign-in (only public route)

/components
  /tasks
    KanbanBoard.tsx
    TaskCard.tsx
    AddTaskModal.tsx
    TaskDetailPanel.tsx   (slide-out)
  /staff
    StaffGrid.tsx
    StaffCard.tsx
    StaffModal.tsx
  /properties
    PropertyList.tsx
  /ai
    ChatWindow.tsx
    MessageBubble.tsx
  /layout
    Sidebar.tsx
    Header.tsx

/lib
  supabase.ts         → Supabase client (server-side)
  anthropic.ts        → Anthropic client + system prompt builder
  types.ts            → shared TypeScript types

/docs/superpowers/specs/
```

### Auth

Clerk middleware wraps all routes. Unauthenticated requests redirect to `/sign-in`. No multi-tenancy — single org.

### Data Flow

- Frontend calls Next.js API routes (`/api/*`)
- API routes use the Supabase service role key (server-side only — never exposed to browser)
- AI chat endpoint fetches live task/staff/property context from Supabase, builds system prompt, streams response via Anthropic SDK
- AI chat history stored in `ai_chat_history` table and loaded on page open

---

## Database Schema

### tasks
```sql
id          uuid primary key default gen_random_uuid()
title       text not null
status      text not null default 'todo'    -- todo | inprogress | done
priority    text not null default 'medium'  -- low | medium | high
assignee    text
due_date    date
notes       text
property    text
created_at  timestamptz default now()
updated_at  timestamptz default now()
```

### staff
```sql
id          uuid primary key default gen_random_uuid()
name        text not null
role        text not null
status      text default 'active'           -- active | leave | inactive
phone       text
email       text
shifts      text[]
notes       text
created_at  timestamptz default now()
```

### properties
```sql
id          uuid primary key default gen_random_uuid()
name        text not null
client      text
city        text                             -- Hamilton | Wanaka
address     text
bedrooms    int
notes       text
active      boolean default true
created_at  timestamptz default now()
```

### ai_chat_history
```sql
id          uuid primary key default gen_random_uuid()
role        text not null                   -- user | assistant
content     text not null
created_at  timestamptz default now()
```

> Extra fields can be added to any table via the Supabase dashboard at any time.

---

## Seed Data

### Staff
| Name | Role | Status | Shifts |
|---|---|---|---|
| Mia T. | Senior Cleaner | active | Mon, Tue, Wed, Thu |
| Sam K. | Cleaner | active | Mon, Wed, Fri |
| Jordan H. | Cleaner | active | Tue, Thu, Sat |
| Priya M. | Cleaner | leave | — |

### Properties
| Name | Client | City | Bedrooms |
|---|---|---|---|
| 42 River Rd | KOSH Properties | Hamilton | 3 |
| Wanaka Apt | KOSH Properties | Wanaka | 2 |
| 12 Lake View | KOSH Properties | Hamilton | 4 |
| 8 Elm St | KOSH Properties | Hamilton | 3 |

### Tasks
| Title | Priority | Assignee | Due |
|---|---|---|---|
| KOSH — 42 River Rd post-stay clean | high | Mia T. | Apr 8 |
| KOSH — Wanaka apt deep clean | medium | Sam K. | Apr 9 |
| New staff induction — Jordan | medium | Shana | Apr 10 |
| QC inspection — 12 Lake View | high | Mia T. | Apr 8 |
| Restock supplies — Hamilton depot | low | Sam K. | Apr 12 |

---

## Pages & Features

### Dashboard (`/dashboard`)
- 4 stat cards: Open Tasks · In Progress · Active Staff · Properties Assigned
- Today's urgent tasks (high priority, due today or overdue)
- Recent activity feed (last 10 task updates)
- Quick-add task button

### Tasks (`/tasks`) — Kanban Board
- Three columns: To Do · In Progress · Done
- Drag-and-drop between columns (`@hello-pangea/dnd`)
- Task cards show: title, priority badge, assignee, due date, property tag
- Click card → slide-out detail panel (edit all fields, add notes)
- Filter bar: by assignee, by priority, by property
- "Add task" button → modal with all fields

### Staff (`/staff`)
- 2-column grid of staff cards
- Each card: initials avatar, name, role, status badge, shift pills, property count
- Click card → edit modal (name, role, phone, email, shifts, status, notes)
- "Add staff member" button → modal

### Properties (`/properties`)
- Table view: name, client, city, bedrooms, active status, notes
- Filter by city (Hamilton / Wanaka) and client
- Inline row editing

### AI Assistant (`/ai`)
- Full persistent chat thread (loaded from `ai_chat_history` on page open)
- Streaming responses via Anthropic API
- Quick-action chips: "What's urgent today?" · "Who's available this weekend?" · "Summarise team workload" · "Draft a message to a client"
- Markdown rendering in AI responses
- Clear history button
- System prompt dynamically injects: open tasks, active staff, active properties, current date

#### AI System Prompt Template
```
You are an AI assistant for LCA Cleaning Services, a short-term rental cleaning business
in New Zealand owned by Shana. You have access to the current state of the business.

CURRENT TASKS (open): [inject open tasks]
STAFF: [inject staff with status and shifts]
PROPERTIES: [inject active properties]
TODAY: [inject current date]

Be concise, direct, and practical. You help Shana run her business — scheduling gaps,
task prioritisation, staff allocation, client communication, business decisions.
Keep responses under 150 words unless a detailed breakdown is needed.
```

---

## API Routes

| Route | Methods | Description |
|---|---|---|
| `/api/tasks` | GET, POST | List all tasks / create task |
| `/api/tasks/[id]` | PATCH, DELETE | Update / delete task |
| `/api/staff` | GET, POST | List all staff / create staff member |
| `/api/staff/[id]` | PATCH, DELETE | Update / delete staff member |
| `/api/properties` | GET, POST | List all properties / create property |
| `/api/properties/[id]` | PATCH, DELETE | Update / delete property |
| `/api/ai/chat` | POST | Streaming AI response |

All routes are server-side only. Supabase service role key never reaches the browser.

---

## Styling & Branding

### Colour Palette
| Token | Value | Usage |
|---|---|---|
| `lca-bg` | `#0d0d0d` | Main background |
| `lca-sidebar` | `#0a0a0a` | Sidebar background |
| `lca-card` | `#1a1a1a` | Card / panel backgrounds |
| `lca-teal` | `#00bfa5` | Accent, buttons, active states |
| `lca-border` | `rgba(255,255,255,0.07)` | Borders |
| Text primary | `#e8e8e8` | Main text |
| Text secondary | `#888` | Supporting text |
| Text muted | `#555` | Placeholders, disabled |

### Sidebar Logo
"**LCA**" bold white · "cleaning" small gray · "services" teal

### Badges
| Type | Values | Colours |
|---|---|---|
| Priority | low / medium / high | gray / yellow / red |
| Staff status | active / on leave / inactive | teal / amber / gray |
| Task status | todo / inprogress / done | gray / blue / teal |

### Typography
- Font: Inter (Tailwind default `font-sans`)
- Desktop-first layout; basic responsiveness but not phone-optimised for v1

---

## Environment Variables

```env
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
ANTHROPIC_API_KEY=
```

---

## Third-Party Setup Checklist

### 1. Supabase (free tier)
1. Go to [supabase.com](https://supabase.com) → Create account → New project
2. Name it `lca-control-panel`, choose a strong password, select region closest to NZ (Sydney)
3. Go to **Settings → API** → copy `Project URL` and `anon public` key → paste into `.env.local`
4. Also copy `service_role` key → paste into `.env.local` as `SUPABASE_SERVICE_ROLE_KEY`
5. Go to **SQL Editor** → run the migration SQL (provided in `/supabase/migrations/`)

### 2. Clerk (free tier)
1. Go to [clerk.com](https://clerk.com) → Create account → New application
2. Name it `LCA Control Panel`, enable **Email** sign-in
3. Go to **API Keys** → copy Publishable Key and Secret Key → paste into `.env.local`
4. Go to **Users** → Invite Shana's email address

### 3. Anthropic API
1. Go to [console.anthropic.com](https://console.anthropic.com) → API Keys → Create key
2. Paste into `.env.local` as `ANTHROPIC_API_KEY`

### 4. Vercel Deployment
1. Push repo to GitHub
2. Go to [vercel.com](https://vercel.com) → New Project → Import from GitHub
3. Add all env vars under **Settings → Environment Variables**
4. Deploy — Vercel auto-deploys on every push to `main`

---

## Build Order

1. Scaffold Next.js 14 + Tailwind + Clerk auth
2. Connect Supabase + run migrations + seed data
3. Build shared layout (Sidebar + Header)
4. Build Tasks page (list first, then drag-and-drop)
5. Build Staff page
6. Build Properties page
7. Build Dashboard with live stats
8. Build AI chat endpoint + streaming UI
9. Wire up all CRUD operations
10. Deploy to Vercel

---

## Out of Scope (v1)

- Mobile optimisation
- Payments / external integrations
- Multi-tenancy
- Real-time collaborative editing
- Hostaway / Stripe integration
