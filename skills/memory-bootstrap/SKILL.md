---
name: memory-bootstrap
description: Set up the index-based memory system, or migrate existing memory files into it. Use when the user asks to set up, bootstrap, install, or migrate to the new memory system, or gives a folder of old memory files to convert. Covers settings.json, the memory folder layout, and the global CLAUDE.md. Also use to upgrade an existing setup to a newer version, or to uninstall it. Does not cover day-to-day saving or recall.
---

# Memory bootstrap

Sets up the memory system defined in `memory-write/format.md`, optionally converting the user's existing memory files into it. Three parts, in order: settings, memory files, global CLAUDE.md. Never overwrite or delete an existing file without showing it to the user first.

## Paths

- **Config dir:** `$CLAUDE_CONFIG_DIR` if set, else `~/.claude`. Every path below is relative to it. Check which one applies before writing anything, and tell the user the resolved path.
- **Memory dir:** `<config dir>/memory/`
- **Global CLAUDE.md:** `<config dir>/CLAUDE.md`
- **Settings:** `<config dir>/settings.json`

## Format

The layout, index file templates, memory file format and index line rules live in `<config dir>/skills/memory-write/format.md`. Read it in full before Step 1, and follow it exactly whenever this skill creates or rewrites memory files or indexes. If the `memory-write`, `memory-recall` or `memory-cleanup` skill isn't installed, stop and tell the user to install it first.

## Step 1: Inventory and plan

1. Resolve the paths. Read `settings.json`, the global `CLAUDE.md`, and the memory dir if they exist.
2. If the user gave existing memory files or a folder, read every file in full. If they gave none, look for Claude Code's built-in auto memory (`<config dir>/projects/*/memory/`) and offer it as a migration source. Otherwise plan an empty setup: `memory/index.md` with the title and empty `## Global` and `## Projects` sections, and `memory/cleanup-state.md`.
3. Ask the user whether the memory system should handle sensitive information. Give examples of what counts:
   - **health:** conditions, medications, injuries, test results
   - **finances:** income, balances, investments, debts
   - **family and relationships:** partner, family planning, genetics
   - **legal or immigration status:** visa, citizenship, legal matters
   - **other people's private details:** names, health, contact info

   Explain that secrets (passwords, API keys, tokens, credentials, account or card numbers) are never saved either way. Use the answer to pick the **Sensitive information** line in Step 4.
4. Show the user one plan covering:
   - the settings changes (Step 2)
   - for each existing file: where its content goes (Step 3), including splits, merges, drops, and lines proposed for `CLAUDE.md`
   - the exact `CLAUDE.md` changes (Step 4)
   - the backup location
5. Get an explicit yes before changing anything. Apply only what they approve.

## Step 2: Settings

Use the `update-config` skill for these edits, merging into existing settings and never replacing the file. If that skill isn't available, show the user the exact diff and edit `settings.json` directly after their yes:
- `"autoMemoryEnabled": false`, so the built-in auto memory doesn't run a second memory system.
- `permissions.allow`: Edit and Write for the memory dir only (`<memory dir>/**`), so a background writer agent can save without a prompt; and Read for the memory dir, so the recall agent can read memories without a prompt.
- `permissions.ask`: Edit for the global `CLAUDE.md`, so standing preferences change only with the user's approval.

If `autoMemoryDirectory` is set, tell the user it no longer applies and offer to remove it.

## Step 3: Memory files

1. **Back up first.** Before moving or rewriting anything, copy the folder of existing memories to a backup folder next to it, in the same parent folder: `<folder name>-backup-YYYY-MM-DD/`. Leave the original folder where it is. Never delete the originals during bootstrap. The user removes the backup later.
2. **Classify each fact** in the old files. A single old file often holds several facts; split them. Place each fact following **Where a memory goes** in `format.md`. For migration:
   - Standing preferences become proposed lines for Step 4, not files.
   - List every project the migration would create in the plan. Each one needs the user's yes.
   - **Drop:** session narrative, to-dos, search steps, dead ends, anything derivable from a repo or a quick lookup, and exact duplicates. List every drop in the plan so the user can keep it instead.
