---
name: secrets-audit-checklist
description: 'Use when a user wants to tidy up or audit the secrets in Railway apps, especially apps that talk to Supabase: "are my Railway secrets sealed", "which env vars are duplicates or unused", "move to the new Supabase sb_secret / publishable keys", "can I turn off the legacy anon and service_role keys". A short, ordered checklist with no scripts: a names-only inventory of variables, sorting them into secrets, duplicates and dead entries, sealing and consolidating into shared variables, removing dead ones, a quick look at Supabase exposure in the dashboard, then moving backends to Supabase secret keys one service at a time and switching the legacy keys off last. The agent never prints or handles secret values, and the user confirms every change.'
---

# Secrets audit checklist (Railway + Supabase)

A plain checklist for cleaning up secrets in apps deployed on Railway, and for moving off Supabase's legacy JWT keys. Work through it in order with the user. Not affiliated with or endorsed by Railway or Supabase.

## Why now
- Supabase is deprecating the legacy `anon` and `service_role` keys by the end of 2026. Every project still on them has a migration to the new keys ahead: `sb_secret_...` for backends and `sb_publishable_...` for browsers and apps.
- The legacy keys keep working alongside the new ones until someone disables them. That's what makes a slow, one-service-at-a-time move possible.

Don't tell the user their setup is fine or broken based on this checklist alone. It finds things to look at; the user confirms them.

## Safety rules (follow all of them)
1. **Never print, read, store or pass along a secret value**, or any part of one. Work with variable names only. Don't run `railway variables`, `railway variable list` or `railway run env`, don't open a `.env` file (listing `.env*` file names is fine), and don't call an MCP tool that returns variable values or raw log text. Code searches print file names and variable names only, never the matching line.
2. **The user handles every value.** They create new keys in the Supabase dashboard and paste them into the Railway dashboard. The agent may write a `${{shared.NAME}}` reference, never a value. Never ask for a key, token or password in the chat. If the user pastes one anyway, don't repeat it, and add that secret to the list of things to rotate.
3. **The user confirms each change, one at a time.** Before each one, say what will change, what might break and how to undo it. Approving one service doesn't approve the next.
4. **Never call the user "secure" or "done".** Say what was checked, what changed, and what is still pending.
5. Don't rotate the legacy JWT secret, delete Supabase keys, change row data, or accept someone else's staged Railway changes.

## Before you start
- Confirm scope: which Railway projects and environments (production only, or staging too), which Supabase projects, and every other place the same keys live: CI secrets, serverless functions on other hosts, Edge Functions, Database Webhooks, cron boxes, local `.env` files, shipped mobile or desktop builds.
- If one Supabase project serves several apps, or both production and test, every key change touches all of them. Plan around all of them.
- Ask for quiet windows (no deploys, low traffic, the user reachable) and agree on how you'll know each service is healthy: a heartbeat or last-run time, a table that should keep getting fresh rows, a page that should return 200.
- Note each service's current deployment ID. That's the rollback target.
- If the CLI says Unauthorized, the user runs `railway login` in their own terminal.

## 1. Inventory: variable names only
Goal: one list of every variable **name** per project, environment and service, plus shared variables.
- With a Railway MCP connector, use tools that return names only. If one shows a value, stop using it and don't repeat what it showed.
- Otherwise the user reads the names from each service's **Variables** tab (not the Raw Editor, which shows values) and types them in. Shared variables are under Project Settings, Shared Variables.
- For each name, note whether it's sealed and whether it's a `${{shared.X}}` reference or a literal value. The CLI shows rendered values, so a reference and a copy look identical there, and most connectors can't see sealed status. Mark both columns **unverified** until the user checks them in the dashboard.
- Note any pending (staged) changes in each environment. Don't deploy them; ask whoever made them.

## 2. Sort the list
Read-only. For each variable, decide with the user:
- [ ] **Secret or not.** API keys, tokens, private keys, signing and webhook secrets, database URLs with a password, the legacy `service_role` key and any `sb_secret_` key are secrets. The publishable key and the legacy `anon` key are meant to be public, but track them for the migration.
- [ ] **Possible duplicates and drift.** The same key stored as separate literal copies on several services (each should become one shared variable), or the same name holding different values on different services (ask which is right; never guess). Without seeing values you can only flag these as "possible"; the user confirms in the dashboard.
- [ ] **Dead entries.** Variables the code never reads. Search the repos by name and print file names only. Also note the reverse: code that reads a name the service doesn't define. Report it; don't add the variable, since that can switch on code nobody tested.
- [ ] **Unsealed secrets**, but only once the user has confirmed in the dashboard that the source really isn't sealed. A reference to a sealed shared variable is fine.
- [ ] **Legacy Supabase keys.** Every place that holds `anon` or `service_role`, on Railway and off it.
- [ ] **Over-exposure.** Services holding keys they don't need: a read-only dashboard with an admin key, a test worker with a production key, a shared variable referenced by a service that never reads it.
- [ ] **Client readiness.** For each service, the Supabase client version that's actually installed (from the lockfile or build log, not the version range). Some older clients reject the new non-JWT keys when the client is created. Raw HTTP callers must send the key in the `apikey` header; if they also send `Authorization: Bearer`, it has to be the same key.

