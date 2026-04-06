# LCA Control Panel Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a private internal control panel for LCA Cleaning Services at `C:\Users\dener\lca-control-panel` with task management, staff scheduling, property tracking, and a streaming AI assistant.

**Architecture:** Next.js 14 App Router with Supabase for data, Clerk for auth, and Anthropic for AI. Frontend calls Next.js API routes exclusively — Supabase service role key never reaches the browser. AI responses stream via Anthropic SDK with live business context injected into each system prompt.

**Tech Stack:** Next.js 14, Supabase, Clerk, Tailwind CSS, @hello-pangea/dnd, @anthropic-ai/sdk, react-markdown, TypeScript

---

## File Map

```
lca-control-panel/
├── app/
│   ├── layout.tsx                        # Root layout with ClerkProvider
│   ├── page.tsx                          # Redirect to /dashboard
│   ├── sign-in/[[...sign-in]]/page.tsx   # Clerk sign-in page
│   ├── dashboard/page.tsx                # Stats + activity + urgent tasks
│   ├── tasks/page.tsx                    # Kanban board page
│   ├── staff/page.tsx                    # Staff grid page
│   ├── properties/page.tsx               # Properties table page
│   ├── ai/page.tsx                       # AI chat page
│   └── api/
│       ├── tasks/route.ts                # GET, POST /api/tasks
│       ├── tasks/[id]/route.ts           # PATCH, DELETE /api/tasks/[id]
│       ├── staff/route.ts                # GET, POST /api/staff
│       ├── staff/[id]/route.ts           # PATCH, DELETE /api/staff/[id]
│       ├── properties/route.ts           # GET, POST /api/properties
│       ├── properties/[id]/route.ts      # PATCH, DELETE /api/properties/[id]
│       └── ai/chat/route.ts              # POST /api/ai/chat (streaming)
├── components/
│   ├── layout/
│   │   ├── Sidebar.tsx                   # Nav sidebar with LCA logo
│   │   └── AppLayout.tsx                 # Sidebar + main content wrapper
│   ├── tasks/
│   │   ├── KanbanBoard.tsx               # DnD board with 3 columns
│   │   ├── TaskCard.tsx                  # Individual task card
│   │   ├── AddTaskModal.tsx              # Create task modal
│   │   └── TaskDetailPanel.tsx           # Slide-out edit panel
│   ├── staff/
│   │   ├── StaffGrid.tsx                 # 2-col grid of staff cards
│   │   ├── StaffCard.tsx                 # Individual staff card
│   │   └── StaffModal.tsx                # Create/edit staff modal
│   ├── properties/
│   │   └── PropertyList.tsx              # Table with inline editing
│   ├── ai/
│   │   ├── ChatWindow.tsx                # Chat thread + input + chips
│   │   └── MessageBubble.tsx             # Single message with markdown
│   └── ui/
│       ├── PriorityBadge.tsx             # low/medium/high badge
│       └── StatusBadge.tsx               # active/leave/inactive badge
├── lib/
│   ├── supabase.ts                       # Server-side Supabase client
│   ├── anthropic.ts                      # Anthropic client + buildSystemPrompt()
│   └── types.ts                          # Shared TypeScript types
├── supabase/
│   └── migrations/
│       └── 001_init.sql                  # Schema + seed data
├── middleware.ts                         # Clerk auth middleware
├── tailwind.config.ts                    # LCA colour tokens
├── .env.local                            # Environment variables (gitignored)
└── .env.example                          # Template for env vars
```

---

## Task 1: Scaffold Project

**Files:**
- Create: `lca-control-panel/` (new repo at `C:\Users\dener\lca-control-panel`)
- Create: `tailwind.config.ts`
- Create: `.env.example`
- Create: `.env.local`
- Create: `middleware.ts`
- Create: `app/layout.tsx`
- Create: `app/page.tsx`
- Create: `app/sign-in/[[...sign-in]]/page.tsx`

- [ ] **Step 1: Scaffold Next.js project**

```bash
cd /c/Users/dener
npx create-next-app@14 lca-control-panel \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir=false \
  --import-alias="@/*"
cd lca-control-panel
```

- [ ] **Step 2: Install dependencies**

```bash
npm install @supabase/supabase-js @clerk/nextjs @anthropic-ai/sdk \
  @hello-pangea/dnd react-markdown
```

- [ ] **Step 3: Update tailwind.config.ts with LCA colour tokens**

Replace the content of `tailwind.config.ts` with:

```typescript
import type { Config } from 'tailwindcss'

const config: Config = {
  content: [
    './pages/**/*.{js,ts,jsx,tsx,mdx}',
    './components/**/*.{js,ts,jsx,tsx,mdx}',
    './app/**/*.{js,ts,jsx,tsx,mdx}',
  ],
  theme: {
    extend: {
      colors: {
        lca: {
          bg: '#0d0d0d',
          sidebar: '#0a0a0a',
          card: '#1a1a1a',
          teal: '#00bfa5',
          border: 'rgba(255,255,255,0.07)',
        },
      },
    },
  },
  plugins: [],
}
export default config
```

- [ ] **Step 4: Create .env.example**

```bash
cat > .env.example << 'EOF'
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
ANTHROPIC_API_KEY=
EOF
```

- [ ] **Step 5: Create .env.local with placeholder values**

```bash
cp .env.example .env.local
```

Open `.env.local` and add placeholder strings so the app compiles without real keys:

```env
NEXT_PUBLIC_SUPABASE_URL=https://placeholder.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=placeholder-anon-key
SUPABASE_SERVICE_ROLE_KEY=placeholder-service-role-key
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_placeholder
CLERK_SECRET_KEY=sk_test_placeholder
ANTHROPIC_API_KEY=sk-ant-placeholder
```

- [ ] **Step 6: Create Clerk middleware**

Create `middleware.ts` at project root:

```typescript
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server'

const isPublicRoute = createRouteMatcher(['/sign-in(.*)'])

export default clerkMiddleware((auth, request) => {
  if (!isPublicRoute(request)) {
    auth().protect()
  }
})

export const config = {
  matcher: ['/((?!.*\\..*|_next).*)', '/', '/(api|trpc)(.*)'],
}
```

- [ ] **Step 7: Create root layout with ClerkProvider**

Replace `app/layout.tsx`:

```typescript
import { ClerkProvider } from '@clerk/nextjs'
import type { Metadata } from 'next'
import './globals.css'

export const metadata: Metadata = {
  title: 'LCA Control Panel',
  description: 'Internal operations panel for LCA Cleaning Services',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <ClerkProvider>
      <html lang="en">
        <body className="bg-lca-bg text-[#e8e8e8] font-sans antialiased">
          {children}
        </body>
      </html>
    </ClerkProvider>
  )
}
```

- [ ] **Step 8: Create root page redirect**

Create `app/page.tsx`:

```typescript
import { redirect } from 'next/navigation'

export default function Home() {
  redirect('/dashboard')
}
```

- [ ] **Step 9: Create sign-in page**

Create `app/sign-in/[[...sign-in]]/page.tsx`:

```typescript
import { SignIn } from '@clerk/nextjs'

export default function SignInPage() {
  return (
    <div className="min-h-screen bg-lca-bg flex items-center justify-center">
      <div>
        <div className="text-center mb-8">
          <span className="text-2xl font-bold text-white">LCA</span>
          <span className="text-2xl text-[#888]"> cleaning </span>
          <span className="text-2xl text-lca-teal">services</span>
        </div>
        <SignIn />
      </div>
    </div>
  )
}
```

- [ ] **Step 10: Initialise git and commit**

```bash
git init
git add -A
git commit -m "feat: scaffold Next.js 14 project with Clerk auth and LCA theme"
```

---

## Task 2: Shared Types and Lib Setup

**Files:**
- Create: `lib/types.ts`
- Create: `lib/supabase.ts`
- Create: `lib/anthropic.ts`

- [ ] **Step 1: Create shared TypeScript types**

Create `lib/types.ts`:

```typescript
export type TaskStatus = 'todo' | 'inprogress' | 'done'
export type TaskPriority = 'low' | 'medium' | 'high'
export type StaffStatus = 'active' | 'leave' | 'inactive'

export interface Task {
  id: string
  title: string
  status: TaskStatus
  priority: TaskPriority
  assignee: string | null
  due_date: string | null
  notes: string | null
  property: string | null
  created_at: string
  updated_at: string
}

export interface Staff {
  id: string
  name: string
  role: string
  status: StaffStatus
  phone: string | null
  email: string | null
  shifts: string[]
  notes: string | null
  created_at: string
}

export interface Property {
  id: string
  name: string
  client: string | null
  city: string | null
  address: string | null
  bedrooms: number | null
  notes: string | null
  active: boolean
  created_at: string
}

export interface ChatMessage {
  id: string
  role: 'user' | 'assistant'
  content: string
  created_at: string
}
```

- [ ] **Step 2: Create Supabase server client**

Create `lib/supabase.ts`:

```typescript
import { createClient } from '@supabase/supabase-js'

// Server-side client using service role key — never import in client components
export function createServerSupabaseClient() {
  return createClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!
  )
}
```

- [ ] **Step 3: Create Anthropic client and system prompt builder**

Create `lib/anthropic.ts`:

