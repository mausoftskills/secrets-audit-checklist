# Secrets Audit Checklist (Railway + Supabase)

A free agent skill (SKILL.md format) for tidying up the secrets in apps deployed on Railway, and for getting off Supabase's legacy `anon` and `service_role` keys before they go away. Your AI agent works through a short, ordered checklist with you: a names-only inventory of your variables, sorting out secrets, duplicates and dead entries, sealing and consolidating into shared variables, a quick look at Supabase exposure, then moving your backends to the new `sb_secret_` keys one service at a time and switching the legacy keys off last.

The agent never prints or handles secret values. You create and paste keys yourself, and you confirm every change before it happens. There are no scripts; it's a checklist the agent follows.

## Why bother
Supabase is deprecating the legacy JWT keys (`anon` and `service_role`) by the end of 2026, so every project still using them has this migration ahead. Most apps that have been running a while also pick up the same key pasted into four services, variables nobody reads any more, and secrets that were never sealed. It's easier to fix all of that in one careful pass than one surprise at a time.

## Install
Copy the `secrets-audit-checklist` folder (the one with `SKILL.md` directly inside) into your agent's skills folder. The folder name has to match the `name` in SKILL.md.

- Claude Code: `~/.claude/skills/` (or `.claude/skills/` in one project)
- Cursor: `~/.cursor/skills/` (or `.cursor/skills/` in one project)
- Other agents that read SKILL.md: their own skills folder, often `.agents/skills/`

For example:
```bash
git clone https://github.com/mausoftskills/secrets-audit-checklist
mkdir -p ~/.claude/skills
cp -r secrets-audit-checklist/secrets-audit-checklist ~/.claude/skills/
```

Then ask your agent something like:
> Go through the secrets audit checklist with me for my Railway project. Don't change anything without asking.

The agent works best with the Railway CLI logged in (run `railway login` yourself) or a Railway MCP connector, but it can walk you through the dashboards by hand too. If you only use Railway, steps 1-4 work for any secret. If you only use Supabase, steps 5-7 still apply; you change the key in your own host's settings.

## What it covers
1. An inventory of variable names (never values), with sealed and reference status marked unverified until you check the dashboard.
2. Sorting: secrets, possible duplicates and drift, dead entries, unsealed secrets, legacy Supabase keys, over-exposed services, and whether your Supabase client version can take the new keys.
3. Sealing and consolidating duplicated secrets into sealed shared variables, one service at a time.
4. Removing dead variables, with evidence and your approval for each one.
5. A quick, read-only look at Supabase exposure in the dashboard: Security Advisor, tables with RLS off, public functions, and the October 2026 change to how new tables are exposed.
6. Moving backends to Supabase secret keys one service at a time, with the legacy keys left on as the rollback.
7. A soak period, a check of the API logs, and only then switching the legacy keys off.
8. A short written record of what was done and what's still pending.

## What the paid version adds
This checklist is meant to be enough on its own if you're careful. The paid pack, [Railway + Supabase Secrets Audit & Key-Rotation Lockdown](https://www.agensi.io/skills/railway-supabase-secrets-audit-key-rotation) ($24), is the same process with tooling around it, for people with many services or who want the agent to do more of the legwork safely:

- An offline Python masker (standard library only) that turns Railway's variable output into a masked table: first 4 characters and length, a shape label, and a per-session fingerprint, so the agent can spot real duplicates and drift without the values reaching the chat. A names-only mode accepts only structured JSON and refuses pasted text.
- A read-only Supabase SQL audit (sections A to K): RLS off with anon writes, wide-open policies, views that bypass RLS, sequences anon can reset, RPCs anon can execute, default privileges, role membership, Auth usage, places inside the database that look like they hold a key, and rollback snapshot generators.
- Lockdown templates for approved fixes, each with its undo and verification queries.
- An optional probe script that tests reads, empty writes, RPCs and header handling against your own Supabase URL.
- Name-only code scan and client version commands, and a count-only log check for verifying each cutover.
- 36 real-world gotchas from a production audit, detailed phase guides with checklists and rollbacks, and a report template the agent fills in.

## Limits
This is a checklist, not a guarantee. It doesn't read your database or your values, so it can only flag things for you to confirm, and finishing it doesn't mean your systems are secure. It means the steps were done carefully and written down. Supabase's and Railway's behaviour can change, so check their docs if something doesn't match.

Railway and Supabase are trademarks of their owners. This project is not affiliated with or endorsed by either.

## License
MIT. See [LICENSE](LICENSE).