Write the result up as a numbered plan: each change, its risk, how to undo it, and who does it (agent or user in the dashboard). The user reviews it before anything changes.

## 3. Seal and consolidate
For each duplicated secret, always in this order:
- [ ] The user creates a shared variable in the dashboard, pastes the value and **seals** it (3-dot menu, then Seal). Sealing is dashboard-only and permanent: the value can never be viewed again, so the user needs an offline copy or a way to regenerate it first.
- [ ] Confirm the shared variable is live, not just staged. A reference to a staged-only shared variable resolves to an empty value.
- [ ] One service at a time, with approval: the agent sets `NAME=${{shared.SHARED_NAME}}`. In a shell, single-quote it (`railway variable set 'NAME=${{shared.SHARED_NAME}}' ...`), because double quotes make bash try to expand `${{`. The CLI redeploys straight away unless told not to.
- [ ] Check the new deployment succeeded, the health signals are fresh and no new auth errors (401, 403, "invalid key") have appeared. Then the next service.
- [ ] The user seals any remaining literal secrets that stay per-service. Warn them that sealed values aren't copied to PR or preview environments, so previews that need them need a separate test credential.

Undo: set the variable back, or the user clicks Rollback on the recorded deployment. If the old value was never visible, the agent can't restore it by hand, so check that before the first change.

## 4. Remove dead entries
- [ ] For each dead variable, show the name, where it lives, the evidence it's unused, and what would break if that evidence is wrong.
- [ ] Get approval per variable (or for an explicit list), then delete it with the CLI or have the user delete it in the dashboard. Many connectors can't delete.
- [ ] Redeploy and check for errors mentioning the missing name.

Undo: the user re-adds the variable, with the value from their own records. Leave legacy Supabase key variables for the very end (step 7).

## 5. A quick look at Supabase exposure (read-only)
This is a pass through the Supabase dashboard, not a full database audit. The user looks; the agent explains what matters.
- [ ] **Security Advisor**: note what it flags, especially tables with RLS turned off.
- [ ] **Tables with RLS off** in exposed schemas: if the public key can write to them, anyone holding that key can change those rows. Find out which tables a public client reads, since those reads must keep working.
- [ ] Turning RLS on with no policy blocks reads as well as writes, which can break a public page. That's a separate, bigger decision; don't rush it.
- [ ] **New tables**: per Supabase's changelog, existing projects stop exposing new tables automatically from 30 October 2026. Tables created before that keep the grants they had.
- [ ] **Functions (RPCs)** the public can call, especially `SECURITY DEFINER` ones, which run with the owner's rights.
- [ ] Check the API logs for real anon or authenticated writes before proposing any change.

Write anything odd down as "needs follow-up". Any fix here is its own approved change with its own undo.

## 6. Move backends to the new keys
Legacy keys stay **enabled** for this whole step. They're the rollback.
- [ ] The user decides: one secret key per backend group (a leak then means rotating one key) or one shared key for all backends (simpler).
- [ ] The user creates the secret key in Supabase (Settings, API Keys), pastes it into a Railway shared variable and seals it. The agent confirms it's live.
- [ ] Order the services from lowest risk to highest. The critical one goes last, with the user around.

Per service, with approval:
- [ ] Point the service's key variable at `${{shared.SUPABASE_SECRET_KEY}}` (or whatever the user named it) and redeploy.
- [ ] Check: the deployment succeeded, no "Invalid API key" or 401 in the boot log, fresh writes still land in a table the service owns (reads alone don't prove much), and the Supabase API logs show this service now using the new key.
- [ ] Wait the agreed soak time before the next one.

Undo: point the variable back (the legacy key still works), or roll back the deployment. Browser and mobile clients move to the publishable key; secret keys are refused on requests that look like they come from a browser. Don't forget the holders outside Railway from "Before you start".

## 7. Soak, then switch the legacy keys off
- [ ] Every service has run clean on the new keys for the agreed time (48 hours is a sensible default, longer if there are weekly jobs).
- [ ] Supabase shows no "last used" for the legacy keys, so the evidence is the API logs: over the soak window, no requests with a legacy key. Report counts, not log lines. Think about weekly crons, old app versions, CI and webhooks.
- [ ] Remove the leftover legacy key variables (step 4 rules).
- [ ] With explicit approval, the user deactivates the legacy keys in the dashboard. This can be turned back on if a missed client shows up. Rotating the legacy JWT secret is a different step that can't be undone; it's out of scope here.
- [ ] Re-check health signals and API logs right after, and again later the same day.

## 8. Keep a short record
Write a short report the user keeps: scope, what was sealed, consolidated and deleted, which services moved to which key and how each was verified, the Supabase findings and what was done about them, what's still pending and who owns it, and what couldn't be checked. Names, dates, deployment IDs and counts only, no values. Give change times in the user's own time zone, labelled.
