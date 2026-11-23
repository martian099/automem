# automem design

This document explains why automem is built the way it is. It covers each design decision, what it solves, what it costs, and what was rejected. For setup and everyday use, see the [README](README.md). For exact behavior, the skills in `skills/` are the source of truth.

## Constraints

- **No hooks, servers, subscriptions, vector databases or local models.**
  - Everything is Markdown: four skills, plus rules in the global `CLAUDE.md`.
  - The trade-off is that the rules are instructions, not code, so a recall or a save can occasionally be skipped. A prompt-submit hook would be more reliable, but it would be one more thing to install and maintain.
- **Describe intent, don't over-prescribe.**
  - The skills state the purpose of each piece and the rules that matter, and leave the judgment to the model.
  - This keeps the system working as models improve. It also avoids patches when tools change.
  - It keeps the design portable to other LLM agents that can read files and delegate to helpers.
  - The exceptions are hard limits where precision matters, which are spelled out exactly: the gate checks, the word limits and the file layout.
- **Must scale to thousands of memories** without growing the main conversation's context.
- **Not tied to code repositories.**
  - Memory covers the user's whole life, not just code.
  - Projects are real efforts, not folders, so recall can retrieve any relevant project's memories from any working folder by walking the indexes.
  - This is the main gap in Claude Code's built-in auto memory, which is scoped to one repository and saves only when Claude decides to.

## Pieces

| Skill | Role | Model |
|---|---|---|
| `memory-recall` | One persistent agent per session. It holds the index and returns the relevant memories for each user message. | Haiku |
| `memory-write` | The main agent writes a proposal, and a background writer gates it, places it, writes it and updates the index. | Sonnet |
| `memory-cleanup` | A background cleaner for expiry, missing or dangling index lines, oversized files and obvious conflicts. | Sonnet |
| `memory-bootstrap` | Setup, migration, upgrade and uninstall. | (main) |

The rules for the layout, the memory format and index lines live in one place, `memory-write/format.md`. The other skills follow it.

```
memory/
├── index.md                  ## Global: one line per topic memory · ## Projects: one line per project
├── topics/<topic>/<slug>.md  global memories
├── projects/<name>/index.md  project index, opened only when the project matches
├── projects/<name>/<slug>.md
└── _archive/                 never indexed, never read by recall
```

## 1. Scaling and the chicken-and-egg problem

There are two ways to fail:
- **Load everything.** Context grows linearly with the store. Claude Code's built-in auto memory, for example, loads a single index and cuts it off at 200 lines.
- **Load on demand.** The model can't decide it needs a memory it hasn't seen, so saved memories go unused without anyone noticing.

**The answer is to see every memory but load only the matches.**
- **Index lines are retrieval cues, not content.** Each one is a few short phrases (at most 30 words, exact names first), and it never states the memory's value. That keeps lines small, and it forces recall to read the real file instead of trusting a summary.
- **The recall agent reads the whole index once per session.** It never uses grep, so no line goes unseen. On each message it matches lines directly and indirectly (a person, preference or past decision involved), reads every candidate in full, keeps only the ones that would change the answer, and returns them verbatim. There's no cap, and a warning appears past 50.
- **Topics are flat; projects are one hop away.** Every topic memory has its own line, because anything can come up in any conversation. A project gets one slim line, because it's usually obvious which project the user is working on. Its own index is opened only on a match.

**What it costs the main conversation:**

| | Cost per message |
|---|---|
| `CLAUDE.md` rules | Fixed. They don't grow with the store. |
| Index | 0. Only the recall agent holds it. |
| Recall | One short query, plus only the memories returned. Memories already sent are listed by path only. |
| Write / cleanup | One short report each. Both run in the background. |
| Archive | 0 |

**The ceiling** is the recall agent's context window, not the main conversation's. That window holds the topic lines, the project lines, and whichever project indexes the session opens. This hasn't been benchmarked at thousands of memories yet.

**Rejected alternatives:**
- **Grep or keyword recall:** it misses memories worded differently from the question.
- **Splitting indexes by folder or size:** it brings back the chicken-and-egg problem one level up.
- **A vector database or embeddings:**
  - It's an extra service to run.
  - More importantly, memory stops being auditable. Embeddings can't be read or hand-edited.
  - In automem, the Markdown files are the single source of truth. The user can view, verify and edit exactly what Claude uses.