```typescript
import Anthropic from '@anthropic-ai/sdk'
import type { Task, Staff, Property } from './types'

export const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
})

export function buildSystemPrompt(
  tasks: Task[],
  staff: Staff[],
  properties: Property[]
): string {
  const today = new Date().toLocaleDateString('en-NZ', {
    weekday: 'long',
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  })

  const taskSummary = tasks
    .map(
      (t) =>
        `- ${t.title} [${t.priority}/${t.status}] assignee: ${t.assignee ?? 'unassigned'} due: ${t.due_date ?? 'no date'} property: ${t.property ?? '—'}`
    )
    .join('\n')

  const staffSummary = staff
    .map(
      (s) =>
        `- ${s.name} (${s.role}) status: ${s.status} shifts: ${s.shifts.join(', ') || 'none'}`
    )
    .join('\n')

  const propertySummary = properties
    .map((p) => `- ${p.name} client: ${p.client ?? '—'} city: ${p.city ?? '—'} beds: ${p.bedrooms ?? '?'}`)
    .join('\n')

  return `You are an AI assistant for LCA Cleaning Services, a short-term rental cleaning business in New Zealand owned by Shana. You have access to the current state of the business.

CURRENT TASKS (open):
${taskSummary || 'No open tasks.'}

STAFF:
${staffSummary || 'No staff records.'}

PROPERTIES:
${propertySummary || 'No active properties.'}

TODAY: ${today}

Be concise, direct, and practical. You help Shana run her business — scheduling gaps, task prioritisation, staff allocation, client communication, business decisions. Keep responses under 150 words unless a detailed breakdown is needed.`
}
```

- [ ] **Step 4: Commit**

```bash
git add lib/
git commit -m "feat: add shared types, Supabase client, and Anthropic prompt builder"
```

---

## Task 3: Database Migration and Seed Data

**Files:**
- Create: `supabase/migrations/001_init.sql`

- [ ] **Step 1: Create migration SQL**

Create `supabase/migrations/001_init.sql`:

```sql
-- Enable UUID extension
create extension if not exists "pgcrypto";

-- Tasks table
create table if not exists tasks (
  id          uuid primary key default gen_random_uuid(),
  title       text not null,
  status      text not null default 'todo',
  priority    text not null default 'medium',
  assignee    text,
  due_date    date,
  notes       text,
  property    text,
  created_at  timestamptz default now(),
  updated_at  timestamptz default now()
);

-- Staff table
create table if not exists staff (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  role        text not null,
  status      text default 'active',
  phone       text,
  email       text,
  shifts      text[] default '{}',
  notes       text,
  created_at  timestamptz default now()
);

-- Properties table
create table if not exists properties (
  id          uuid primary key default gen_random_uuid(),
  name        text not null,
  client      text,
  city        text,
  address     text,
  bedrooms    int,
  notes       text,
  active      boolean default true,
  created_at  timestamptz default now()
);

-- AI chat history table
create table if not exists ai_chat_history (
  id          uuid primary key default gen_random_uuid(),
  role        text not null,
  content     text not null,
  created_at  timestamptz default now()
);

-- Seed staff
insert into staff (name, role, status, shifts) values
  ('Mia T.', 'Senior Cleaner', 'active', array['Mon','Tue','Wed','Thu']),
  ('Sam K.', 'Cleaner', 'active', array['Mon','Wed','Fri']),
  ('Jordan H.', 'Cleaner', 'active', array['Tue','Thu','Sat']),
  ('Priya M.', 'Cleaner', 'leave', array[]::text[]);

-- Seed properties
insert into properties (name, client, city, bedrooms) values
  ('42 River Rd', 'KOSH Properties', 'Hamilton', 3),
  ('Wanaka Apt', 'KOSH Properties', 'Wanaka', 2),
  ('12 Lake View', 'KOSH Properties', 'Hamilton', 4),
  ('8 Elm St', 'KOSH Properties', 'Hamilton', 3);

-- Seed tasks
insert into tasks (title, status, priority, assignee, due_date, property) values
  ('KOSH — 42 River Rd post-stay clean', 'todo', 'high', 'Mia T.', '2026-04-08', '42 River Rd'),
  ('KOSH — Wanaka apt deep clean', 'todo', 'medium', 'Sam K.', '2026-04-09', 'Wanaka Apt'),
  ('New staff induction — Jordan', 'todo', 'medium', 'Shana', '2026-04-10', null),
  ('QC inspection — 12 Lake View', 'todo', 'high', 'Mia T.', '2026-04-08', '12 Lake View'),
  ('Restock supplies — Hamilton depot', 'todo', 'low', 'Sam K.', '2026-04-12', null);
```

- [ ] **Step 2: Note how to run migration**

Once you have your Supabase project created:
1. Go to your Supabase dashboard → SQL Editor
2. Paste the full contents of `supabase/migrations/001_init.sql`
3. Click "Run"

The tables will be created and seed data inserted.

- [ ] **Step 3: Commit migration**

```bash
git add supabase/
git commit -m "feat: add database migration and seed data"
```

---

## Task 4: API Routes

**Files:**
- Create: `app/api/tasks/route.ts`
- Create: `app/api/tasks/[id]/route.ts`
- Create: `app/api/staff/route.ts`
- Create: `app/api/staff/[id]/route.ts`
- Create: `app/api/properties/route.ts`
- Create: `app/api/properties/[id]/route.ts`

- [ ] **Step 1: Create tasks collection route**

Create `app/api/tasks/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase'

export async function GET() {
  const supabase = createServerSupabaseClient()
  const { data, error } = await supabase
    .from('tasks')
    .select('*')
    .order('created_at', { ascending: false })
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data)
}

export async function POST(request: Request) {
  const supabase = createServerSupabaseClient()
  const body = await request.json()
  const { data, error } = await supabase.from('tasks').insert(body).select().single()
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data, { status: 201 })
}
```

- [ ] **Step 2: Create tasks item route**

Create `app/api/tasks/[id]/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase'

export async function PATCH(request: Request, { params }: { params: { id: string } }) {
  const supabase = createServerSupabaseClient()
  const body = await request.json()
  const { data, error } = await supabase
    .from('tasks')
    .update({ ...body, updated_at: new Date().toISOString() })
    .eq('id', params.id)
    .select()
    .single()
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data)
}

export async function DELETE(_: Request, { params }: { params: { id: string } }) {
  const supabase = createServerSupabaseClient()
  const { error } = await supabase.from('tasks').delete().eq('id', params.id)
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return new NextResponse(null, { status: 204 })
}
```

- [ ] **Step 3: Create staff collection route**

Create `app/api/staff/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase'

export async function GET() {
  const supabase = createServerSupabaseClient()
  const { data, error } = await supabase
    .from('staff')
    .select('*')
    .order('name', { ascending: true })
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data)
}

export async function POST(request: Request) {
  const supabase = createServerSupabaseClient()
  const body = await request.json()
  const { data, error } = await supabase.from('staff').insert(body).select().single()
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data, { status: 201 })
}
```

- [ ] **Step 4: Create staff item route**

Create `app/api/staff/[id]/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase'

export async function PATCH(request: Request, { params }: { params: { id: string } }) {
  const supabase = createServerSupabaseClient()
  const body = await request.json()
  const { data, error } = await supabase
    .from('staff')
    .update(body)
    .eq('id', params.id)
    .select()
    .single()
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data)
}

export async function DELETE(_: Request, { params }: { params: { id: string } }) {
  const supabase = createServerSupabaseClient()
  const { error } = await supabase.from('staff').delete().eq('id', params.id)
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return new NextResponse(null, { status: 204 })
}
```

- [ ] **Step 5: Create properties collection route**

Create `app/api/properties/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase'

export async function GET() {
  const supabase = createServerSupabaseClient()
  const { data, error } = await supabase
    .from('properties')
    .select('*')
    .order('name', { ascending: true })
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data)
}

export async function POST(request: Request) {
  const supabase = createServerSupabaseClient()
  const body = await request.json()
  const { data, error } = await supabase.from('properties').insert(body).select().single()
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data, { status: 201 })
}
```

- [ ] **Step 6: Create properties item route**

Create `app/api/properties/[id]/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase'

export async function PATCH(request: Request, { params }: { params: { id: string } }) {
  const supabase = createServerSupabaseClient()
  const body = await request.json()
  const { data, error } = await supabase
    .from('properties')
    .update(body)
    .eq('id', params.id)
    .select()
    .single()
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return NextResponse.json(data)
}

export async function DELETE(_: Request, { params }: { params: { id: string } }) {
  const supabase = createServerSupabaseClient()
  const { error } = await supabase.from('properties').delete().eq('id', params.id)
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return new NextResponse(null, { status: 204 })
}
```

- [ ] **Step 7: Commit API routes**

```bash
git add app/api/
git commit -m "feat: add CRUD API routes for tasks, staff, and properties"
```

---

## Task 5: Shared Layout (Sidebar + AppLayout)

**Files:**
- Create: `components/layout/Sidebar.tsx`
- Create: `components/layout/AppLayout.tsx`

- [ ] **Step 1: Create Sidebar component**

Create `components/layout/Sidebar.tsx`:

