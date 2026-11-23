# automem

**A long-term memory for Claude Code that runs itself.**

Claude Code forgets you between conversations. automem gives it a memory that lasts: it saves what matters, recalls it when it's relevant, and tidies itself over time. It works for code and for everything else, like a move, a trip or a budget.

It's four **skills** (instruction packs Claude loads for a specific job) plus a few rules in your **`CLAUDE.md`** (a notes file Claude reads at the start of every conversation). There's no server, no subscription, no database and no extra model to install.

---

## Why automem?

### Why not Claude Code's built-in memory?

By default, Claude Code's auto memory is kept per project folder. What Claude learns in one folder isn't there when you work in another, but real work doesn't stay in one folder. You might need a decision from another project, or plan a trip from a coding folder. You can point every folder at one shared memory, but then everything piles into a single index, and only its first 200 lines are loaded. Claude also saves only when it happens to notice. automem organizes memory by what you're working on, not where. It saves at set moments, like a correction or a decision, and keeps the quote each memory came from.

### Why not one memory file?

The simplest memory is one file Claude reads at the start of every conversation, like `CLAUDE.md`. That works for a few dozen notes, but the whole file loads every time, relevant or not, and Claude can only hold so much text at once. A thousand memories is a novel's worth of text in every conversation, crowding out the task you actually asked for.

### Why not a folder of memory files?

The next idea is separate files, opened only when needed. That hits a chicken-and-egg problem: Claude can't know it needs a memory it hasn't read. An index file helps only if Claude stops to check it, and in the middle of a task it usually doesn't. Say you told Claude months ago that you're allergic to shellfish, and today you ask for a recipe. Nothing prompts it to look. The memory was saved but never used, and nothing tells you. Keyword or embedding search doesn't fix this either, because a recipe request doesn't look like an allergy note.

### Why not a vector memory system?

A vector database can hold far more memories than automem. But it's a second system to install and run, often alongside hooks or extra packages, and not every setup allows those. automem needs nothing beyond Claude itself, or any capable LLM. No hooks, no packages and no services are required. Your memories also stay as plain text files you can read, check and edit yourself. With embeddings, what the system actually remembers is hidden inside numbers.

### What automem does instead

In automem, checking memory is a separate, required step before every reply. A helper holds an **index** with one short line per memory saying *when* it matters, such as `shellfish; allergy; food; recipes`. It checks every line against your message, opens a project's own index when that project comes up, and hands Claude only the memories that match. That solves both problems: no memory is skipped for lack of looking, and only what's relevant is loaded. The index lives in the helper, not in your conversation, at the cost of a short wait each time. Projects you're not working on take one line each, so the index stays small even as memory grows into the thousands.

### Built to scale

automem stays light however much you save. The only part that grows with everything you've saved is the index, at one short line per general memory and one per project. Saving a new memory reads the indexes and the few memories close to it, not the rest. Cleanup covers only general memory and the projects you just used, and opens only what changed since the last cleanup. Expired memories move to an archive that's never read again. Apart from the very first cleanup, no step reads all your memories at once.

### Low token usage

Memory adds little to your token usage. Tokens are the small chunks of text that Claude's usage is measured in. Your conversation receives only the memories that matter to it, and any it already has are named rather than sent again. The helper keeps the index in its own working space, not yours, and re-reads it only when a save or cleanup changes it. It also runs on Claude Haiku, a smaller and cheaper model. A writer starts only when there's something to save, and cleanup runs in the background about every two weeks. Each message costs a short wait, one extra turn in your conversation to receive recall's reply, and a Haiku call that's mostly cached. That's far less than loading your whole memory into every message.

---

## Design principle: describe the idea, trust the model

automem has no code of its own and depends on no outside system. It's plain-language instructions that Claude follows.

