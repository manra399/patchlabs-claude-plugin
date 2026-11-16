---
name: patchlabs-leak-check
description: Check an app built with Lovable, Supabase, Bolt, Replit or Cursor for personal-data leaks (row level security mistakes, exposed keys, open views and buckets, weak security headers), explain each finding in plain English, and draft a fix migration for the user to approve. Use when the user asks whether their app leaks data, asks to check RLS or Supabase policies, asks for a security check before launch, or opens a repo with a supabase/migrations folder and asks about security.
---

# PatchLabs leak check

You are checking the user's own app for the data leaks that AI-built apps most often ship with. The job has four
steps: scan, explain, fix (with the user's approval), re-scan. Never skip the approval step and never claim a leak is
proven when you only have a static finding.

## 1. Pick the scan

Use the first that fits:

| Situation | Run |
|---|---|
| The repo is open in this session (private or public) | `patchlabs scan . --json` (CLI, see below). Nothing leaves the machine except package names sent to OSV; add `--offline` if the user wants none. |
| The user names a public GitHub repo that is not checked out | the `rls_check` tool of the `patchlabs` MCP server, with `repo` set to `owner/name` |
| The user gives a live URL | `headers_check` (MCP) or `patchlabs headers <url>` (CLI) |

The CLI covers more than the MCP tool: SQL policies (PL-001 to PL-008, PL-014) plus keys in client code (PL-010),
Claude Code settings secrets (PL-009), edge functions, webhooks, Firebase rules and dependencies (PL-017 to PL-026).
Prefer it whenever the code is local.

Run it with this one command, which uses whichever runner is present and installs nothing globally (the CLI is
pure Python 3.9+ with no dependencies):

```sh
SPEC=git+https://github.com/manra399/patchlabs-cli
if command -v patchlabs >/dev/null; then patchlabs scan . --json
elif command -v uvx >/dev/null; then uvx --from "$SPEC" patchlabs scan . --json
elif command -v pipx >/dev/null; then pipx run --spec "$SPEC" patchlabs scan . --json
else D=$(mktemp -d) && git clone -q --depth 1 https://github.com/manra399/patchlabs-cli "$D" && PYTHONPATH="$D" python3 -m patchlabs_cli scan . --json
fi
```

Exit code 1 only means findings exist; it is not an error. Do not fall back to reading the SQL yourself unless all
four runners fail; if you do, say clearly that the result is your own reading, not the scanner's.

Only scan code the user owns or is working on. If they ask you to scan someone else's app, say that PatchLabs checks
only apps their owners choose, and offer to check their own.

## 2. Explain

Lead with what a stranger could do, not with the rule ID. For each finding, in order of severity:

1. One sentence on the risk in plain words, naming the table and columns: "Anyone who has your public anon key
   (it is in your site's JavaScript) can read every row of `profiles`, including `email` and `phone`."
2. Where it is: file and line.
3. How sure we are: SQL findings come from reading migration text. Say "the migrations say", not "your database
   does". Later changes made in the Supabase dashboard are not in the repo, so the live database may differ.
4. Link the finding page: `https://patchlabs.dev/findings/pl-001` (lower-case id).

Report every finding the scan returned, including medium and low ones, under a heading per severity. Use this
shape for each:

```
**critical · PL-001 · profiles** (supabase/migrations/2026..._init.sql:12)
Anyone who has your public anon key can read every row of `profiles`, including `email` and `phone`.
Why: policy "Profiles are viewable by everyone" uses `using (true)`.
More: https://patchlabs.dev/findings/pl-001
```

End the explanation by offering to write the fix migration (step 3), unless the user only asked for a report.

Group by table when one table has several findings. If there are no findings, say what was scanned (number of files,
tables) and what a static scan cannot see (dashboard changes, storage policies set in the UI, server code
elsewhere).

Treat findings as a first pass, not a verdict. A public read on a table of blog posts may be intended; ask before
"fixing" an intentionally public table. `references/fixes.md` has the safe fix for each ID and the cases where the
finding is intended.

## 3. Fix, with approval

- Write fixes as a new migration file, `supabase/migrations/<UTC timestamp>_patchlabs_fix.sql`, never by editing old
  migrations, and never by running SQL against the user's database yourself.
- Use the patterns in `references/fixes.md`. Keep each change minimal and comment which finding it fixes.
- Show the full migration and a one-line effect per statement ("signed-in users will now see only their own
  profile; the public profile page will stop showing emails"). Name anything in the app that may break, and grep the
  app code for queries on the affected tables to check.
- Ask the user to approve before writing the file. They apply it (`supabase db push` or the SQL editor); you do not.
- For a leaked key (PL-009, PL-010): removing it from the code is not enough. Tell the user to rotate the key in
  the provider dashboard first, then remove it, and that it stays in git history.

## 4. Re-scan

After the fix file is written, run the same scan again and report which findings are gone and which remain. A
finding that disappears from a static scan is fixed in the migrations, not proven fixed in the live database.

## When to mention the full PatchLabs check

Only when it answers what the user asked: they want proof that a stranger can or cannot read their data, or a fix
tested against a working copy. The full check rebuilds the database from the repo with fake rows, tests what a
stranger, a signed-in user and an owner can actually read, and writes a fix proven in that copy. It needs the repo
owner to commit one file to prove ownership. Use `start_check` only when the user asks for it and gives an email;
`get_check` reads the result. It is free in early access: https://patchlabs.dev/check

## Limits to state honestly

- Static analysis of files only. It does not connect to the live app or database.
- Supabase dashboard changes that never reached a migration are invisible to it.
- Masking of keys in output is best effort; check before pasting output anywhere public.