3. **Rewrite** each kept fact in the memory file format from `format.md`.
   - Keep any quote of the user's words as `source`. If the old file has none, use `source: migrated from <old path> (YYYY-MM-DD of migration)`. Never invent a quote.
   - Use `created` from the old file's dates if any, otherwise the migration date.
   - Mark facts with a stated end date as time-bound with `expires`. Flag undated volatile facts (balances, plans, appointments) to the user as possibly stale. Don't guess dates.
4. **Build the indexes.** Write a line for every global memory in `## Global`, one line per approved project in `## Projects`, and every project memory in that project's `index.md`, all following the index line rules in `format.md`. Indexes are never split for size.
5. **Old projects the user calls finished or abandoned:** ask whether to migrate each one into `_archive/projects/<name>/` (never indexed) or drop it.
6. **Cleanup state:** write `memory/cleanup-state.md`, and `projects/<name>/cleanup-state.md` for each project, with `last-cleanup: <today, YYYY-MM-DD>`, since the migration just produced a clean store.

## Step 4: Global CLAUDE.md

Show the user the exact diff, then apply it with their approval. Keep every unrelated section of the existing file. If it already has a `## Memory` section from an older system, show both and ask which to keep. Don't merge them silently.

Add or replace these two sections:

````markdown
## Standing preferences

<one line per user-approved standing preference; source quote in parentheses when known>

## Memory

**Recall.** Before answering each message the user types, get the relevant memories from the recall agent, following the `memory-recall` skill. Invoke that skill on the first user message of a session, and again whenever its instructions are no longer in context. Never read `memory/index.md` in the main conversation. If a memory is expired or old, say so rather than presenting it as current.

<Sensitive information line, from the user's Step 1 answer:>
<If yes:> **Sensitive information.** The memory system has full permission to save, recall and use sensitive information (health, finances, family, immigration, other people's details). Never write secrets (passwords, API keys, tokens, credentials, account or card numbers) to memory files or CLAUDE.md.
<If no:> **Sensitive information.** Never save sensitive information (health, finances, family, immigration, other people's details) to memory, unless the user explicitly asks to remember a specific item. Never write secrets (passwords, API keys, tokens, credentials, account or card numbers) to memory files or CLAUDE.md.

**When to save** (main agent only): when any of these happen, invoke the `memory-write` skill. With that skill, the main agent writes a proposal, and a background writer agent started by the skill checks it against the save gate, then saves it or rejects it. Never write memory files directly, except for an action the writer's report explicitly asks the main agent to carry out.
1. The user corrects Claude (highest priority).
2. The user states or confirms a preference.
3. The user makes a decision.
4. The user shares a lasting fact about themselves, their people, or a project, including inside another request.
5. The user shows the same unstated preference twice or more; cite both instances.
6. The user repeats or contradicts a loaded memory; propose an update.
7. Hard-won work yields a reusable lesson; propose the lesson, not the solution or research.
8. The user says "remember X", "forget X", or "save what you learned" (the last means: sweep the session).
````

## Step 5: Verify and report

- Every kept fact from the old files is in exactly one new file or one approved `CLAUDE.md` line.
- Every memory file has an index line, and every index line points to a file that exists.
- `settings.json` parses as valid JSON, and only the keys above changed.
- Report what moved where, what was dropped, what needs the user's decision (stale facts, proposed projects awaiting approval), and the backup path. Tell the user the changes take effect in the next session.

## Upgrade

When the user installs a newer version of these skills over an existing setup:
1. Compare the `## Memory` section of the global `CLAUDE.md` with the Step 4 template, and the settings with Step 2. Show the diff and apply it with the user's approval.
2. Check the memory dir against the layout in `format.md`, for example missing `cleanup-state.md` files or an old archive folder. List what doesn't match, and fix it with the user's approval, backing up first as in Step 3.

## Uninstall

When the user asks to remove the memory system:
1. Show the diff, then remove the `## Memory` section from the global `CLAUDE.md`. Ask whether to keep `## Standing preferences`.
2. Remove the permission rules from Step 2. Ask whether to turn `autoMemoryEnabled` back on.
3. Never delete the memory dir. Tell the user where it is, so they can keep, back up or delete it themselves.
4. Tell the user to delete the four skill folders (`memory-bootstrap`, `memory-write`, `memory-recall`, `memory-cleanup`) from `<config dir>/skills/`.
