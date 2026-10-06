# The congress field brief — the canonical pattern

This repository is **the reference implementation for every Ora congress field
brief from here on.** EURETINA 2026 is the build that proved it: run live in
Vienna, on phones, by a team of ten, for five days.

**Do not reference the ESCRS form.** It is superseded. When asked to build a
brief for a new congress, start from this repository and nothing else.

---

## 1. The ground rule, before anything else

**This repository is public. The app lives here. The data never does.**

Never commit briefs, meeting details, contacts, passwords, SQL load scripts,
Outlook or Salesforce exports, or a downloaded snapshot. `.gitignore` blocks
the usual file types as a backstop, but it is a backstop, not permission —
anything carrying real meeting or company data stays out whatever it is called.

This extends to code comments. A comment naming a real company we are meeting
says more than it needs to. Write round it.

Everything the team types lives in Supabase, behind the sign-in. The only
things in git are the app, its two vendored libraries, the icons and this file.

---

## 2. What the thing actually is

**One file.** `index.html` — ~6,900 lines, all CSS and JS inline. No build
step, no bundler, no framework, no npm install. You edit it and it ships.

This is deliberate and it is not up for renegotiation on a whim. During a
congress the BD lead is on a phone in a conference hall with poor signal and
fifteen minutes between meetings. A build pipeline is a thing that can break
at the worst possible moment. One file cannot.

**Two vendored dependencies**, served from the repo, never a CDN:
`supabase.js` and `xlsx.full.min.js`. The only third-party request the page
makes is Google Fonts.

**Hosted on GitHub Pages from `main`.** A push deploys in about 40 seconds.
There is no staging. Verify before you push.

### The spine of the file

Sections are marked with `/* ---------- name ---------- */` banners. Follow
them; they are the file's table of contents. Roughly in order: theme tokens
and CSS → day/venue definitions → the exhibitor floor list → state → the
meeting model → account roll-ups → sign-in → persistence → the change log →
Salesforce handover → search → the executive overview → day timelines →
company cards → the editor sheets → events → boot.

---

## 3. Data model

A single generic Supabase table does all of it:

```
docs { collection text, id text, data jsonb }
```

Collections: `meetings`, `proposed`, `targets`, `outcomes`, `debriefs`,
`briefs`, `pis`, `meta`, `log`.

A small Firestore-shaped shim sits over it — `db.doc("meetings/" + id).set(m)`
and `db.collection("meetings").onSnapshot(...)` — so reads are live across every
phone in the team without anyone refreshing.

**`slug()` is the universal key.** Lowercase, `&` → `and`, anything not
alphanumeric → `-`, trimmed, capped at 60 chars. Company identity across
meetings, briefs, targets and the exhibitor floor is `slug(name)` and nothing
else. Briefs are looked up by `slug(brief.name)`, **not** by document id —
get this wrong and a brief silently fails to attach to its card.

### A meeting reaches more than one company

`m.also` is a list of other companies in the room. Counts stay **per meeting**;
coverage goes **per company**. A joint dinner is one meeting in every total,
but each company named is reached, shows the meeting on its own card, and is
asked for its own brief. Use `meetingCos(m)` and `meetingCovers(m, accId)` —
never read `m.company` directly when deciding what an account has had.

Deliberately *not* applied where a meeting must stay singular: an opportunity
belongs to one company, and one debrief is owed per meeting, not per guest.

---

## 4. Access

**Supabase Auth, plus a `members` table** of `{email, name, role}`. Sign in,
then the email (lowercased, exact match) must be on that list. `role === "editor"`
grants write; anything else is read-only.

**The role check in the browser is cosmetic.** It hides buttons. The actual
lock is Row Level Security in Postgres. Policies must grant to `authenticated`
only — never to `anon` — so that the publishable key sitting in this public
repo is worth nothing to a stranger.

Expected policies: `docs` readable by any member, writable only by editors;
`members` readable by the signed-in user. `lock-down-check.sql` (untracked,
regenerate as needed) holds the verification queries.

**RLS enabled is not the same as policies existing.** Policies on a table with
RLS switched off are inert — they look correct and enforce nothing. Always
check `pg_class.relrowsecurity` as well as `pg_policies`, and prove it with an
anonymous `curl` against `/rest/v1/docs`. An empty array means locked.

