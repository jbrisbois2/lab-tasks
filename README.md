# Lab Tasks

A dead-simple shared weekly task list for the lab. One list, everyone sees it, everyone can add and check off. Every action is stamped with a name and timestamp — that attribution is the whole point.

**Live site:** https://jbrisbois2.github.io/lab-tasks/

No accounts, no passwords. You type your name once (saved in your browser), and it goes on every task you add or check off.

---

## First-time Supabase setup

The site talks to a Supabase project for shared state. If you're setting this up from scratch (or forking it), do these steps once.

### 1. Create a project

Sign up at [supabase.com](https://supabase.com) (free tier is fine) and create a new project. Wait for it to finish provisioning.

### 2. Run this SQL in the Supabase SQL Editor

Open **SQL Editor** in the Supabase dashboard, paste the following, and click **Run**:

```sql
-- Table
create table if not exists public.tasks (
  id uuid primary key default gen_random_uuid(),
  title text not null check (char_length(title) between 1 and 500),
  created_by text not null check (char_length(created_by) between 1 and 40),
  created_at timestamptz not null default now(),
  completed_by text check (completed_by is null or char_length(completed_by) between 1 and 40),
  completed_at timestamptz
);

-- Row-Level Security
alter table public.tasks enable row level security;

-- Anyone (anon key) can read, insert, and update. DELETE is intentionally omitted.
create policy "anon can read tasks"
  on public.tasks for select
  to anon
  using (true);

create policy "anon can insert tasks"
  on public.tasks for insert
  to anon
  with check (true);

create policy "anon can update tasks"
  on public.tasks for update
  to anon
  using (true)
  with check (true);

-- Explicit grants (RLS still enforces the policies above)
grant select, insert, update on public.tasks to anon;

-- Enable realtime for the tasks table
do $$
begin
  if not exists (
    select 1 from pg_publication_tables
    where pubname = 'supabase_realtime'
      and schemaname = 'public'
      and tablename = 'tasks'
  ) then
    execute 'alter publication supabase_realtime add table public.tasks';
  end if;
end $$;
```

If the realtime `alter publication` step fails for any reason, you can enable it in the UI: **Database → Replication → `supabase_realtime` → toggle `tasks` on**.

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

Go to **https://jbrisbois2.github.io/lab-tasks/**. First time you visit, enter your name — it's saved in your browser so you only do it once (there's a "change" link next to your name if you need to). Type a task in the box at the top and hit **Add**; check the box next to any task to mark it done. Anyone in the lab can add or check off anything, and the page updates live as others make changes — no refresh needed. Checked-off tasks move to the **Completed** section at the bottom (click to expand) and stay there permanently as a record of what got done. If someone checks something off by mistake, just uncheck it.

---

## Security note

The anon key is embedded in the page source — that's expected for Supabase's public-write model, and the Row-Level Security policies above are what actually enforce safety. With those policies in place, anyone with the anon key can only:

- **SELECT, INSERT, UPDATE** rows in `public.tasks`

They **cannot**:

- **DELETE** any row (no delete policy — completed tasks are archived forever)
- Read or write any other table in your Supabase project (new tables default to RLS-blocked, and no policies grant the anon role access to them)

Worst case: a stranger who finds the URL adds junk tasks or checks things off. If that happens, rotate the anon key (see above).
