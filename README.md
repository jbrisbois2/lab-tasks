# Lab Tasks

A dead-simple shared weekly task list for the lab. One list, everyone sees it, everyone can add and check off. Every action is stamped with a name and timestamp — that attribution is the whole point.

**Live site:** https://jbrisbois2.github.io/lab-tasks/

No accounts, no passwords. You pick your name from a shared list (or add yourself if you're new). That name gets stamped on every task you add or check off.

---

## First-time Supabase setup

The site talks to a Supabase project for shared state. If you're setting this up from scratch (or forking it), do these steps once.

### 1. Create a project

Sign up at [supabase.com](https://supabase.com) (free tier is fine) and create a new project. Wait for it to finish provisioning.

### 2. Run this SQL in the Supabase SQL Editor

Open **SQL Editor** in the Supabase dashboard, paste the following, and click **Run**:

```sql
-- ============================================================
-- Table: tasks
-- ============================================================
create table if not exists public.tasks (
  id uuid primary key default gen_random_uuid(),
  title text not null check (char_length(title) between 1 and 500),
  created_by text not null check (char_length(created_by) between 1 and 40),
  created_at timestamptz not null default now(),
  completed_by text check (completed_by is null or char_length(completed_by) between 1 and 40),
  completed_at timestamptz,
  day_of_week text check (day_of_week is null or day_of_week in ('Mon','Tue','Wed','Thu','Fri','Sat','Sun')),
  scheduled_week_start date not null default date_trunc('week', now())::date
);

alter table public.tasks enable row level security;

drop policy if exists "anon can read tasks" on public.tasks;
create policy "anon can read tasks" on public.tasks
  for select to anon using (true);

drop policy if exists "anon can insert tasks" on public.tasks;
create policy "anon can insert tasks" on public.tasks
  for insert to anon with check (true);

drop policy if exists "anon can update tasks" on public.tasks;
create policy "anon can update tasks" on public.tasks
  for update to anon using (true) with check (true);

drop policy if exists "anon can delete tasks" on public.tasks;
create policy "anon can delete tasks" on public.tasks
  for delete to anon using (true);

grant select, insert, update, delete on public.tasks to anon;

-- ============================================================
-- Table: lab_members (shared name list)
-- ============================================================
create table if not exists public.lab_members (
  id uuid primary key default gen_random_uuid(),
  name text not null check (char_length(name) between 1 and 40),
  created_at timestamptz not null default now()
);

create unique index if not exists lab_members_name_ci_uniq
  on public.lab_members (lower(name));

alter table public.lab_members enable row level security;

drop policy if exists "anon can read members" on public.lab_members;
create policy "anon can read members" on public.lab_members
  for select to anon using (true);

drop policy if exists "anon can insert members" on public.lab_members;
create policy "anon can insert members" on public.lab_members
  for insert to anon with check (true);

grant select, insert on public.lab_members to anon;

-- ============================================================
-- Enable realtime on both tables
-- ============================================================
do $$
begin
  if not exists (select 1 from pg_publication_tables
                 where pubname = 'supabase_realtime' and schemaname='public' and tablename='tasks')
  then execute 'alter publication supabase_realtime add table public.tasks'; end if;
  if not exists (select 1 from pg_publication_tables
                 where pubname = 'supabase_realtime' and schemaname='public' and tablename='lab_members')
  then execute 'alter publication supabase_realtime add table public.lab_members'; end if;
end $$;
```

If realtime doesn't turn on via SQL, enable it in the UI: **Database → Replication → `supabase_realtime` → toggle `tasks` and `lab_members` on**.

### 3. Wire up the keys

Open `index.html`. Near the top of the `<script>` block you'll find:

```js
const SUPABASE_URL = 'https://YOUR-PROJECT.supabase.co';
const SUPABASE_ANON_KEY = 'eyJ...';
```

Replace both with the values from **Supabase Dashboard → Project Settings → API**:
- `SUPABASE_URL` → the **Project URL**
- `SUPABASE_ANON_KEY` → the **`anon` / `public`** key (NOT the `service_role` key)

Commit and push. GitHub Pages will redeploy in ~1 minute.

---

## Rotating / changing the keys later

Same as step 3 above: edit the two constants at the top of `index.html`'s inline script, commit, push. That's it — there's no build step and no config file.

If you're rotating the anon key because it leaked or someone's abusing it:
1. In Supabase Dashboard → Project Settings → API, click **Reset** on the anon key.
2. Paste the new key into `index.html`, commit, push.

---

## How to use (send this to your labmates)

Go to **https://jbrisbois2.github.io/lab-tasks/**. First time you visit, pick your name from the shared list — or click **"+ Add new person…"** if you're not on it yet. Your choice is saved in your browser so you only do this once (there's a "change" link next to your name if you need to switch). Type a task in the box at the top, choose which week it's for (defaults to this week; you can pick up to 6 weeks ahead), and hit **Add**. Each task has two pills you can click anytime to change: a **week** pill (move it to a different week) and a **day** pill (tag it with Mon–Sun within that week). Anyone can change either at any time. Check the box to mark a task done, uncheck to bring it back. If someone made a typo or added something by mistake, click the small **×** on the task to delete it (there's a confirmation prompt). Tasks are grouped under headers like "This week", "Next week", "Last week (overdue)", etc., with this week highlighted. The page updates live as others make changes — no refresh needed. Checked-off tasks move to the **Completed** section at the bottom (click to expand) as a record of what got done.

---

## Free-tier maintenance

- **GitHub Pages**: free forever for public repos. No action needed.
- **Supabase free tier pauses after ~7 days of inactivity.** If nobody in the lab visits the site for a week, the database is paused and the site will show "Load error" until someone restores it. To restore: go to https://supabase.com/dashboard/project/hhfjjtyllxckjuvvpsrl and click the **Restore project** button in the banner at the top. Takes ~30 seconds; all data is preserved.

---

## Security note

The anon key is embedded in the page source — that's expected for Supabase's public-write model. Row-Level Security (RLS) policies (see SQL above) are what actually enforce safety. With those policies in place, anyone with the anon key can:

- **SELECT, INSERT, UPDATE, DELETE** rows in `public.tasks`
- **SELECT, INSERT** rows in `public.lab_members` (no delete/update — names are permanent)

They **cannot** read or write any other table in your Supabase project (new tables default to RLS-blocked, and no policies grant the anon role access to them).

Worst case: a stranger who finds the URL could add junk tasks, check things off, or delete tasks. If that happens, rotate the anon key (see above). If deletion becomes a problem, you can also remove the `delete` policy and grant on the tasks table to make tasks permanent again.