```typescript
'use client'

import Link from 'next/link'
import { usePathname } from 'next/navigation'
import { UserButton } from '@clerk/nextjs'

const navItems = [
  { href: '/dashboard', label: 'Dashboard', icon: '▤' },
  { href: '/tasks', label: 'Tasks', icon: '✓' },
  { href: '/staff', label: 'Staff', icon: '◉' },
  { href: '/properties', label: 'Properties', icon: '⌂' },
  { href: '/ai', label: 'AI Assistant', icon: '◈' },
]

export default function Sidebar() {
  const pathname = usePathname()

  return (
    <aside className="w-56 min-h-screen bg-lca-sidebar border-r border-[rgba(255,255,255,0.07)] flex flex-col">
      {/* Logo */}
      <div className="px-6 py-6 border-b border-[rgba(255,255,255,0.07)]">
        <div className="leading-tight">
          <span className="text-xl font-bold text-white">LCA</span>
          <br />
          <span className="text-xs text-[#888]">cleaning </span>
          <span className="text-xs text-lca-teal">services</span>
        </div>
      </div>

      {/* Nav */}
      <nav className="flex-1 px-3 py-4 space-y-1">
        {navItems.map((item) => {
          const active = pathname === item.href
          return (
            <Link
              key={item.href}
              href={item.href}
              className={`flex items-center gap-3 px-3 py-2 rounded-md text-sm transition-colors ${
                active
                  ? 'bg-lca-teal/10 text-lca-teal'
                  : 'text-[#888] hover:text-[#e8e8e8] hover:bg-white/5'
              }`}
            >
              <span className="text-base">{item.icon}</span>
              {item.label}
            </Link>
          )
        })}
      </nav>

      {/* User */}
      <div className="px-6 py-4 border-t border-[rgba(255,255,255,0.07)]">
        <UserButton afterSignOutUrl="/sign-in" />
      </div>
    </aside>
  )
}
```

- [ ] **Step 2: Create AppLayout wrapper**

Create `components/layout/AppLayout.tsx`:

```typescript
import Sidebar from './Sidebar'

export default function AppLayout({ children }: { children: React.ReactNode }) {
  return (
    <div className="flex min-h-screen bg-lca-bg">
      <Sidebar />
      <main className="flex-1 overflow-auto">
        {children}
      </main>
    </div>
  )
}
```

- [ ] **Step 3: Commit layout**

```bash
git add components/layout/
git commit -m "feat: add Sidebar and AppLayout components"
```

---

## Task 6: UI Badge Components

**Files:**
- Create: `components/ui/PriorityBadge.tsx`
- Create: `components/ui/StatusBadge.tsx`

- [ ] **Step 1: Create PriorityBadge**

Create `components/ui/PriorityBadge.tsx`:

```typescript
import type { TaskPriority } from '@/lib/types'

const styles: Record<TaskPriority, string> = {
  low: 'bg-[#333] text-[#888]',
  medium: 'bg-yellow-900/40 text-yellow-400',
  high: 'bg-red-900/40 text-red-400',
}

export default function PriorityBadge({ priority }: { priority: TaskPriority }) {
  return (
    <span className={`text-xs px-2 py-0.5 rounded font-medium ${styles[priority]}`}>
      {priority}
    </span>
  )
}
```

- [ ] **Step 2: Create StatusBadge**

Create `components/ui/StatusBadge.tsx`:

```typescript
import type { StaffStatus } from '@/lib/types'

const styles: Record<StaffStatus, string> = {
  active: 'bg-teal-900/40 text-lca-teal',
  leave: 'bg-amber-900/40 text-amber-400',
  inactive: 'bg-[#333] text-[#888]',
}

const labels: Record<StaffStatus, string> = {
  active: 'Active',
  leave: 'On Leave',
  inactive: 'Inactive',
}

export default function StatusBadge({ status }: { status: StaffStatus }) {
  return (
    <span className={`text-xs px-2 py-0.5 rounded font-medium ${styles[status]}`}>
      {labels[status]}
    </span>
  )
}
```

- [ ] **Step 3: Commit badges**

```bash
git add components/ui/
git commit -m "feat: add PriorityBadge and StatusBadge components"
```

---

## Task 7: Tasks Page — Kanban Board

**Files:**
- Create: `components/tasks/TaskCard.tsx`
- Create: `components/tasks/AddTaskModal.tsx`
- Create: `components/tasks/TaskDetailPanel.tsx`
- Create: `components/tasks/KanbanBoard.tsx`
- Create: `app/tasks/page.tsx`

- [ ] **Step 1: Create TaskCard**

Create `components/tasks/TaskCard.tsx`:

```typescript
import type { Task } from '@/lib/types'
import PriorityBadge from '@/components/ui/PriorityBadge'

interface TaskCardProps {
  task: Task
  onClick: (task: Task) => void
}

export default function TaskCard({ task, onClick }: TaskCardProps) {
  const isOverdue =
    task.due_date && new Date(task.due_date) < new Date() && task.status !== 'done'

  return (
    <div
      onClick={() => onClick(task)}
      className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-lg p-3 cursor-pointer hover:border-lca-teal/30 transition-colors"
    >
      <p className="text-sm text-[#e8e8e8] mb-2 leading-snug">{task.title}</p>
      <div className="flex flex-wrap gap-1 mb-2">
        <PriorityBadge priority={task.priority} />
        {task.property && (
          <span className="text-xs px-2 py-0.5 rounded bg-[#222] text-[#888]">
            {task.property}
          </span>
        )}
      </div>
      <div className="flex items-center justify-between text-xs text-[#555]">
        <span>{task.assignee ?? 'Unassigned'}</span>
        {task.due_date && (
          <span className={isOverdue ? 'text-red-400' : ''}>
            {new Date(task.due_date).toLocaleDateString('en-NZ', { day: 'numeric', month: 'short' })}
          </span>
        )}
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create AddTaskModal**

Create `components/tasks/AddTaskModal.tsx`:

```typescript
'use client'

import { useState } from 'react'
import type { Task, TaskPriority, TaskStatus } from '@/lib/types'

interface AddTaskModalProps {
  onClose: () => void
  onAdd: (task: Task) => void
  staff: { name: string }[]
  properties: { name: string }[]
}

