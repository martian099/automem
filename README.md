# automem

**A long-term memory for Claude Code that runs itself.**

Claude Code forgets you between conversations. automem gives it a memory that lasts: it saves what matters, recalls it when it's relevant, and tidies itself over time. It works for code and for everything else, like a move, a trip or a budget.

It's four **skills** (instruction packs Claude loads for a specific job) plus a few rules in your **`CLAUDE.md`** (a notes file Claude reads at the start of every conversation). There's no server, no subscription, no database and no extra model to install.

---

## Built to scale, and nothing gets skipped

Why not just keep one big notebook of everything Claude should know? Because every memory system faces a dilemma:

- **Load everything, and it doesn't scale.**
  - Claude can only keep a limited amount of text in mind at once, its *working space*.
  - A thousand memories is roughly 100,000 words, as long as a novel, loaded into every message.
  - Your actual task gets crowded out, and Claude follows the notes less closely.
- **Load memories only when needed, and you hit a chicken-and-egg problem.**
  - Claude can't know it needs a memory it hasn't loaded.
  - Say you told Claude months ago that you're allergic to shellfish. Today you ask for a dinner recipe. If nothing points Claude to that memory, you get a shrimp recipe.
  - The memory was saved perfectly and still skipped, and nothing told you.

**automem solves both with an index that Claude always checks but rarely loads.**

1. **Every memory is always seen.**
   - Every memory has a short entry in an index, like the index at the back of a book.
   - Before each reply, a helper goes through the *whole* index, so every memory is considered every time.
2. **Entries say *when* a memory matters, not what it says.**
   - For example: `shellfish; allergy; food; recipes; restaurants`.
   - A recipe question matches the entry, even if you never said "allergy" today.
   - Short cues mean thousands of entries fit where a notebook of a few hundred wouldn't.
3. **Only real matches are loaded.**
   - The helper opens the matching memories, checks they really help, and passes on only those.
   - Your conversation gets the shellfish memory, not the other thousand.
4. **The index never enters your conversation.**
   - A separate helper holds it, so a bigger memory doesn't cost your conversation any working space.
5. **The layout keeps the index lean.**
   - General memories (people, health, tastes, habits) each get their own line, so none is ever hidden.
   - A project with hundreds of memories takes one line until you work on it. Then its own index opens.

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

Memories are plain text files in `~/.claude/memory/`, and you can read them anytime.

## Good to know

- **Short wait:** each reply waits a few seconds while recall runs ("Checking memory…").
- **Instructions, not code:** recall and saving run on instructions, so Claude occasionally skips one.
- **Approval prompts:** automem has mostly been tested with approval prompts off. With Claude Code's default settings, saves may ask for your approval.
- **One computer:** memory stays on your computer. It doesn't sync between machines.
- **Not yet benchmarked:** the design targets thousands of memories, but it hasn't been measured at that size yet.

**Want the details?** [DESIGN.md](DESIGN.md) covers the reasoning behind each design decision, what each part costs, the alternatives that were rejected, and the full list of limitations.

## License

[MIT](LICENSE)
