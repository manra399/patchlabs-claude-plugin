# PatchLabs for Claude Code

A Claude Code plugin that checks apps built with Lovable, Supabase, Bolt, Replit or Cursor for personal-data leaks,
explains each finding in plain English, and drafts a fix migration for you to approve.

It bundles two things:

- **Skill `patchlabs-leak-check`**: tells Claude how to scan (the free [patchlabs-cli](https://github.com/manra399/patchlabs-cli)
  for a repo on your machine, the PatchLabs MCP server for a public repo or a live site), how to explain findings
  without overstating them, how to write the fix as a new migration you approve, and to re-scan afterwards.
- **MCP server `patchlabs`** at `https://api.patchlabs.dev/mcp/public` (no key): `rls_check`, `headers_check`,
  `start_check`, `get_check`.

## Install

In Claude Code:

```
/plugin marketplace add manra399/patchlabs-claude-plugin
/plugin install patchlabs@patchlabs
```

Then ask, in a repo with a `supabase/migrations` folder:

> Does this app leak user data?

or, for any public repo:

> Run a PatchLabs RLS check on manra399/patchlabs-demo-app

The local scan needs Python 3.9+ and `uvx` (or `pipx`); nothing else is installed.

## Use the skill elsewhere

The skill is a plain [Agent Skill](https://agentskills.io) folder: `plugins/patchlabs/skills/patchlabs-leak-check/`.
Copy it into `~/.claude/skills/` (Claude Code), upload it as a zip under Settings, Capabilities, Skills (claude.ai),
or put it wherever your agent loads skills from.

## What it does not do

- It never runs SQL against your database. Fixes are written as a migration file, shown to you first, and applied
  by you.
- The scans are static: they read files. They do not prove a leak exists in the live database; the
  [full PatchLabs check](https://patchlabs.dev/check) does that in a rebuilt copy with fake rows.
- It checks apps whose owners ask. It is not for scanning other people's apps.

## Links

- Findings explained: https://patchlabs.dev/findings
- MCP docs: https://patchlabs.dev/docs/mcp
- Plugin docs: https://patchlabs.dev/docs/claude-plugin

Apache-2.0.