export default function AddTaskModal({ onClose, onAdd, staff, properties }: AddTaskModalProps) {
  const [form, setForm] = useState({
    title: '',
    status: 'todo' as TaskStatus,
    priority: 'medium' as TaskPriority,
    assignee: '',
    due_date: '',
    property: '',
    notes: '',
  })
  const [saving, setSaving] = useState(false)

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    setSaving(true)
    const res = await fetch('/api/tasks', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...form,
        assignee: form.assignee || null,
        due_date: form.due_date || null,
        property: form.property || null,
        notes: form.notes || null,
      }),
    })
    const task = await res.json()
    onAdd(task)
    onClose()
  }

  return (
    <div className="fixed inset-0 bg-black/60 flex items-center justify-center z-50">
      <div className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-xl w-full max-w-md p-6">
        <h2 className="text-lg font-semibold text-[#e8e8e8] mb-4">Add Task</h2>
        <form onSubmit={handleSubmit} className="space-y-3">
          <input
            required
            placeholder="Task title"
            value={form.title}
            onChange={(e) => setForm({ ...form, title: e.target.value })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] placeholder-[#555] focus:outline-none focus:border-lca-teal"
          />
          <div className="grid grid-cols-2 gap-3">
            <select
              value={form.priority}
              onChange={(e) => setForm({ ...form, priority: e.target.value as TaskPriority })}
              className="bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
            >
              <option value="low">Low</option>
              <option value="medium">Medium</option>
              <option value="high">High</option>
            </select>
            <select
              value={form.status}
              onChange={(e) => setForm({ ...form, status: e.target.value as TaskStatus })}
              className="bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
            >
              <option value="todo">To Do</option>
              <option value="inprogress">In Progress</option>
              <option value="done">Done</option>
            </select>
          </div>
          <select
            value={form.assignee}
            onChange={(e) => setForm({ ...form, assignee: e.target.value })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
          >
            <option value="">Unassigned</option>
            {staff.map((s) => (
              <option key={s.name} value={s.name}>{s.name}</option>
            ))}
          </select>
          <select
            value={form.property}
            onChange={(e) => setForm({ ...form, property: e.target.value })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
          >
            <option value="">No property</option>
            {properties.map((p) => (
              <option key={p.name} value={p.name}>{p.name}</option>
            ))}
          </select>
          <input
            type="date"
            value={form.due_date}
            onChange={(e) => setForm({ ...form, due_date: e.target.value })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
          />
          <textarea
            placeholder="Notes (optional)"
            value={form.notes}
            onChange={(e) => setForm({ ...form, notes: e.target.value })}
            rows={2}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] placeholder-[#555] focus:outline-none focus:border-lca-teal resize-none"
          />
          <div className="flex gap-2 pt-1">
            <button
              type="button"
              onClick={onClose}
              className="flex-1 py-2 rounded-md border border-[rgba(255,255,255,0.07)] text-sm text-[#888] hover:text-[#e8e8e8]"
            >
              Cancel
            </button>
            <button
              type="submit"
              disabled={saving}
              className="flex-1 py-2 rounded-md bg-lca-teal text-black text-sm font-medium hover:bg-lca-teal/90 disabled:opacity-50"
            >
              {saving ? 'Adding...' : 'Add Task'}
            </button>
          </div>
        </form>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create TaskDetailPanel**

Create `components/tasks/TaskDetailPanel.tsx`:

```typescript
'use client'

import { useState } from 'react'
import type { Task, TaskPriority, TaskStatus } from '@/lib/types'
import PriorityBadge from '@/components/ui/PriorityBadge'

interface TaskDetailPanelProps {
  task: Task
  onClose: () => void
  onUpdate: (task: Task) => void
  onDelete: (id: string) => void
  staff: { name: string }[]
  properties: { name: string }[]
}

export default function TaskDetailPanel({
  task,
  onClose,
  onUpdate,
  onDelete,
  staff,
  properties,
}: TaskDetailPanelProps) {
  const [form, setForm] = useState({
    title: task.title,
    status: task.status,
    priority: task.priority,
    assignee: task.assignee ?? '',
    due_date: task.due_date ?? '',
    property: task.property ?? '',
    notes: task.notes ?? '',
  })
  const [saving, setSaving] = useState(false)

  async function handleSave() {
    setSaving(true)
    const res = await fetch(`/api/tasks/${task.id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        ...form,
        assignee: form.assignee || null,
        due_date: form.due_date || null,
        property: form.property || null,
        notes: form.notes || null,
      }),
    })
    const updated = await res.json()
    onUpdate(updated)
    setSaving(false)
  }

  async function handleDelete() {
    if (!confirm('Delete this task?')) return
    await fetch(`/api/tasks/${task.id}`, { method: 'DELETE' })
    onDelete(task.id)
  }

  return (
    <div className="fixed inset-0 z-40" onClick={onClose}>
      <div
        className="absolute right-0 top-0 h-full w-full max-w-sm bg-lca-card border-l border-[rgba(255,255,255,0.07)] p-6 overflow-y-auto"
        onClick={(e) => e.stopPropagation()}
      >
        <div className="flex items-center justify-between mb-6">
          <h2 className="text-lg font-semibold text-[#e8e8e8]">Task Details</h2>
          <button onClick={onClose} className="text-[#555] hover:text-[#888] text-xl">×</button>
        </div>

        <div className="space-y-4">
          <div>
            <label className="block text-xs text-[#555] mb-1">Title</label>
            <input
              value={form.title}
              onChange={(e) => setForm({ ...form, title: e.target.value })}
              className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
            />
          </div>

          <div className="grid grid-cols-2 gap-3">
            <div>
              <label className="block text-xs text-[#555] mb-1">Status</label>
              <select
                value={form.status}
                onChange={(e) => setForm({ ...form, status: e.target.value as TaskStatus })}
                className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
              >
                <option value="todo">To Do</option>
                <option value="inprogress">In Progress</option>
                <option value="done">Done</option>
              </select>
            </div>
            <div>
              <label className="block text-xs text-[#555] mb-1">Priority</label>
              <select
                value={form.priority}
                onChange={(e) => setForm({ ...form, priority: e.target.value as TaskPriority })}
                className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
              >
                <option value="low">Low</option>
                <option value="medium">Medium</option>
                <option value="high">High</option>
              </select>
            </div>
          </div>

          <div>
            <label className="block text-xs text-[#555] mb-1">Assignee</label>
            <select
              value={form.assignee}
              onChange={(e) => setForm({ ...form, assignee: e.target.value })}
              className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
            >
              <option value="">Unassigned</option>
              {staff.map((s) => (
                <option key={s.name} value={s.name}>{s.name}</option>
              ))}
            </select>
          </div>

          <div>
            <label className="block text-xs text-[#555] mb-1">Property</label>
            <select
              value={form.property}
              onChange={(e) => setForm({ ...form, property: e.target.value })}
              className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
            >
              <option value="">No property</option>
              {properties.map((p) => (
                <option key={p.name} value={p.name}>{p.name}</option>
              ))}
            </select>
          </div>

          <div>
            <label className="block text-xs text-[#555] mb-1">Due Date</label>
            <input
              type="date"
              value={form.due_date}
              onChange={(e) => setForm({ ...form, due_date: e.target.value })}
              className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
            />
          </div>

          <div>
            <label className="block text-xs text-[#555] mb-1">Notes</label>
            <textarea
              value={form.notes}
              onChange={(e) => setForm({ ...form, notes: e.target.value })}
              rows={4}
              className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal resize-none"
            />
          </div>

          <div className="flex gap-2 pt-2">
            <button
              onClick={handleDelete}
              className="px-3 py-2 rounded-md border border-red-900/50 text-red-400 text-sm hover:bg-red-900/20"
            >
              Delete
            </button>
            <button
              onClick={handleSave}
              disabled={saving}
              className="flex-1 py-2 rounded-md bg-lca-teal text-black text-sm font-medium hover:bg-lca-teal/90 disabled:opacity-50"
            >
              {saving ? 'Saving...' : 'Save Changes'}
            </button>
          </div>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Create KanbanBoard**

Create `components/tasks/KanbanBoard.tsx`:

```typescript
'use client'

import { useState } from 'react'
import { DragDropContext, Droppable, Draggable, DropResult } from '@hello-pangea/dnd'
import type { Task, TaskStatus } from '@/lib/types'
import TaskCard from './TaskCard'
import TaskDetailPanel from './TaskDetailPanel'
import AddTaskModal from './AddTaskModal'

const COLUMNS: { id: TaskStatus; label: string }[] = [
  { id: 'todo', label: 'To Do' },
  { id: 'inprogress', label: 'In Progress' },
  { id: 'done', label: 'Done' },
]

interface KanbanBoardProps {
  initialTasks: Task[]
  staff: { name: string }[]
  properties: { name: string }[]
}

export default function KanbanBoard({ initialTasks, staff, properties }: KanbanBoardProps) {
  const [tasks, setTasks] = useState<Task[]>(initialTasks)
  const [selectedTask, setSelectedTask] = useState<Task | null>(null)
  const [showAddModal, setShowAddModal] = useState(false)
  const [filterAssignee, setFilterAssignee] = useState('')
  const [filterPriority, setFilterPriority] = useState('')
  const [filterProperty, setFilterProperty] = useState('')

  const filtered = tasks.filter((t) => {
    if (filterAssignee && t.assignee !== filterAssignee) return false
    if (filterPriority && t.priority !== filterPriority) return false
    if (filterProperty && t.property !== filterProperty) return false
    return true
  })

  async function onDragEnd(result: DropResult) {
    if (!result.destination) return
    const taskId = result.draggableId
    const newStatus = result.destination.droppableId as TaskStatus
    const task = tasks.find((t) => t.id === taskId)
    if (!task || task.status === newStatus) return

    setTasks((prev) =>
      prev.map((t) => (t.id === taskId ? { ...t, status: newStatus } : t))
    )
    await fetch(`/api/tasks/${taskId}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ status: newStatus }),
    })
  }

  function handleUpdate(updated: Task) {
    setTasks((prev) => prev.map((t) => (t.id === updated.id ? updated : t)))
    setSelectedTask(null)
  }

  function handleDelete(id: string) {
    setTasks((prev) => prev.filter((t) => t.id !== id))
    setSelectedTask(null)
  }

  function handleAdd(task: Task) {
    setTasks((prev) => [task, ...prev])
  }

  return (
    <div>
      {/* Filter bar */}
      <div className="flex gap-3 mb-6 flex-wrap">
        <select
          value={filterAssignee}
          onChange={(e) => setFilterAssignee(e.target.value)}
          className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-1.5 text-sm text-[#888] focus:outline-none focus:border-lca-teal"
        >
          <option value="">All Assignees</option>
          {staff.map((s) => <option key={s.name} value={s.name}>{s.name}</option>)}
        </select>
        <select
          value={filterPriority}
          onChange={(e) => setFilterPriority(e.target.value)}
          className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-1.5 text-sm text-[#888] focus:outline-none focus:border-lca-teal"
        >
          <option value="">All Priorities</option>
          <option value="high">High</option>
          <option value="medium">Medium</option>
          <option value="low">Low</option>
        </select>
        <select
          value={filterProperty}
          onChange={(e) => setFilterProperty(e.target.value)}
          className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-1.5 text-sm text-[#888] focus:outline-none focus:border-lca-teal"
        >
          <option value="">All Properties</option>
          {properties.map((p) => <option key={p.name} value={p.name}>{p.name}</option>)}
        </select>
        <button
          onClick={() => setShowAddModal(true)}
          className="ml-auto px-4 py-1.5 bg-lca-teal text-black text-sm font-medium rounded-md hover:bg-lca-teal/90"
        >
          + Add Task
        </button>
      </div>

      {/* Kanban columns */}
      <DragDropContext onDragEnd={onDragEnd}>
        <div className="grid grid-cols-3 gap-4">
          {COLUMNS.map((col) => {
            const colTasks = filtered.filter((t) => t.status === col.id)
            return (
              <div key={col.id} className="bg-lca-card rounded-xl border border-[rgba(255,255,255,0.07)]">
                <div className="px-4 py-3 border-b border-[rgba(255,255,255,0.07)] flex items-center justify-between">
                  <span className="text-sm font-medium text-[#888]">{col.label}</span>
                  <span className="text-xs text-[#555] bg-[#111] px-2 py-0.5 rounded-full">
                    {colTasks.length}
                  </span>
                </div>
                <Droppable droppableId={col.id}>
                  {(provided, snapshot) => (
                    <div
                      ref={provided.innerRef}
                      {...provided.droppableProps}
                      className={`p-3 space-y-2 min-h-[200px] transition-colors ${
                        snapshot.isDraggingOver ? 'bg-lca-teal/5' : ''
                      }`}
                    >
                      {colTasks.map((task, index) => (
                        <Draggable key={task.id} draggableId={task.id} index={index}>
                          {(provided) => (
                            <div
                              ref={provided.innerRef}
                              {...provided.draggableProps}
                              {...provided.dragHandleProps}
                            >
                              <TaskCard task={task} onClick={setSelectedTask} />
                            </div>
                          )}
                        </Draggable>
                      ))}
                      {provided.placeholder}
                    </div>
                  )}
                </Droppable>
              </div>
            )
          })}
        </div>
      </DragDropContext>

      {selectedTask && (
        <TaskDetailPanel
          task={selectedTask}
          onClose={() => setSelectedTask(null)}
          onUpdate={handleUpdate}
          onDelete={handleDelete}
          staff={staff}
          properties={properties}
        />
      )}

      {showAddModal && (
        <AddTaskModal
          onClose={() => setShowAddModal(false)}
          onAdd={handleAdd}
          staff={staff}
          properties={properties}
        />
      )}
    </div>
  )
}
```

- [ ] **Step 5: Create tasks page**

Create `app/tasks/page.tsx`:

```typescript
import AppLayout from '@/components/layout/AppLayout'
import KanbanBoard from '@/components/tasks/KanbanBoard'
import { createServerSupabaseClient } from '@/lib/supabase'

export default async function TasksPage() {
  const supabase = createServerSupabaseClient()
  const [{ data: tasks }, { data: staff }, { data: properties }] = await Promise.all([
    supabase.from('tasks').select('*').order('created_at', { ascending: false }),
    supabase.from('staff').select('name').order('name'),
    supabase.from('properties').select('name').eq('active', true).order('name'),
  ])

  return (
    <AppLayout>
      <div className="p-8">
        <h1 className="text-2xl font-bold text-[#e8e8e8] mb-6">Tasks</h1>
        <KanbanBoard
          initialTasks={tasks ?? []}
          staff={staff ?? []}
          properties={properties ?? []}
        />
      </div>
    </AppLayout>
  )
}
```

- [ ] **Step 6: Commit tasks page**

```bash
git add components/tasks/ app/tasks/
git commit -m "feat: add kanban board with drag-and-drop, task cards, add modal, and detail panel"
```

---

## Task 8: Staff Page

**Files:**
- Create: `components/staff/StaffCard.tsx`
- Create: `components/staff/StaffModal.tsx`
- Create: `components/staff/StaffGrid.tsx`
- Create: `app/staff/page.tsx`

- [ ] **Step 1: Create StaffCard**

Create `components/staff/StaffCard.tsx`:

```typescript
import type { Staff } from '@/lib/types'
import StatusBadge from '@/components/ui/StatusBadge'

const DAY_ORDER = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']

interface StaffCardProps {
  staff: Staff
  onClick: (staff: Staff) => void
}

export default function StaffCard({ staff, onClick }: StaffCardProps) {
  const initials = staff.name
    .split(' ')
    .map((n) => n[0])
    .join('')
    .toUpperCase()
    .slice(0, 2)

  const sortedShifts = [...staff.shifts].sort(
    (a, b) => DAY_ORDER.indexOf(a) - DAY_ORDER.indexOf(b)
  )

  return (
    <div
      onClick={() => onClick(staff)}
      className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-xl p-5 cursor-pointer hover:border-lca-teal/30 transition-colors"
    >
      <div className="flex items-start gap-4">
        <div className="w-10 h-10 rounded-full bg-lca-teal/20 text-lca-teal font-bold text-sm flex items-center justify-center flex-shrink-0">
          {initials}
        </div>
        <div className="flex-1 min-w-0">
          <div className="flex items-center justify-between gap-2 mb-0.5">
            <p className="text-sm font-medium text-[#e8e8e8] truncate">{staff.name}</p>
            <StatusBadge status={staff.status} />
          </div>
          <p className="text-xs text-[#888] mb-3">{staff.role}</p>
          <div className="flex flex-wrap gap-1">
            {sortedShifts.map((day) => (
              <span
                key={day}
                className="text-xs px-2 py-0.5 rounded bg-[#222] text-[#888]"
              >
                {day}
              </span>
            ))}
            {sortedShifts.length === 0 && (
              <span className="text-xs text-[#555]">No shifts</span>
            )}
          </div>
        </div>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create StaffModal**

Create `components/staff/StaffModal.tsx`:

```typescript
'use client'

import { useState } from 'react'
import type { Staff, StaffStatus } from '@/lib/types'

const ALL_DAYS = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun']

interface StaffModalProps {
  staff?: Staff
  onClose: () => void
  onSave: (staff: Staff) => void
  onDelete?: (id: string) => void
}

export default function StaffModal({ staff, onClose, onSave, onDelete }: StaffModalProps) {
  const [form, setForm] = useState({
    name: staff?.name ?? '',
    role: staff?.role ?? '',
    status: staff?.status ?? ('active' as StaffStatus),
    phone: staff?.phone ?? '',
    email: staff?.email ?? '',
    shifts: staff?.shifts ?? [],
    notes: staff?.notes ?? '',
  })
  const [saving, setSaving] = useState(false)

  function toggleDay(day: string) {
    setForm((prev) => ({
      ...prev,
      shifts: prev.shifts.includes(day)
        ? prev.shifts.filter((d) => d !== day)
        : [...prev.shifts, day],
    }))
  }

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault()
    setSaving(true)
    const payload = {
      ...form,
      phone: form.phone || null,
      email: form.email || null,
      notes: form.notes || null,
    }
    let result: Staff
    if (staff) {
      const res = await fetch(`/api/staff/${staff.id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      })
      result = await res.json()
    } else {
      const res = await fetch('/api/staff', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      })
      result = await res.json()
    }
    onSave(result)
    onClose()
  }

  async function handleDelete() {
    if (!staff || !onDelete) return
    if (!confirm(`Delete ${staff.name}?`)) return
    await fetch(`/api/staff/${staff.id}`, { method: 'DELETE' })
    onDelete(staff.id)
    onClose()
  }

  return (
    <div className="fixed inset-0 bg-black/60 flex items-center justify-center z-50">
      <div className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-xl w-full max-w-md p-6 max-h-[90vh] overflow-y-auto">
        <h2 className="text-lg font-semibold text-[#e8e8e8] mb-4">
          {staff ? 'Edit Staff Member' : 'Add Staff Member'}
        </h2>
        <form onSubmit={handleSubmit} className="space-y-3">
          <input
            required
            placeholder="Full name"
            value={form.name}
            onChange={(e) => setForm({ ...form, name: e.target.value })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] placeholder-[#555] focus:outline-none focus:border-lca-teal"
          />
          <input
            required
            placeholder="Role (e.g. Senior Cleaner)"
            value={form.role}
            onChange={(e) => setForm({ ...form, role: e.target.value })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] placeholder-[#555] focus:outline-none focus:border-lca-teal"
          />
          <select
            value={form.status}
            onChange={(e) => setForm({ ...form, status: e.target.value as StaffStatus })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] focus:outline-none focus:border-lca-teal"
          >
            <option value="active">Active</option>
            <option value="leave">On Leave</option>
            <option value="inactive">Inactive</option>
          </select>
          <input
            type="tel"
            placeholder="Phone (optional)"
            value={form.phone}
            onChange={(e) => setForm({ ...form, phone: e.target.value })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] placeholder-[#555] focus:outline-none focus:border-lca-teal"
          />
          <input
            type="email"
            placeholder="Email (optional)"
            value={form.email}
            onChange={(e) => setForm({ ...form, email: e.target.value })}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] placeholder-[#555] focus:outline-none focus:border-lca-teal"
          />
          <div>
            <p className="text-xs text-[#555] mb-2">Shifts</p>
            <div className="flex gap-2 flex-wrap">
              {ALL_DAYS.map((day) => (
                <button
                  key={day}
                  type="button"
                  onClick={() => toggleDay(day)}
                  className={`px-3 py-1 rounded text-xs font-medium transition-colors ${
                    form.shifts.includes(day)
                      ? 'bg-lca-teal text-black'
                      : 'bg-[#111] border border-[rgba(255,255,255,0.07)] text-[#888]'
                  }`}
                >
                  {day}
                </button>
              ))}
            </div>
          </div>
          <textarea
            placeholder="Notes (optional)"
            value={form.notes}
            onChange={(e) => setForm({ ...form, notes: e.target.value })}
            rows={2}
            className="w-full bg-[#111] border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-2 text-sm text-[#e8e8e8] placeholder-[#555] focus:outline-none focus:border-lca-teal resize-none"
          />
          <div className="flex gap-2 pt-1">
            {staff && onDelete && (
              <button
                type="button"
                onClick={handleDelete}
                className="px-3 py-2 rounded-md border border-red-900/50 text-red-400 text-sm hover:bg-red-900/20"
              >
                Delete
              </button>
            )}
            <button
              type="button"
              onClick={onClose}
              className="flex-1 py-2 rounded-md border border-[rgba(255,255,255,0.07)] text-sm text-[#888] hover:text-[#e8e8e8]"
            >
              Cancel
            </button>
            <button
              type="submit"
              disabled={saving}
              className="flex-1 py-2 rounded-md bg-lca-teal text-black text-sm font-medium hover:bg-lca-teal/90 disabled:opacity-50"
            >
              {saving ? 'Saving...' : 'Save'}
            </button>
          </div>
        </form>
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create StaffGrid**

Create `components/staff/StaffGrid.tsx`:

```typescript
'use client'

import { useState } from 'react'
import type { Staff } from '@/lib/types'
import StaffCard from './StaffCard'
import StaffModal from './StaffModal'

export default function StaffGrid({ initialStaff }: { initialStaff: Staff[] }) {
  const [staff, setStaff] = useState<Staff[]>(initialStaff)
  const [editingStaff, setEditingStaff] = useState<Staff | null>(null)
  const [showAddModal, setShowAddModal] = useState(false)

  function handleSave(updated: Staff) {
    setStaff((prev) => {
      const exists = prev.find((s) => s.id === updated.id)
      return exists ? prev.map((s) => (s.id === updated.id ? updated : s)) : [updated, ...prev]
    })
  }

  function handleDelete(id: string) {
    setStaff((prev) => prev.filter((s) => s.id !== id))
  }

  return (
    <div>
      <div className="flex justify-end mb-6">
        <button
          onClick={() => setShowAddModal(true)}
          className="px-4 py-2 bg-lca-teal text-black text-sm font-medium rounded-md hover:bg-lca-teal/90"
        >
          + Add Staff Member
        </button>
      </div>

      <div className="grid grid-cols-2 gap-4">
        {staff.map((s) => (
          <StaffCard key={s.id} staff={s} onClick={setEditingStaff} />
        ))}
      </div>

      {editingStaff && (
        <StaffModal
          staff={editingStaff}
          onClose={() => setEditingStaff(null)}
          onSave={handleSave}
          onDelete={handleDelete}
        />
      )}

      {showAddModal && (
        <StaffModal
          onClose={() => setShowAddModal(false)}
          onSave={handleSave}
        />
      )}
    </div>
  )
}
```

- [ ] **Step 4: Create staff page**

Create `app/staff/page.tsx`:

```typescript
import AppLayout from '@/components/layout/AppLayout'
import StaffGrid from '@/components/staff/StaffGrid'
import { createServerSupabaseClient } from '@/lib/supabase'

export default async function StaffPage() {
  const supabase = createServerSupabaseClient()
  const { data: staff } = await supabase
    .from('staff')
    .select('*')
    .order('name', { ascending: true })

  return (
    <AppLayout>
      <div className="p-8">
        <h1 className="text-2xl font-bold text-[#e8e8e8] mb-6">Staff</h1>
        <StaffGrid initialStaff={staff ?? []} />
      </div>
    </AppLayout>
  )
}
```

- [ ] **Step 5: Commit staff page**

```bash
git add components/staff/ app/staff/
git commit -m "feat: add staff grid with cards, add/edit modal, and shift pill toggles"
```

---

## Task 9: Properties Page

**Files:**
- Create: `components/properties/PropertyList.tsx`
- Create: `app/properties/page.tsx`

- [ ] **Step 1: Create PropertyList**

Create `components/properties/PropertyList.tsx`:

```typescript
'use client'

import { useState } from 'react'
import type { Property } from '@/lib/types'

export default function PropertyList({ initialProperties }: { initialProperties: Property[] }) {
  const [properties, setProperties] = useState<Property[]>(initialProperties)
  const [editingId, setEditingId] = useState<string | null>(null)
  const [editForm, setEditForm] = useState<Partial<Property>>({})
  const [filterCity, setFilterCity] = useState('')
  const [filterClient, setFilterClient] = useState('')

  const cities = [...new Set(properties.map((p) => p.city).filter(Boolean))] as string[]
  const clients = [...new Set(properties.map((p) => p.client).filter(Boolean))] as string[]

  const filtered = properties.filter((p) => {
    if (filterCity && p.city !== filterCity) return false
    if (filterClient && p.client !== filterClient) return false
    return true
  })

  function startEdit(p: Property) {
    setEditingId(p.id)
    setEditForm({ ...p })
  }

  async function saveEdit(id: string) {
    const res = await fetch(`/api/properties/${id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(editForm),
    })
    const updated = await res.json()
    setProperties((prev) => prev.map((p) => (p.id === id ? updated : p)))
    setEditingId(null)
  }

  async function toggleActive(p: Property) {
    const res = await fetch(`/api/properties/${p.id}`, {
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ active: !p.active }),
    })
    const updated = await res.json()
    setProperties((prev) => prev.map((x) => (x.id === p.id ? updated : x)))
  }

  return (
    <div>
      <div className="flex gap-3 mb-6 flex-wrap">
        <select
          value={filterCity}
          onChange={(e) => setFilterCity(e.target.value)}
          className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-1.5 text-sm text-[#888] focus:outline-none focus:border-lca-teal"
        >
          <option value="">All Cities</option>
          {cities.map((c) => <option key={c} value={c}>{c}</option>)}
        </select>
        <select
          value={filterClient}
          onChange={(e) => setFilterClient(e.target.value)}
          className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-md px-3 py-1.5 text-sm text-[#888] focus:outline-none focus:border-lca-teal"
        >
          <option value="">All Clients</option>
          {clients.map((c) => <option key={c} value={c}>{c}</option>)}
        </select>
      </div>

      <div className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-xl overflow-hidden">
        <table className="w-full text-sm">
          <thead>
            <tr className="border-b border-[rgba(255,255,255,0.07)]">
              <th className="text-left px-4 py-3 text-xs text-[#555] font-medium">Name</th>
              <th className="text-left px-4 py-3 text-xs text-[#555] font-medium">Client</th>
              <th className="text-left px-4 py-3 text-xs text-[#555] font-medium">City</th>
              <th className="text-left px-4 py-3 text-xs text-[#555] font-medium">Beds</th>
              <th className="text-left px-4 py-3 text-xs text-[#555] font-medium">Active</th>
              <th className="text-left px-4 py-3 text-xs text-[#555] font-medium">Notes</th>
              <th className="px-4 py-3"></th>
            </tr>
          </thead>
          <tbody>
            {filtered.map((p) => {
              const isEditing = editingId === p.id
              return (
                <tr key={p.id} className="border-b border-[rgba(255,255,255,0.04)] hover:bg-white/[0.02]">
                  <td className="px-4 py-3 text-[#e8e8e8]">
                    {isEditing ? (
                      <input
                        value={editForm.name ?? ''}
                        onChange={(e) => setEditForm({ ...editForm, name: e.target.value })}
                        className="bg-[#111] border border-lca-teal rounded px-2 py-1 text-sm text-[#e8e8e8] w-full focus:outline-none"
                      />
                    ) : p.name}
                  </td>
                  <td className="px-4 py-3 text-[#888]">
                    {isEditing ? (
                      <input
                        value={editForm.client ?? ''}
                        onChange={(e) => setEditForm({ ...editForm, client: e.target.value })}
                        className="bg-[#111] border border-lca-teal rounded px-2 py-1 text-sm text-[#e8e8e8] w-full focus:outline-none"
                      />
                    ) : (p.client ?? '—')}
                  </td>
                  <td className="px-4 py-3 text-[#888]">
                    {isEditing ? (
                      <input
                        value={editForm.city ?? ''}
                        onChange={(e) => setEditForm({ ...editForm, city: e.target.value })}
                        className="bg-[#111] border border-lca-teal rounded px-2 py-1 text-sm text-[#e8e8e8] w-full focus:outline-none"
                      />
                    ) : (p.city ?? '—')}
                  </td>
                  <td className="px-4 py-3 text-[#888]">
                    {isEditing ? (
                      <input
                        type="number"
                        value={editForm.bedrooms ?? ''}
                        onChange={(e) => setEditForm({ ...editForm, bedrooms: parseInt(e.target.value) || null })}
                        className="bg-[#111] border border-lca-teal rounded px-2 py-1 text-sm text-[#e8e8e8] w-16 focus:outline-none"
                      />
                    ) : (p.bedrooms ?? '—')}
                  </td>
                  <td className="px-4 py-3">
                    <button
                      onClick={() => toggleActive(p)}
                      className={`text-xs px-2 py-0.5 rounded font-medium ${
                        p.active
                          ? 'bg-teal-900/40 text-lca-teal'
                          : 'bg-[#333] text-[#888]'
                      }`}
                    >
                      {p.active ? 'Active' : 'Inactive'}
                    </button>
                  </td>
                  <td className="px-4 py-3 text-[#888]">
                    {isEditing ? (
                      <input
                        value={editForm.notes ?? ''}
                        onChange={(e) => setEditForm({ ...editForm, notes: e.target.value })}
                        className="bg-[#111] border border-lca-teal rounded px-2 py-1 text-sm text-[#e8e8e8] w-full focus:outline-none"
                      />
                    ) : (p.notes ?? '—')}
                  </td>
                  <td className="px-4 py-3 text-right">
                    {isEditing ? (
                      <div className="flex gap-2 justify-end">
                        <button
                          onClick={() => setEditingId(null)}
                          className="text-xs text-[#888] hover:text-[#e8e8e8]"
                        >
                          Cancel
                        </button>
                        <button
                          onClick={() => saveEdit(p.id)}
                          className="text-xs text-lca-teal hover:text-lca-teal/80"
                        >
                          Save
                        </button>
                      </div>
                    ) : (
                      <button
                        onClick={() => startEdit(p)}
                        className="text-xs text-[#555] hover:text-[#888]"
                      >
                        Edit
                      </button>
                    )}
                  </td>
                </tr>
              )
            })}
          </tbody>
        </table>
      </div>
    </div>
  )
}
```

- [ ] **Step 2: Create properties page**

Create `app/properties/page.tsx`:

```typescript
import AppLayout from '@/components/layout/AppLayout'
import PropertyList from '@/components/properties/PropertyList'
import { createServerSupabaseClient } from '@/lib/supabase'

export default async function PropertiesPage() {
  const supabase = createServerSupabaseClient()
  const { data: properties } = await supabase
    .from('properties')
    .select('*')
    .order('name', { ascending: true })

  return (
    <AppLayout>
      <div className="p-8">
        <h1 className="text-2xl font-bold text-[#e8e8e8] mb-6">Properties</h1>
        <PropertyList initialProperties={properties ?? []} />
      </div>
    </AppLayout>
  )
}
```

- [ ] **Step 3: Commit properties page**

```bash
git add components/properties/ app/properties/
git commit -m "feat: add properties table with inline editing and city/client filters"
```

---

## Task 10: Dashboard

**Files:**
- Create: `app/dashboard/page.tsx`

- [ ] **Step 1: Create dashboard page**

Create `app/dashboard/page.tsx`:

```typescript
import AppLayout from '@/components/layout/AppLayout'
import { createServerSupabaseClient } from '@/lib/supabase'
import Link from 'next/link'
import type { Task } from '@/lib/types'

function StatCard({ label, value, accent }: { label: string; value: number; accent?: boolean }) {
  return (
    <div className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-xl p-6">
      <p className="text-sm text-[#555] mb-1">{label}</p>
      <p className={`text-3xl font-bold ${accent ? 'text-lca-teal' : 'text-[#e8e8e8]'}`}>
        {value}
      </p>
    </div>
  )
}

export default async function DashboardPage() {
  const supabase = createServerSupabaseClient()
  const today = new Date().toISOString().split('T')[0]

  const [{ data: tasks }, { data: staff }, { data: properties }] = await Promise.all([
    supabase.from('tasks').select('*').order('updated_at', { ascending: false }),
    supabase.from('staff').select('*').eq('status', 'active'),
    supabase.from('properties').select('*').eq('active', true),
  ])

  const allTasks = (tasks ?? []) as Task[]
  const openTasks = allTasks.filter((t) => t.status === 'todo').length
  const inProgressTasks = allTasks.filter((t) => t.status === 'inprogress').length
  const activeStaff = (staff ?? []).length
  const activeProperties = (properties ?? []).length

  const urgentTasks = allTasks.filter(
    (t) =>
      t.status !== 'done' &&
      t.priority === 'high' &&
      t.due_date !== null &&
      t.due_date <= today
  )

  const recentActivity = allTasks.slice(0, 10)

  return (
    <AppLayout>
      <div className="p-8">
        <div className="flex items-center justify-between mb-8">
          <h1 className="text-2xl font-bold text-[#e8e8e8]">Dashboard</h1>
          <Link
            href="/tasks"
            className="px-4 py-2 bg-lca-teal text-black text-sm font-medium rounded-md hover:bg-lca-teal/90"
          >
            + Add Task
          </Link>
        </div>

        {/* Stat cards */}
        <div className="grid grid-cols-4 gap-4 mb-8">
          <StatCard label="Open Tasks" value={openTasks} />
          <StatCard label="In Progress" value={inProgressTasks} accent />
          <StatCard label="Active Staff" value={activeStaff} />
          <StatCard label="Active Properties" value={activeProperties} />
        </div>

        <div className="grid grid-cols-2 gap-6">
          {/* Urgent tasks */}
          <div className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-xl p-5">
            <h2 className="text-sm font-medium text-[#888] mb-4">Urgent Today</h2>
            {urgentTasks.length === 0 ? (
              <p className="text-sm text-[#555]">No urgent tasks due today.</p>
            ) : (
              <div className="space-y-2">
                {urgentTasks.map((t) => (
                  <div
                    key={t.id}
                    className="flex items-center justify-between text-sm"
                  >
                    <span className="text-[#e8e8e8] truncate">{t.title}</span>
                    <span className="text-xs text-[#555] ml-2 flex-shrink-0">
                      {t.assignee ?? 'Unassigned'}
                    </span>
                  </div>
                ))}
              </div>
            )}
          </div>

          {/* Recent activity */}
          <div className="bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-xl p-5">
            <h2 className="text-sm font-medium text-[#888] mb-4">Recent Activity</h2>
            <div className="space-y-2">
              {recentActivity.map((t) => (
                <div key={t.id} className="flex items-center justify-between text-sm">
                  <span className="text-[#e8e8e8] truncate">{t.title}</span>
                  <span
                    className={`text-xs ml-2 flex-shrink-0 ${
                      t.status === 'done'
                        ? 'text-lca-teal'
                        : t.status === 'inprogress'
                        ? 'text-blue-400'
                        : 'text-[#555]'
                    }`}
                  >
                    {t.status === 'inprogress' ? 'in progress' : t.status}
                  </span>
                </div>
              ))}
            </div>
          </div>
        </div>
      </div>
    </AppLayout>
  )
}
```

- [ ] **Step 2: Commit dashboard**

```bash
git add app/dashboard/
git commit -m "feat: add dashboard with stat cards, urgent tasks, and recent activity"
```

---

## Task 11: AI Chat API Route

**Files:**
- Create: `app/api/ai/chat/route.ts`

- [ ] **Step 1: Create streaming AI chat route**

Create `app/api/ai/chat/route.ts`:

```typescript
import { anthropic, buildSystemPrompt } from '@/lib/anthropic'
import { createServerSupabaseClient } from '@/lib/supabase'
import type { Task, Staff, Property } from '@/lib/types'

export async function POST(request: Request) {
  const { message, history } = await request.json() as {
    message: string
    history: { role: 'user' | 'assistant'; content: string }[]
  }

  const supabase = createServerSupabaseClient()
  const [{ data: tasks }, { data: staff }, { data: properties }] = await Promise.all([
    supabase.from('tasks').select('*').neq('status', 'done'),
    supabase.from('staff').select('*').eq('status', 'active'),
    supabase.from('properties').select('*').eq('active', true),
  ])

  const systemPrompt = buildSystemPrompt(
    (tasks ?? []) as Task[],
    (staff ?? []) as Staff[],
    (properties ?? []) as Property[]
  )

  // Save user message to DB (fire-and-forget)
  supabase.from('ai_chat_history').insert({ role: 'user', content: message }).then(() => {})

  const stream = anthropic.messages.stream({
    model: 'claude-sonnet-4-5',
    max_tokens: 400,
    system: systemPrompt,
    messages: [
      ...history.map((m) => ({ role: m.role, content: m.content })),
      { role: 'user', content: message },
    ],
  })

  // Stream response and capture full text for DB save
  const encoder = new TextEncoder()
  let fullResponse = ''

  const readable = new ReadableStream({
    async start(controller) {
      for await (const chunk of stream) {
        if (
          chunk.type === 'content_block_delta' &&
          chunk.delta.type === 'text_delta'
        ) {
          const text = chunk.delta.text
          fullResponse += text
          controller.enqueue(encoder.encode(text))
        }
      }
      // Save assistant response to DB after streaming
      await supabase
        .from('ai_chat_history')
        .insert({ role: 'assistant', content: fullResponse })
      controller.close()
    },
  })

  return new Response(readable, {
    headers: {
      'Content-Type': 'text/plain; charset=utf-8',
      'Transfer-Encoding': 'chunked',
    },
  })
}
```

- [ ] **Step 2: Commit AI route**

```bash
git add app/api/ai/
git commit -m "feat: add streaming AI chat API route with live business context"
```

---

## Task 12: AI Chat UI

**Files:**
- Create: `components/ai/MessageBubble.tsx`
- Create: `components/ai/ChatWindow.tsx`
- Create: `app/ai/page.tsx`

- [ ] **Step 1: Install react-markdown types**

```bash
npm install @types/react-markdown 2>/dev/null || true
```

- [ ] **Step 2: Create MessageBubble**

Create `components/ai/MessageBubble.tsx`:

```typescript
import ReactMarkdown from 'react-markdown'
import type { ChatMessage } from '@/lib/types'

export default function MessageBubble({ message }: { message: ChatMessage }) {
  const isUser = message.role === 'user'

  return (
    <div className={`flex ${isUser ? 'justify-end' : 'justify-start'}`}>
      <div
        className={`max-w-[80%] rounded-xl px-4 py-3 text-sm ${
          isUser
            ? 'bg-lca-teal text-black'
            : 'bg-lca-card border border-[rgba(255,255,255,0.07)] text-[#e8e8e8]'
        }`}
      >
        {isUser ? (
          <p>{message.content}</p>
        ) : (
          <ReactMarkdown
            components={{
              p: ({ children }) => <p className="mb-2 last:mb-0">{children}</p>,
              ul: ({ children }) => <ul className="list-disc list-inside mb-2">{children}</ul>,
              ol: ({ children }) => <ol className="list-decimal list-inside mb-2">{children}</ol>,
              li: ({ children }) => <li className="mb-0.5">{children}</li>,
              strong: ({ children }) => <strong className="font-semibold">{children}</strong>,
              code: ({ children }) => (
                <code className="bg-black/20 px-1 rounded text-xs font-mono">{children}</code>
              ),
            }}
          >
            {message.content}
          </ReactMarkdown>
        )}
      </div>
    </div>
  )
}
```

- [ ] **Step 3: Create ChatWindow**

Create `components/ai/ChatWindow.tsx`:

```typescript
'use client'

import { useRef, useState, useEffect } from 'react'
import type { ChatMessage } from '@/lib/types'
import MessageBubble from './MessageBubble'

const QUICK_ACTIONS = [
  "What's urgent today?",
  "Who's available this weekend?",
  'Summarise team workload',
  'Draft a message to a client',
]

export default function ChatWindow({ initialHistory }: { initialHistory: ChatMessage[] }) {
  const [messages, setMessages] = useState<ChatMessage[]>(initialHistory)
  const [input, setInput] = useState('')
  const [streaming, setStreaming] = useState(false)
  const bottomRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' })
  }, [messages])

  async function sendMessage(text: string) {
    if (!text.trim() || streaming) return
    setInput('')
    setStreaming(true)

    const userMsg: ChatMessage = {
      id: Date.now().toString(),
      role: 'user',
      content: text,
      created_at: new Date().toISOString(),
    }
    setMessages((prev) => [...prev, userMsg])

    const assistantMsg: ChatMessage = {
      id: (Date.now() + 1).toString(),
      role: 'assistant',
      content: '',
      created_at: new Date().toISOString(),
    }
    setMessages((prev) => [...prev, assistantMsg])

    const history = messages.map((m) => ({ role: m.role, content: m.content }))

    const res = await fetch('/api/ai/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ message: text, history }),
    })

    const reader = res.body!.getReader()
    const decoder = new TextDecoder()
    let accumulated = ''

    while (true) {
      const { done, value } = await reader.read()
      if (done) break
      accumulated += decoder.decode(value, { stream: true })
      setMessages((prev) =>
        prev.map((m) =>
          m.id === assistantMsg.id ? { ...m, content: accumulated } : m
        )
      )
    }

    setStreaming(false)
  }

  async function clearHistory() {
    if (!confirm('Clear all chat history?')) return
    await fetch('/api/ai/chat/history', { method: 'DELETE' })
    setMessages([])
  }

  return (
    <div className="flex flex-col h-[calc(100vh-140px)]">
      {/* Quick action chips */}
      <div className="flex gap-2 mb-4 flex-wrap">
        {QUICK_ACTIONS.map((action) => (
          <button
            key={action}
            onClick={() => sendMessage(action)}
            disabled={streaming}
            className="px-3 py-1.5 bg-lca-card border border-[rgba(255,255,255,0.07)] text-xs text-[#888] rounded-full hover:border-lca-teal/40 hover:text-[#e8e8e8] transition-colors disabled:opacity-50"
          >
            {action}
          </button>
        ))}
      </div>

      {/* Messages */}
      <div className="flex-1 overflow-y-auto space-y-4 pr-1">
        {messages.length === 0 && (
          <p className="text-sm text-[#555] text-center mt-12">
            Ask me anything about your business.
          </p>
        )}
        {messages.map((m) => (
          <MessageBubble key={m.id} message={m} />
        ))}
        <div ref={bottomRef} />
      </div>

      {/* Input */}
      <div className="mt-4 flex gap-2">
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === 'Enter' && !e.shiftKey && sendMessage(input)}
          placeholder="Ask about tasks, staff, scheduling..."
          disabled={streaming}
          className="flex-1 bg-lca-card border border-[rgba(255,255,255,0.07)] rounded-lg px-4 py-3 text-sm text-[#e8e8e8] placeholder-[#555] focus:outline-none focus:border-lca-teal disabled:opacity-50"
        />
        <button
          onClick={() => sendMessage(input)}
          disabled={streaming || !input.trim()}
          className="px-4 py-3 bg-lca-teal text-black text-sm font-medium rounded-lg hover:bg-lca-teal/90 disabled:opacity-50"
        >
          {streaming ? '...' : 'Send'}
        </button>
        <button
          onClick={clearHistory}
          className="px-3 py-3 border border-[rgba(255,255,255,0.07)] text-[#555] text-xs rounded-lg hover:text-[#888]"
          title="Clear history"
        >
          ✕
        </button>
      </div>
    </div>
  )
}
```

- [ ] **Step 4: Create clear history API route**

Create `app/api/ai/chat/history/route.ts`:

```typescript
import { NextResponse } from 'next/server'
import { createServerSupabaseClient } from '@/lib/supabase'

