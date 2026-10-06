# Standing up the next congress

Read `CLAUDE.md` first — it says what this app is and why. This file is the
runbook for pointing it at a new congress.

**Do not rebuild this app from a description.** It is ~6,900 lines of one file
carrying several weeks of field-tested fixes that are not visible in any
summary of it. Fork the repository. Everything below assumes you did.

A fresh Claude Code session can execute this whole file. Give it the new
congress details and tell it to work through the steps in order.

---

## Step 0 — Fork

Fork or "Use this template" on GitHub. Keep it private if the plan allows.
If it must be public, §1 of `CLAUDE.md` is not optional.

Then **Settings → Pages → Deploy from a branch → `main` / root.** A push
deploys in about 40 seconds.

---

## Step 1 — A fresh Supabase project

Never reuse the previous congress's project. New congress, new database.

Create the project, then paste this into **SQL Editor** in one go. It creates
the schema and locks it in the same breath — there is no window where the
tables exist unprotected.

```sql
-- Two tables carry everything.
create table if not exists public.docs (
  collection text not null,
  id         text not null,
  data       jsonb not null default '{}'::jsonb,
  primary key (collection, id)
);

create table if not exists public.members (
  email text primary key,
  name  text not null,
  role  text not null default 'viewer'   -- 'editor' or 'viewer'
);

-- Lock them before a single real brief goes in.
alter table public.docs    enable row level security;
alter table public.members enable row level security;

-- Who is on the team. SECURITY DEFINER so the check is not itself subject to
-- the policy that uses it, which would recurse forever.
create or replace function public.my_role() returns text
  language sql stable security definer set search_path = public as $$
  select m.role from public.members m
  where  m.email = lower(auth.jwt() ->> 'email');
$$;

-- Grants are to "authenticated" only. The anonymous role gets nothing, so the
-- publishable key sitting in the page source is worth nothing to a stranger.
drop policy if exists docs_read   on public.docs;
drop policy if exists docs_insert on public.docs;
drop policy if exists docs_update on public.docs;
drop policy if exists docs_delete on public.docs;

create policy docs_read   on public.docs for select to authenticated
  using (public.my_role() is not null);
create policy docs_insert on public.docs for insert to authenticated
  with check (public.my_role() = 'editor');
create policy docs_update on public.docs for update to authenticated
  using (public.my_role() = 'editor') with check (public.my_role() = 'editor');
create policy docs_delete on public.docs for delete to authenticated
  using (public.my_role() = 'editor');

-- You may read your own row on the team list and nobody else's.
drop policy if exists members_read on public.members;
drop policy if exists members_self on public.members;
create policy members_self on public.members for select to authenticated
  using (email = lower(auth.jwt() ->> 'email'));
```

### Verify it before going further

```sql
-- Both must be true. Policies on a table with RLS off are inert: they look
-- correct and enforce nothing. This is the check people skip.
select relname, relrowsecurity from pg_class
where  relnamespace = 'public'::regnamespace
and    relname in ('docs','members');

-- Expect 4 rows for docs, 1 for members, all granted to {authenticated}.
select tablename, policyname, cmd, roles::text
from   pg_policies
where  schemaname = 'public' and tablename in ('docs','members')
order  by tablename, cmd;
```

**Then prove it from outside.** This is what a stranger with your page source
can do:

```
curl "https://<project>.supabase.co/rest/v1/docs?select=*&limit=5" \
  -H "apikey: <your publishable key>"
```

`[]` means locked. Rows coming back means you are not done.

---

## Step 2 — Point the app at it

In `index.html`, near the top of the script:

```js
const SUPABASE_URL = "https://<project>.supabase.co";
const SUPABASE_KEY = "<publishable key>";   // the publishable/anon key, never the service key
```

The **service role key must never appear in this file.** It bypasses RLS
entirely and the file is served to every browser that opens the app.

---

## Step 3 — The congress itself

Five data blocks and a handful of labels. Search for the `const` name; line
numbers drift.

| What | Where | Notes |
|---|---|---|
| `FORUM` | agenda array | Sessions for a satellite/pre-congress day. Delete if there isn't one. |
| `EIS` | agenda array | Same, for the second pre-congress track. |
| `PROGRAMMES` | wraps the two above | Heading, intro and footnote per programme day. |
| `DAYS` | the day tabs | One object per tab. See below. |
| `EXHIBITORS` | the floor | `{n:"Company", b:"B50"}` — name and stand number. |

`COMPANIES` is **derived** from `PROGRAMMES` at load. Never hand-edit it.

### The shape of a day

```js
{key:"thu1",                       // internal id, used in meeting records
 short:"Thu", label:"Thu 1",       // tab text, narrow and normal
 long:"Thursday 1 October",
 date:"1 Oct 2026",                // "D Mon YYYY" — the now-line parses this
 sub:"Congress Day 1",
 venue:"Venue name, City",
 event:"The congress name 2027",
 ec:"euretina",                    // CSS class for the event colour
 place:"viecon"}                   // CSS class for the venue colour
```

`date` must stay in `D Mon YYYY` form. The now-line parses the month through
a lookup table precisely because locale month names are not dependable —
`Intl` in `en-GB` renders September as "Sept", which silently broke this once.

`ec` and `place` are CSS class suffixes. If you introduce new ones, add the
matching rules in the stylesheet or those chips render unstyled.

### Labels to change

Roughly 30 occurrences each of the congress name and the city, plus the venue
acronym. Search and replace, then read the diff — some are in prose that needs
rewriting, not just renaming.

- `<title>` and the `<h1>` in the masthead
- the copyright block at the top of the file, the `author` and `copyright`
  meta tags, and the year
- the calendar-invite subject prefix (search for the congress name in the
  Salesforce handover section)
- the `intro` and `foot` strings in `PROGRAMMES`

---

## Step 4 — The team

Insert one row per person. `editor` can write; anything else is read-only.

```sql
insert into public.members (email, name, role) values
  ('first.last@ora.com', 'First Last', 'editor'),
  ('someone@ora.com',    'Someone',    'viewer');
```

Then create each person an **Auth user** (Authentication → Users → Add user)
with the same email. Both halves are required: Auth proves who they are, the
`members` row says whether they are on this congress.

Send passwords individually, never in the group message.

---

## Step 5 — Briefs

Four sections each — **What they do / Where they stand / Why it matters to us
/ Where to start.** Paste them in from the app's own brief box, one block at a
time. Nothing here requires the Supabase console.

---

## Step 6 — Walk it before anyone else sees it

Deploy, open it on a phone, and walk one full day of the agenda end to end:
open a meeting, log a debrief, run a search, clear it, open a gap sheet and
scroll it to the bottom.

Then share the link.

---

## What a fresh session should be told

Paste something like this into a new Claude Code session on the forked repo:

> This repo is a fork of our EURETINA 2026 congress field brief. Read
> `CLAUDE.md` and `SETUP.md` first.
>
> We are standing it up for **\<congress name, city, dates\>**.
>
> Work through `SETUP.md` steps 2–3: point it at the new Supabase project
> (I'll give you the URL and publishable key), replace the `DAYS` array with
> the new day tabs, clear `FORUM`/`EIS`/`PROGRAMMES` and `EXHIBITORS` ready
> for the new programme, and update the title, masthead and copyright.
>
> Don't change anything else. The app's behaviour is already correct and was
> tested in the field — treat §5 of `CLAUDE.md` as settled.
>
> I'll give you the programme, the exhibitor list and the companies after
> that.

The last paragraph matters. Without it a fresh session will treat a working
app as a draft and start improving things that were already decided.