- **A fresh recall agent per message:** it re-reads the index on every message and costs more at scale.
- **A blocking recall tool (an MCP server running headless `claude -p`):** it's one more moving part. The accepted cost is about 3 main-agent passes per message instead of 2.

## 2. Saving: propose, then gate

- **Triggers** are listed in `CLAUDE.md`: a correction (highest priority), a preference, a decision, a lasting fact, a pattern seen at least twice, a contradiction of a saved memory, a hard-won lesson, and explicit remember, forget or "save what you learned".
- **The main agent only proposes.** A proposal quotes the user verbatim. The main agent never edits memory files.
- **The writer rejects by default.** A proposal must pass every check:
  - **Source:** the user's words or confirmation; two or more instances for a pattern; real work for a lesson.
  - **Faithful:** it says no more than the quote.
  - **Lasting:** a temporary fact needs a stated end date.
  - **Not derivable:** it can't already be found in a repo or `CLAUDE.md`.
  - **Useful:** a future answer would go wrong without it. "Remember X" skips only this check.
  - **Allowed:** it respects the sensitive-information setting, and secrets are never saved.
- **Conflicts:**
  - **Duplicate:** the new quote is added to the existing memory.
  - **Refinement:** the existing memory is updated.
  - **Clear replacement:** the value is updated, with a `Previously:` note.
  - **Unclear:** the writer asks the user.
- **Placement:** every memory goes to exactly one place.
  - **Standing preference:** a line in `CLAUDE.md`, which the user approves.
  - **Project memory:** new projects need the user's yes.
  - **Topic:** everything else.
- **Every save updates the index line.** An unindexed memory is invisible, so this step is never skipped.
- **The writer reads indexes in full, never by grep,** so it decides with the complete picture.

## 3. Cleanup: cheap and conservative

- **Trigger:** the writer checks after each save. Cleanup starts when the last one was 14 or more days ago. There's no per-message check.
- **Scope:** topics, plus only the projects used this session. "Clean all projects" overrides that.
- **Changed files** are found by the `updated:` frontmatter date, never by modification time, which editors and sync tools change without changing the content. Only changed files get the deeper checks, and each is compared only with neighbors that share names or subject, never all pairs.
- **Fixes:**
  - It archives expired memories.
  - It indexes files the index missed and drops lines that point to missing files.
  - It splits files that break the file rules (more than 50 words per section, or several independent facts).
  - It regenerates index lines over 30 words.
  - It fixes obvious duplicates, stale facts and quote mismatches.
  - Anything unclear goes to the user. Unanswered questions persist to the next run.
- **Index lines are left alone by default.** It never rewrites a passing line for style.
- **Archive vs. delete:** expired memories, split originals and merged duplicates are archived. Only "forget" deletes.

## 4. Safety

- **Helpers never delete or touch `CLAUDE.md`.** Writer and cleaner are told never to delete files or edit `CLAUDE.md`. They hand both actions to the main agent, which checks that an archive copy exists before deleting.
- **No overlapping writers.** Within a session, one writer runs at a time, and the cleaner never runs alongside a writer. Agents re-read a file right before editing it.
- **Sensitive information is opt-in at setup.** Secrets (passwords, keys, tokens, account and card numbers) are never saved.

## Limitations

- **Approval prompts.** Claude Code treats `~/.claude/` as a protected path, and allow rules may not pre-approve writes there. In default mode, saves may prompt for approval. Testing has been mostly in auto mode.
- **Skipped rules.** Instruction-driven triggers occasionally don't fire.
- **Updates mid-session.** A memory updated after recall returned it is listed by path only, so the main conversation may hold the old text until the next session.
- **No cross-session lock.** Avoid saving from two sessions at once. Memory is local to one machine.
- **No memory in subagents.** Subagents don't run recall.
- **Manual chores.** Standing preferences in `CLAUDE.md`, un-archiving, and archiving a whole project are all done by hand.
- **Untested at scale and in non-interactive runs.** Neither thousands of memories nor `claude -p` / Agent SDK runs have been tested.