export async function DELETE() {
  const supabase = createServerSupabaseClient()
  const { error } = await supabase.from('ai_chat_history').delete().neq('id', '00000000-0000-0000-0000-000000000000')
  if (error) return NextResponse.json({ error: error.message }, { status: 500 })
  return new NextResponse(null, { status: 204 })
}
```

- [ ] **Step 5: Create AI page**

Create `app/ai/page.tsx`:

```typescript
import AppLayout from '@/components/layout/AppLayout'
import ChatWindow from '@/components/ai/ChatWindow'
import { createServerSupabaseClient } from '@/lib/supabase'

export default async function AiPage() {
  const supabase = createServerSupabaseClient()
  const { data: history } = await supabase
    .from('ai_chat_history')
    .select('*')
    .order('created_at', { ascending: true })

  return (
    <AppLayout>
      <div className="p-8">
        <h1 className="text-2xl font-bold text-[#e8e8e8] mb-6">AI Assistant</h1>
        <ChatWindow initialHistory={history ?? []} />
      </div>
    </AppLayout>
  )
}
```

- [ ] **Step 6: Commit AI chat UI**

```bash
git add components/ai/ app/ai/ app/api/ai/
git commit -m "feat: add AI chat UI with streaming responses, quick actions, and persistent history"
```

---

## Task 13: Final Wiring and Build Check

**Files:**
- Modify: `app/globals.css` (ensure Tailwind base styles)

- [ ] **Step 1: Ensure globals.css uses Tailwind directives**

Replace the content of `app/globals.css` with:

```css
@tailwind base;
@tailwind components;
@tailwind utilities;