---

## 5. What makes it work in the field

These are the decisions that earned their place. Keep them.

**Phone first, genuinely.** Not "responsive" as an afterthought — the primary
user is one-handed, standing up, in a hall. Test at 390px before 1100px.
The breakpoint that carries the phone layout is 760px; 359px is the narrow
fallback. Several narrower component breakpoints exist besides those.

**Exactly one thing scrolls.** Sheets fill the screen, take their height from
the overlay around them, and scroll in the body only — header and footer
pinned. Never size a sheet with `100dvh`; on iOS it disagrees with a fixed
overlay the moment Safari's toolbar moves, and you get two competing scrollers
and a page that reads as frozen. Give scrolling children `min-height:0` and
`overscroll-behavior:contain`, and hold the page behind still while a sheet
is open.

**Every list has a way out.** Clear buttons, close buttons, Escape. If the only
way to undo something is to hold backspace, that is a bug.

**No dead controls.** A disabled button fires no click and reads as broken.
Keep controls live and let the result explain itself.

**The brief arrives as one paste.** Nobody retypes a brief into four fields on
a phone. `parseBriefBlock()` takes a whole block of text and splits it on its
headings; `fillBriefForm()` fills only the sections it found and never wipes
what is already there.

**Nothing requires the database console.** If a task needs someone to open
Supabase mid-congress, it is not finished. Imports, briefs, debriefs, new
companies — all of it goes in from a phone.

**Data is copied, never moved.** When a floor conversation is placed on a day,
its write-up is carried across, the original is left intact, and the carry
refuses to run if the target already has a real debrief. A mis-tap must never
be able to lose something written on the floor.

**Say what things are.** The import reads Outlook meeting invites, so the
button says invites, not Salesforce. Labels that send people looking for a
system they will never open are a real cost.

### The four-part company brief

Every company card carries the same four sections. Do not invent new ones:

1. **What they do** — who they are, plainly
2. **Where they stand** — stage, pipeline, recent news
3. **Why it matters to us** — the Ora angle, concretely
4. **Where to start** — the opening line for the person in the room

---

## 6. Testing

**Drive the real page. Do not reason about the code and call it verified.**

`const SNAPSHOT = null;` is the seam — search for it rather than trusting a
line number. Replace it with a
seeded JSON object and the page boots with that data and no network. Load it
in Playwright (Chromium is at `/opt/pw-browsers/chromium`; Playwright itself
lives in `/opt/node22/lib/node_modules`, so set `NODE_PATH`).

Then use real clicks, real typing, real wheel and touch events, and a faked
clock via `addInitScript` where time matters. Measure the DOM — scroll heights,
bounding boxes, computed styles — rather than trusting a screenshot alone.

This discipline has caught, at minimum: a toggle rendered `disabled` so it
could never fire; `Intl` rendering September as "Sept" against a stored "Sep"
so a date comparison was always false; a now-line placed by start time so it
sat below a meeting still running; a debrief carry-over stranded inside a
branch that never executed; and the two-scroller bug above. None of those were
visible by reading the diff.

---

## 7. Starting the next congress

1. Fork or copy this repository. Keep it private if the plan allows; if it
   must be public, the ground rule in §1 is not optional.
2. Point it at a fresh Supabase project. Create `docs` and `members`.
   **Enable RLS and verify it before a single real brief goes in.**
3. Update, in `index.html`: the `DAYS` array (keys, labels, dates, venues),
   the exhibitor floor list, the congress name in the masthead, and the
   copyright line.
4. Clear the brief cache and any sample data.
5. Seed `members` with the team and their roles.
6. Then write the briefs — four sections each, pasted in from a phone.

Deploy, open it on a phone, and walk one full day of the agenda before anyone
else sees it.

---

## 8. Working here

- Commit messages explain **why**, not what. Match the surrounding style.
- Comments in the code do the same. They are plain-spoken and they say what a
  decision cost and what it prevents. Keep that register.
- Never put a model name or identifier in a commit, comment or any other
  artifact pushed to the repo.
- Push to `main` — that is what Pages serves.
- After pushing, tell the user to **force-close** the app before re-testing.
  iOS caches hard and a backgrounded web app will show the old version.
