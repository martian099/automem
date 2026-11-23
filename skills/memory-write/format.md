# Memory format

The single source of truth for the memory layout, where a memory goes, the index files, the memory file format and index line rules. The memory writer uses it on every save, `memory-bootstrap` uses it during migration, and cleanup uses it when reorganizing. Paths are relative to the config dir (`$CLAUDE_CONFIG_DIR` if set, else `~/.claude`).

## Layout

```
<config dir>/CLAUDE.md        always loaded: standing preferences + memory rules (no index import)
<config dir>/memory/
├── index.md                  read by the recall agent, not loaded into the main conversation
├── cleanup-state.md          last cleanup time of global memory, written by memory-cleanup
├── topics/<topic>/<slug>.md  global memories, one per file
├── projects/
│   └── <name>/
│       ├── index.md          read by the recall agent when the project's line in memory/index.md matches
│       ├── cleanup-state.md  last cleanup time of this project (cleaned only in sessions that use it)
│       └── <slug>.md         project memories, one per file
└── _archive/<original path>  archived memories and projects, at their original path (e.g. _archive/projects/<name>/): never listed in any index, never read by recall
```

**Delete vs archive.** "Forget X" deletes the memory. "Archive X", expired memories (archived by cleanup) and originals replaced by a split are moved to `_archive/` instead, never deleted.

Recall runs in a separate recall agent (see `memory-recall`), so index size doesn't cost the main conversation any context. Indexes are never split for size.

A **project** is a time-bound effort the user is working on, such as an apartment search, a scraping experiment, or a design. It isn't tied to a repo or folder, and a single task can involve several projects. Projects that go nowhere are archived. New projects exist only after the user approves them.

## Where a memory goes

Each memory goes to exactly one place:

1. **Standing preference** applies to every reply or task regardless of topic (how to answer, ask, format, tone). It goes in the `## Standing preferences` section of the global `CLAUDE.md`, as one line with the source quote in parentheses, and only after the user approves. It never becomes a memory file.
2. **Project memory** belongs to one active project. It goes to `projects/<name>/<slug>.md`. If it touches several projects, pick the single project it mainly belongs to; there are no cross-links. If no active project fits, it needs a new project, which only the user can approve. If the user declines and doesn't point it elsewhere, it isn't saved.
3. **Global memory** is everything else. It goes to `topics/<topic>/<slug>.md`. Reuse an existing topic when one fits. Otherwise use a short lowercase topic name, such as `people`, `health`, `work`, `daily-life`, `tastes` or `web-scraping`.

Slugs are short, specific and kebab-case: `movie-wind-down.md`, not `habit.md`. One memory per file. Split anything that holds independent facts.

## Index files

`memory/index.md` has exactly these sections:

```markdown
# Memory index

## Global
- topics/<topic>/<slug>.md — <index line>

## Projects
- projects/<name>/ — <project goal; key names; task types>
  - <slug>.md — <index line>     ← pinned project memory (sub-bullet, path relative to the project folder)
```

`projects/<name>/index.md`:

```markdown
# <name> memory index
<one line: the project's goal; started YYYY-MM-DD>

- <slug>.md — <index line>
```

- Every memory file has exactly one index line in its own index: `## Global` for topics, or its project's `index.md`. A pin is an extra line.
- Paths in `memory/index.md` are relative to `memory/`. Paths in a project's `index.md`, and pin paths, are relative to that project's folder.
- **Pins:** a pinned project memory is also a sub-bullet under its project's line in `memory/index.md`. Only the user adds or removes pins. When a memory is renamed or deleted, its pin is updated or removed too.
- Archived projects have no lines anywhere.

## Memory file format