* {
  box-sizing: border-box;
}

body {
  background-color: #0d0d0d;
}

/* Custom scrollbar for dark theme */
::-webkit-scrollbar {
  width: 6px;
}
::-webkit-scrollbar-track {
  background: transparent;
}
::-webkit-scrollbar-thumb {
  background: rgba(255,255,255,0.1);
  border-radius: 3px;
}
::-webkit-scrollbar-thumb:hover {
  background: rgba(255,255,255,0.2);
}
```

- [ ] **Step 2: Run build to check for TypeScript/compilation errors**

```bash
npm run build
```

Expected: Build succeeds. If errors appear, fix them before proceeding. Common issues:
- Missing `'use client'` on components that use `useState`/`useEffect`
- Type mismatches — check `lib/types.ts` definitions match usage

- [ ] **Step 3: Start dev server and verify pages load**

```bash
npm run dev
```

Open `http://localhost:3000` — should redirect to `/dashboard`. With placeholder env vars, Clerk will show an error on protected routes — this is expected until real keys are added.

- [ ] **Step 4: Create .gitignore entry for .env.local**

Verify `.env.local` is already in `.gitignore` (create-next-app adds this by default). Check:

```bash
grep ".env.local" .gitignore
```

Expected output: `.env.local`

- [ ] **Step 5: Final commit**

