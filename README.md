# Bloom Touchpoint Tracker

Turns the Inoreader tag feeds (Exec Departure, Exec Hire, Ownership/Investment,
Client Watch) into a live, filterable dashboard of relevant BD signal —
qualified, summarized, and contact-enriched by Claude, no manual triage.

```
┌─────────────┐     ┌──────────────────┐     ┌───────────────┐     ┌───────────────┐
│  Inoreader   │ --> │  Claude Code job │ --> │   Supabase    │ --> │  Dashboard    │
│  tag feeds   │     │ (GitHub Actions, │     │  (Postgres +  │     │  (Next.js on  │
│  (RSS URLs)  │     │  daily cron)     │     │  Auth)        │     │  Vercel)      │
└─────────────┘     └──────────────────┘     └───────────────┘     └───────────────┘
```

## What's in here

- `prompts/qualify.md` — the qualification + enrichment logic Claude Code runs
  against each day's new tagged headlines. This is the file to edit if the
  filtering rules need adjusting (what counts as relevant, contact-lookup
  rules, etc.) — no code changes needed for that.
- `supabase/schema.sql` — the Postgres table. Run once in the Supabase SQL editor.
- `data/bloom-clients.json` — **you maintain this.** Bloom's client/prospect
  list and known contacts. Used both for the "Existing Client" match and to
  skip a web search when you already know the right contact.
- `scripts/run_tracker.py` — the daily job: fetch feeds, diff against Supabase,
  call Claude Code, write results.
- `.github/workflows/tracker.yml` — the schedule (7am ET daily + manual trigger).
- `dashboard/` — the Next.js app. Deploy this to Vercel.

## Setup order

1. **Supabase**: create a new project (or reuse your existing Bloom Supabase
   org from the sporting model portal project). Run `supabase/schema.sql` in
   the SQL editor. Under Authentication, enable Email (magic link) as a
   sign-in method. Grab three values from Project Settings -> API:
   - `Project URL`
   - `anon public` key
   - `service_role` key (keep this one secret — it's for the GitHub Action only)

2. **Edit `data/bloom-clients.json`** with your real client list and any
   known contacts.

3. **GitHub repo secrets** (Settings -> Secrets and variables -> Actions):
   - `ANTHROPIC_API_KEY`
   - `SUPABASE_URL`
   - `SUPABASE_SERVICE_KEY`
   - `INOREADER_FEED_URLS` — comma-separated, no spaces, e.g.
     `https://www.inoreader.com/stream/.../tag/exec-departure,https://www.inoreader.com/stream/.../tag/exec-hire,...`

4. **Test the job manually** before trusting the schedule: go to the Actions
   tab, select "Touchpoint Tracker", click "Run workflow". Check the Supabase
   table editor afterward to confirm rows landed.

5. **Deploy the dashboard**: import the repo into Vercel, set the **root
   directory to `dashboard`**, and add the two `NEXT_PUBLIC_*` env vars from
   `dashboard/.env.local.example` (anon key only — never put the service role
   key in the dashboard's environment).

6. **Invite colleagues**: in Supabase, under Authentication -> Users, they can
   just visit the dashboard URL and request a magic link themselves — no
   invite step needed unless you want to restrict sign-ups, in which case
   disable public sign-ups in Supabase Auth settings and add users manually.

## A few honest notes

- The `article_url` UNIQUE constraint in the database *is* the dedup ledger —
  there's no separate "seen items" file to keep in sync. Simpler than the
  file-based version we sketched before Supabase was in the picture.
- The contact-lookup step is the most failure-prone part of the pipeline by
  nature — some stories just don't have a clean, confident contact. The
  dashboard shows this honestly via the confidence dot rather than hiding it;
  "Not found" is a correct answer, not a bug.
- Local Claude Code testing: you can run `python3 scripts/run_tracker.py`
  directly on your machine (with the four env vars exported) before wiring up
  GitHub Actions, to sanity-check the whole flow end to end.