- **Ideas, not scripts.** The skills explain *what* the memory system is for and *how* its parts fit together: what's worth saving, what an index entry is for, when to ask you. They leave the judgment to the model rather than prescribing every step.
- **It gets better as models get better.** Rigid rules can get in the way of a smart model. High-level instructions let each new Claude model apply its own intelligence, so the system improves without being rewritten.
- **Fewer patches.** There's no code to break when a tool, an API or a model changes. Improving automem usually means editing a sentence, not shipping a fix.
- **You can audit everything.**
  - Your memories are plain text files, not entries in a database or numbers in a vector index.
  - What you see in the files is exactly what Claude uses, so you can open any memory, check where it came from, and fix or delete it by hand.
- **The idea is portable.** The design of an always-checked index, a strict writer and a background cleaner isn't specific to Claude. Any capable LLM agent that can read files and run helper agents could follow the same instructions. Today's skills are written for Claude Code.

---

## How it works

```
 You type a message
        │
        ▼
 ┌──────────────┐  your message + a short summary  ┌────────────────────┐
 │ Main Claude  │ ───────────────────────────────▶ │  Recall helper     │
 │ conversation │ ◀─────────────────────────────── │  (holds the index) │
 └──────────────┘   only the memories that matter  └────────────────────┘
        │ something worth remembering happened?
        ▼
 ┌──────────────┐
 │ Writer helper│ ── checks the proposal against your exact words, then saves or rejects it
 └──────────────┘
        │ every couple of weeks
        ▼
 ┌──────────────┐
 │ Cleaner      │ ── archives expired memories, fixes duplicates and gaps
 └──────────────┘
```

- **It saves at clear moments.** Saves happen when you correct Claude, state a preference, make a decision, share a lasting fact, or say "remember…". Corrections come first.
- **It only saves what you actually said.**
  - Claude can only *propose* a memory. A separate writer starts from "no" and saves it only if it's your words (not Claude's guess), it will stay true, and it's useful.
  - Every memory keeps the quote it came from.
  - You get a short report of each save, so mistakes are easy to catch.
- **It asks instead of guessing.** If a new fact contradicts an old one and it's not clear which is right, the writer asks you.
- **It cleans itself.**
  - About every two weeks, a background cleaner archives expired memories, merges duplicates and fills index gaps.
  - It only checks what you've been using, so it stays cheap.
- **Nothing disappears by accident.** Old memories are archived, not deleted. Only "forget" deletes.
- **You choose how personal it gets.** At setup you decide whether it may hold sensitive topics like health or finances. Passwords and keys are never saved.

---

## Install

You need [Claude Code](https://claude.com/claude-code).

1. Copy the four folders in `skills/` into `~/.claude/skills/`. (`~` is your home folder.)
2. In Claude Code, say **"Set up the memory system."**
3. Claude asks you one question about sensitive information, then shows its plan:
   - the settings it will change;
   - the rules it will add to `CLAUDE.md`;
   - how it will move over any existing memory files, after backing them up.

   Nothing changes until you say yes.
4. Start a new conversation. Memory is on.

To upgrade, copy the new folders over the old ones and say "upgrade the memory system". To remove it, say "uninstall the memory system". Your memories are never deleted for you.

## Everyday use

It runs on its own. You can also say:
- "Remember that…"
- "Forget…"
- "Archive…"
- "Save what you learned"
- "Clean up my memory"

Memories are plain text files in `~/.claude/memory/`. You can read or edit them anytime. If you edit one by hand, update its `updated:` date so the next cleanup checks it.

## Good to know

- **Short wait:** each reply waits a few seconds while recall runs ("Checking memory…").
- **Instructions, not code:** recall and saving run on instructions, so Claude occasionally skips one.
- **Approval prompts:** automem has mostly been tested with approval prompts off. With Claude Code's default settings, saves may ask for your approval.
- **One computer:** memory stays on your computer. It doesn't sync between machines.
- **Not yet benchmarked:** the design targets thousands of memories, but it hasn't been measured at that size yet.

**Want the details?** [DESIGN.md](DESIGN.md) covers the reasoning behind each design decision, what each part costs, the alternatives that were rejected, and the full list of limitations.

## License

[MIT](LICENSE)