```bash
git add -A
git commit -m "feat: complete LCA Control Panel v1 — all pages wired and build verified"
```

---

## Task 14: Deployment to Vercel

- [ ] **Step 1: Create GitHub repo and push**

```bash
gh repo create lca-control-panel --private --source=. --remote=origin --push
```

Or manually: create a new private repo at github.com, then:

```bash
git remote add origin https://github.com/YOUR_USERNAME/lca-control-panel.git
git push -u origin main
```

- [ ] **Step 2: Set up Supabase**

1. Go to [supabase.com](https://supabase.com) → Sign up → New project
2. Name: `lca-control-panel`, Region: `ap-southeast-2` (Sydney, closest to NZ)
3. Settings → API → copy **Project URL**, **anon public key**, **service_role key**
4. SQL Editor → paste and run `supabase/migrations/001_init.sql`
5. Verify tables appear under Table Editor

- [ ] **Step 3: Set up Clerk**

1. Go to [clerk.com](https://clerk.com) → Sign up → Create application
2. Name: `LCA Control Panel`, enable Email sign-in
3. API Keys → copy **Publishable Key** and **Secret Key**
4. Users → Invite users (add Shana's email)

- [ ] **Step 4: Get Anthropic API key**

1. Go to [console.anthropic.com](https://console.anthropic.com)
2. API Keys → Create new key → copy it

- [ ] **Step 5: Deploy to Vercel**

1. Go to [vercel.com](https://vercel.com) → New Project → Import from GitHub → select `lca-control-panel`
2. Settings → Environment Variables → add all 6 vars:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `SUPABASE_SERVICE_ROLE_KEY`
   - `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
   - `CLERK_SECRET_KEY`
   - `ANTHROPIC_API_KEY`
3. Deploy

- [ ] **Step 6: Update .env.local with real keys**

Fill in real values in `.env.local` for local development. This file is gitignored — safe to store real keys locally.

- [ ] **Step 7: Verify live deployment**

Visit your Vercel URL (e.g. `lca-control-panel.vercel.app`):
- Sign in with Clerk
- Dashboard loads with seed data stats
- Tasks kanban shows 5 seeded tasks
- Staff grid shows 4 seeded staff
- Properties table shows 4 seeded properties
- AI assistant responds to a test message

---

## Self-Review Notes

**Spec coverage check:**
- Dashboard with 4 stat cards ✓ (Task 10)
- Today's urgent tasks + recent activity ✓ (Task 10)
- Kanban with drag-and-drop ✓ (Task 7)
- Task card: title, priority badge, assignee, due date, property tag ✓ (Task 7)
- Slide-out detail panel ✓ (Task 7)
- Filter bar on tasks ✓ (Task 7)
- Add task modal ✓ (Task 7)
- Staff grid 2-col ✓ (Task 8)
- Staff card: initials avatar, name, role, status badge, shift pills ✓ (Task 8)
- Staff edit modal ✓ (Task 8)
- Properties table with inline edit ✓ (Task 9)
- Filter by city/client ✓ (Task 9)
- AI persistent history ✓ (Tasks 11, 12)
- Streaming responses ✓ (Task 11)
- Quick-action chips ✓ (Task 12)
- Clear history button ✓ (Task 12)
- Markdown rendering ✓ (Task 12)
- Seed data ✓ (Task 3)
- LCA brand colours + logo ✓ (Tasks 1, 5)
- Clerk auth ✓ (Tasks 1, 5)
- All API routes ✓ (Task 4)
- Deployment checklist ✓ (Task 14)
