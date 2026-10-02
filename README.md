# Listenva Document Processor (GitHub Actions)

This tiny repo contains **one file that matters**: `.github/workflows/process-documents.yml`.

It does not contain any Listenva application code, and it doesn't need to —
all it does is repeatedly call an endpoint that already exists and already
works on your live InfinityFree deployment (`cron/process.php`). Nothing
about Listenva's PHP code changes because of this repo.

## What problem this solves

Phase 0 testing found that a single real processing step (one TTS chunk) can
take 15–45 seconds. Pinging that endpoint once every few minutes (the
original Cloudflare Worker plan) meant a real document could take days to
finish, because almost all the time between calls is wasted doing nothing.

This workflow calls the same endpoint dozens of times back-to-back during
each run (up to ~25 minutes of continuous calling), with a short pause
between calls, then stops. A schedule re-triggers it every 10 minutes as a
safety net. Net effect: processing keeps moving almost continuously instead
of advancing one step every several minutes.

## One-time setup

1. **Create this repo on GitHub** — public, as discussed (keeps Actions
   minutes free and unlimited; nothing secret lives in the code itself).
2. **Add your existing `CRON_SECRET` as a GitHub Actions secret** — not a
   new value, reuse the one already in your live `.env`:
   - Repo → **Settings** → **Secrets and variables** → **Actions** →
     **New repository secret**
   - Name: `LISTENVA_CRON_SECRET`
   - Value: the same `CRON_SECRET` value currently in your live `.env`
3. **Commit `.github/workflows/process-documents.yml`** to the repo (on the
   default branch — usually `main`).
4. That's it. GitHub will pick up the schedule automatically within a few
   minutes. You can also go to the **Actions** tab → **Listenva Document
   Processor** → **Run workflow** to trigger it immediately instead of
   waiting for the schedule.

## One config change worth making on the live `.env`

Based on what Phase 0 measured (a single TTS chunk can take up to ~45
seconds), it's safer to make each individual call to `cron/process.php`
handle fewer documents at once, so one HTTP call can't accidentally try to
do more work than InfinityFree's execution window allows. In the live
`.env`, set:

```
WORKER_MAX_DOCUMENTS_PER_RUN=1
```

(It currently defaults to 20 if unset — fine when steps are fast, risky now
that we know a single step can take up to 45 seconds. Setting it to 1 means
each call does at most one document's one step, and the GitHub Actions loop
compensates by simply calling it again immediately.)

## How to watch it work

**Actions tab → Listenva Document Processor → click any run** shows the
live log — you'll see each call's output (`document 9: chunk_generated:1/3`
etc.) scroll by in real time.

## Checking on document 9 (already sitting in `generating_audio` from Phase 0 testing)

Once this workflow is running, you don't need to manually reload the
benchmark `step` URL anymore — this will pick document 9 up on its own
within the next scheduled run (or immediately if you manually trigger it)
and keep advancing it automatically. Check its progress any time with the
same `report`/`verify` benchmark URLs as before, or just watch the Actions
log.