```markdown
---
kind: preference | correction | decision | fact | pattern | lesson
created: YYYY-MM-DD
updated: YYYY-MM-DD
expires: YYYY-MM-DD   # only for time-bound facts with a stated end date; otherwise omit the line
source: "<the user's exact words>" (YYYY-MM-DD)   # add a line per later quote that confirms or changes it; a lesson with no quote: none (conclusion of <the work>) (YYYY-MM-DD)
---
<Fact: complete enough to act on without the original chat: what it is, plus any numbers, names and conditions.>

**Context:** <what changes how the fact should be used: what it replaced, who decided it, conditions and exceptions, what's still uncertain>
**Apply:** <how it should change Claude's work>   ← preferences, corrections and lessons only
```

Length:
- Write for Claude, not a person: fragments are fine.
- **Fact**, **Context** and **Apply** are each at most 50 words. Omit Context or Apply if there's nothing to say.
- **Deletion test:** cut every word whose removal wouldn't change what Claude does next time.
- **Never include:** how the topic came up, conversation narrative, Claude's reasoning, a restatement of the source quote, generic advice, hedges, or anything the user didn't say or confirm.
- Over budget usually means two facts. Split them.

Wording:
- Name everyone: "the user", "Claude", other people by name. Never I, me or you, except inside `source` quotes.
- Use absolute dates.
- Keep the user's terms, numbers and conditions. Shorten, but never change the meaning.
- When a fact is replaced, add `Previously: <old value> (until YYYY-MM-DD)` to Context.

Good:
```
Cedar exports close at 17:20 UTC, moved from 18:40 UTC.
**Context:** Ops requested the change; Finance signed off; the chair told Ops to update the runbook. No effective date was given, so the runbook may still say 18:40.
**Apply:** Use 17:20 UTC for Cedar export cutoffs. If the runbook disagrees or the date matters, flag the effective-date gap and check with Ops.
```

Bad:
```
While asking Claude to tidy some meeting notes, the user shared that Cedar exports
currently close at 18:40 UTC but Ops asked for 17:20, and it seems Finance signed off,
so the new time is probably 17:20, which could be useful for future export configuration.
```
The bad one is narrative ("while asking…"), restates the source, hedges ("seems", "probably"), and ends on generic filler. It also loses the open question the good one keeps.

## Index line rules

Index lines exist to make the right memory get opened, not to summarize it. They are optimized for recall.

1. Write **a few short phrases** (1–4 words each), separated by semicolons. Each phrase names a situation, question or decision where the memory should be opened.
   - **Cut every word that doesn't help retrieval:** possessives, pronouns, articles, "the user", "when", "how to", and filler verbs. "John Doe; height" finds the memory as well as "John Doe's height" or "his height". Before writing a line, ask what can be cut without making the right question less likely to match.
   - **At most 30 words per line,** counting everything after the `—` (not the path). This is a hard limit, and it applies to every line in every index, including project lines and pins.
   - **Project lines stay slim** on purpose: the user makes clear which project they're working on, and the project's own `index.md` is opened once it matches.
   - **A memory line that needs more than 30 words:** if the memory file breaks the file rules (a section over 50 words, or independent facts), split it into several files, each with one focused memory and its own index line, and archive the original. Otherwise never split it; rewrite the line to fit.
   - **A project line over 30 words** in `memory/index.md`: shorten it to the project's goal and key names.
2. Put exact names first: projects, people, tools, IDs.
3. Never state the memory's content or value, and never write a sentence that summarizes it. No synonym lists. A one-word phrase is fine when a name on the line anchors it ("John Doe; height"). A line made only of generic words ("time; schedule") isn't.

Examples:
```
- topics/people/john-doe-height.md — John Doe; height
- topics/daily-life/movie-wind-down.md — movie start time; bedtime; evening schedule
- topics/work/run-naming.md — experiment runs; run prefix; tracker tags
- projects/juniper/retention.md — Juniper; data retention; TTL; cleanup jobs
```

Too long: `— John Doe's height; how tall he is; planning his clothing sizes` (possessives, pronouns, filler verbs, and a guessed use the user never mentioned).
