# automem

**A long-term memory for Claude Code that runs itself.**

Claude Code is Anthropic's AI assistant that works on your own computer. automem gives it a memory that lasts between conversations: it saves what matters, recalls it when it's relevant, and tidies itself over time. It's for anyone who uses Claude Code regularly, whether for code or for everyday things like planning a move, a trip or a budget.

**It's built to scale, and nothing it saves gets lost.** automem is designed to hold thousands of memories without filling up Claude's working space. It also makes sure Claude knows every memory exists. Those are the two places simple memory setups break down.

automem is made of four **skills** (instruction packs Claude loads for a specific job) and a few rules added to your **`CLAUDE.md`** (a notes file Claude reads at the start of every conversation). There's no server, no subscription, no special database and no extra AI model to install. Claude does all the work.

\---

## Built to scale: why not just keep one big notebook?

Every memory system faces a dilemma:

* **Load everything** into every conversation, and it stops scaling once memory gets large.
* **Load memories only when they're needed,** and you hit a chicken-and-egg problem: Claude can't know it needs a memory it hasn't loaded.

automem is designed to solve both at once.

### Side one: the notebook doesn't scale

The obvious way to give Claude a memory is one notebook file with everything in it, loaded into every conversation. That works for the first few dozen notes. Then it stops working.

**Why one notebook breaks.**

* **It fills Claude's working space.** Claude can only keep a limited amount of text in mind at once, called its working space or context. Everything in the notebook takes up room in every conversation, even when it has nothing to do with what you're asking.
* **The numbers add up fast.** A typical memory with its context is around 100 words. A thousand memories is roughly 100,000 words, about as long as a novel, loaded into every single message. That eats a large share of Claude's working space, or all of it. Long before that point, your actual task gets crowded out, and Claude follows the notebook less closely.
* **Searching instead isn't the answer.** Keyword search looks like the way out, but it misses any memory worded differently from your question. Ask about "dinner plans" and it won't find the memory about "restaurant reservations".

### Side two: the chicken-and-egg problem

So the next idea is to keep memories in separate files and have Claude open the right one when it's needed. But **Claude can't open a memory it doesn't know exists.**

To realize a memory is relevant, Claude has to know what's in it. To know what's in it, Claude has to load it. A memory that sits in a file Claude never thinks to open might as well not exist. It was saved, but it never gets used.

This is the most common way AI memory quietly fails. A memory can be saved perfectly and still be skipped in exactly the conversation where it mattered, and nothing tells you it happened. Imagine telling Claude months ago that you're allergic to shellfish, then asking for a dinner recipe today. If nothing in today's conversation points Claude to that memory, you get a shrimp recipe.

### How automem solves both: an index Claude always checks

The fix is an **index**: a list of every memory, like the index at the back of a book. It's built so that every memory is *always seen*, but only the relevant ones are *loaded*.

1. **Every memory has an entry, and every entry is checked on every message.** Before each reply, the recall helper goes through the whole index, not a search result. So it always knows every memory that exists, even ones you haven't mentioned in months. That breaks the chicken-and-egg loop.
2. **Entries say *when* a memory matters, not <i>what</i> it says.** An entry like `shellfish; allergy; food; recipes; restaurants` names the situations where the memory should be opened. The dinner-recipe question matches "recipes" and "food", and the memory gets opened, even though you never said "allergy" today. Because entries are short cues rather than full memories, thousands of them fit where a notebook of a few hundred wouldn't.
3. **The file layout keeps the index lean without hiding anything.**

   * **General memories** (topics such as people, health, tastes and work habits) each get their own line in the main index. Anything can come up in any conversation, so none of them is ever hidden behind another layer.
   * **Project memories** are grouped by project. The main index holds one line per project, and the project's own index is opened as soon as your message touches that project. A project with hundreds of memories costs one line until you work on it, and is fully visible once you do.
4. **Only confirmed matches are loaded.** The recall helper opens each candidate file and checks that it really helps before sending it to your main conversation. Your main conversation gets the shellfish memory, not the other thousand.
5. **No memory is left without an entry.** A memory without an entry would be invisible again, so the writer adds an entry with every save, and cleanup looks for memories the index missed and adds them back.

### What each part costs your main conversation

Every part of the system is designed so that its cost grows with *what's relevant right now*, not with *how much you've saved*.

|Layer|What it costs your main conversation|Why it stays small|
|-|-|-|
|**The rules in `CLAUDE.md`**|The same few hundred words, every conversation|They describe how memory works, not what's in it, so they never grow.|
|**The index**|Nothing|A separate recall helper holds it (decision 3). Your main conversation never sees it.|
|**Index entries**|Nothing|Each entry is a short cue of at most 30 words, not the memory itself. The recall helper can scan thousands of them.|
|**Projects**|Nothing, until you work on one|The main index holds one entry per project. A project's own index is opened only when your message is about that project, so a project with hundreds of memories costs a single entry the rest of the time.|
|**Recall results**|Only the memories that matter|The helper reads each candidate memory to confirm it's useful before sending it. Memories it already sent are only named, not sent again.|
|**Saving**|A short proposal and a short report|The writer runs separately and reads only the index plus the few memories related to the new one.|
|**Cleanup**|A short report|The cleaner runs separately. It checks only topics plus projects used this conversation, gives a closer read only to memories changed since the last cleanup, and compares each one only with memories that share its names or subject. It never compares every memory with every other.|
|**The archive**|Nothing|Expired and replaced memories move out of the index, so they're never scanned again.|

### Alternatives that were considered and rejected

* **Load everything:** this is the notebook problem above.
* **Keyword search:** it misses memories worded differently from the question.
* **Splitting the index into folders, or splitting it once it reaches a size limit:** this adds a chicken-and-egg problem, because Claude can't know it needs a sub-index it hasn't read. A separate recall helper makes splitting unnecessary.
* **A vector database:** it needs an extra service to install and run.
* **A fresh recall helper for every message:** it would re-read the whole index each time. A single helper that stays alive for the conversation reads it once.

### Where the limit is

The ceiling isn't your main conversation's working space. It's the recall helper's working space, which holds the main index for the conversation. Because entries are short cues and projects take up a single entry until they're opened, that index can hold thousands of entries. These are design estimates: automem hasn't yet been benchmarked at that size.

\---

## The problem

If you use Claude Code every day, you've probably run into these:

1. **It forgets you between conversations.** Every new conversation starts from zero. You explain your preferences, your projects and your situation again and again.
2. **The built-in options don't go far enough.**

   * **`CLAUDE.md` is manual.** It holds only what you, or Claude when asked, put in it. All of it loads into every conversation and takes up Claude's **working space**: the limited amount of text Claude can keep in mind at once. The more memory you put there, the less room is left for your actual task.
   * **Auto memory is siloed.** Claude Code's built-in auto memory is stored per code repository (a project's code folder) by default, so what it learns in one doesn't follow you to another. You can point it at one shared folder, but then a single index file loads into every conversation, and only its first 200 lines (about 25 KB) are read.
   * **Nothing checks what gets saved,** in either option.
3. **Life isn't only code.** Facts about people, tastes, health or plans don't belong to any code folder, but they're exactly what an assistant should remember.
4. **Saving is unreliable.** Left to itself, an AI either forgets to save things or saves too much: guesses, one-off details, and its own suggestions dressed up as your preferences.
5. **Memory rots.** Over time memories go out of date, contradict each other, or get saved twice. Nobody tidies them up.
6. **It doesn't scale.** Loading a memory list into every conversation only works while the list is small. Simple keyword search misses memories that are worded differently from your question.
7. **Most memory tools come with baggage.** Many need a paid service, a separate server, a special search database, an AI model running on your own computer, or **hooks** (small programs set to run automatically when certain things happen).
8. **Saved memories get skipped.** Claude can't know it should open a memory it hasn't loaded, so a memory can be saved and then never used. This is the chicken-and-egg problem described above.

automem is designed around these eight problems. The rest of this page explains each design decision: the problem it solves, what it does, and why it was chosen.

\---

## How it works

Three words to know first:

* **Memory:** one small text file holding one fact, preference or decision, along with the words you used when you said it.
* **Index:** a list of every memory, like the index at the back of a book. Each entry is a few short phrases that tell Claude *when* to open that memory. There's one main index, plus one for each project.
* **Agent:** a helper copy of Claude that the main conversation starts for one job. The helpers do the memory work and report back in plain words.

```
 You type a message
        │
        ▼
 ┌──────────────┐  your message + a short summary  ┌────────────────────┐
 │ Main Claude  │ ───────────────────────────────▶ │  Recall agent      │
 │ conversation │ ◀─────────────────────────────── │  (holds the index) │
 └──────────────┘   the memories that matter       └────────────────────┘
        │
        │ something worth remembering happened?
        ▼
 ┌──────────────┐  a proposal, quoting your exact words
 │ Writer agent │ ── checks it, then saves it or rejects it, and reports back
 └──────────────┘
        │ after a save, if 14+ days since the last cleanup
        ▼
 ┌──────────────┐
 │ Cleaner agent│ ── archives expired memories, fixes duplicates and gaps
 └──────────────┘
```

Your main conversation doesn't read or edit memory files itself. It does only two things a helper asks for, and you can see them happen: deleting a file, and adding a line to `CLAUDE.md` that you approved.

|Skill|What it does|
|-|-|
|`memory-bootstrap`|One-time setup: settings, the memory folder, and the `CLAUDE.md` rules. It can also move your old memory files over, upgrade, or uninstall.|
|`memory-recall`|Before every reply, finds the memories relevant to your message.|
|`memory-write`|Turns "this is worth remembering" into a saved memory, through a strict check.|
|`memory-cleanup`|Keeps your memories tidy in the background.|

\---

## Design decisions

Each decision lists the **problem** it solves, **what it does**, and **why** it was chosen.

### 1\. Only skills and plain-text rules: no hooks, servers or extra models

* **Problem:** #7, memory tools come with baggage.
* **What it does:**

  * Everything is plain-text files in Markdown, a simple text format you can open in any editor.
  * Rules in your `CLAUDE.md` tell Claude when to recall and when to save. The skills tell it how.
  * Setup changes only your settings file and `CLAUDE.md`, and only after you approve the plan.
* **Why:** Anyone with Claude Code can install it by copying four folders. Nothing runs outside Claude.
* **Trade-off:** The rules are instructions, not code, so Claude may occasionally skip a recall or a save. A hook that runs on every message would be more reliable, but it means extra software to install and maintain, so hooks were left out on purpose.

### 2\. One memory per file, and an index of short "look here" cues

* **Problem:** #6, it doesn't scale; and #8, saved memories get skipped.
* **What it does:**

  * Each memory is its own small file.
  * The index lists every memory with a few short phrases saying *when* it's useful, never *what* it says. For example, the entry is `John Doe; height`, not "John Doe is 6 feet tall".
* **Why:**

  * **Nothing goes unseen.** Recall reads the whole index rather than searching it, so every entry gets looked at. Whether an entry gets picked still depends on Claude's judgment, which is why entries are kept short and clear.
  * **The real file is always read.** An index entry never states the value, so Claude has to open the memory itself before using it. It never relies on a summary that might be incomplete or out of date.

### 3\. A dedicated recall agent holds the index

* **Problem:** #2 and #6, working space and scale. An index of thousands of entries would crowd out your actual task if it sat in the main conversation.
* **What it does:**

  * **One reading per conversation:** at the start, a helper running Claude Haiku (a smaller, faster and cheaper Claude model) reads the whole index and keeps it for the rest of the conversation.
  * **For each message you type:**

    1. The main conversation sends the helper your message plus a short summary of what's going on.
    2. The helper picks every memory that could matter, including indirect links such as a person, preference or past decision your message touches.
    3. It opens those files to make sure they're useful.
    4. It sends them back in full.
  * **No repeats:** memories already sent earlier in the conversation are only named, not sent again.
* **Why:**

  * **The main conversation stays light.** The index never enters it. The main conversation pays only for a short query and for the memories actually returned.
  * **The limit is the helper's working space, not yours.** That's why the index doesn't need to be split up as it grows.
* **Details:**

  * **No cap:** there's no limit on how many memories come back, so nothing relevant is cut off.
  * **Warnings:** if more than 50 come back, or the index points to a file that no longer exists, you get a one-line warning so a cleanup can fix it.
  * **A short wait:** each reply waits for recall. You'll see "Checking memory…" first, and the answer follows when the helper replies. That's usually several seconds, and longer when many memories are read.

### 4\. Clear rules for *when* to save

* **Problem:** #4, saving is unreliable.
* **What it does:** Your `CLAUDE.md` lists eight moments that trigger a save:

  1. You correct Claude. This has the highest priority.
  2. You state or confirm a preference.
  3. You make a decision.
  4. You share a lasting fact about yourself, your people or a project.
  5. You show the same unstated preference twice or more.
  6. You repeat or contradict a saved memory.
  7. Hard-won work produces a reusable lesson.
  8. You say "remember X", "forget X" or "save what you learned".
* **Why:**

  * **Saving is predictable.** Claude saves at known moments instead of guessing.
  * **Corrections come first,** because repeating a mistake you've already corrected is the most frustrating kind of forgetting.

### 5\. Propose, then check: a strict writer decides what gets saved

* **Problem:** #4 again. An AI that saves freely fills memory with guesses and noise.
* **What it does:**

  * **Claude only proposes.** The main conversation can only suggest a memory.
  * **A separate writer decides.** The writer is a helper running Claude Sonnet, and it works on the proposal while you keep chatting.
  * **It starts from "no".** A proposal is saved only if it passes every check:

|Check|Question it asks|
|-|-|
|Source|Did *you* say or confirm this? For a habit, did you show it at least twice? A lesson can come from work that actually happened. Claude's own suggestion never counts.|
|Faithful|Does it say no more than you said? "This one time" can't become "always".|
|Lasting|Will it still be true after this conversation? A temporary fact needs a stated end date.|
|Not already written down|Is it missing from places Claude can already look, like your files or `CLAUDE.md`?|
|Useful|Would a future answer go wrong without it? Saying "remember X" skips this check, and only this one.|
|Allowed|Does it follow your sensitive-information setting? Passwords and keys are never saved.|

* **Why:**

  * **Memory you can trust.** Every memory records where it came from, usually your exact words with a date, so you can always see why it exists.
  * **Mistakes are easy to catch.** After each save you get a short report of what was saved or changed, and where.

### 6\. Conflicts go to you, not to a guess

* **Problem:** #5, memory rots. New information often overlaps with or contradicts old memories.
* **What it does:** Before saving, the writer reads the related memories it can find through the index.

  * **Already saved:** it isn't saved twice. If you just confirmed it, your new words are added to the existing memory.
  * **More detail:** the new detail updates the existing memory.
  * **A clear correction:** the old value is replaced, and it stays in the file as a "Previously:" note.
  * **Can't tell which is right:** the writer asks you, quoting both statements.
* **Why:** The system never silently picks a winner between two things you said.

### 7\. Three places for a memory: standing preferences, projects and topics

* **Problem:** #2 and #3. Memory tied to code folders doesn't fit how people actually live and work.
* **What it does:** Every memory goes to exactly one place.

  * **Standing preferences** are rules for every reply, such as "keep answers short". They go into your `CLAUDE.md`, but only after you approve the exact line.
  * **Projects** are time-bound efforts, like an apartment search or a product launch.

    * A project is defined by the work, not by a folder, and one conversation can touch several projects.
    * A new project is created only after you say yes. If you say no, the memory isn't saved.
    * A memory that touches several projects goes into the one it mainly belongs to.
  * **Topics** hold everything else, such as people, health, tastes and work habits.
* **Why:**

  * **Facts follow you,** and project details stay grouped together.
  * **The layout is also what keeps the index both complete and lean** (see "Built to scale" above).

    * Every topic memory has its own line in the main index, so it's never hidden.
    * Each project takes one line until your message touches it, and then its full index is opened.

### 8\. Automatic background cleanup, kept cheap and careful

* **Problem:** #5 and #6. Memory goes stale, and checking thousands of memories every time is expensive.
* **What it does:**

  * **When:**

    * After a save, if 14 or more days have passed since the last cleanup (or there has never been one), a background cleaner (Claude Sonnet) starts automatically.
    * You can also ask for a cleanup anytime.
    * If you never save anything, cleanup never starts on its own.
  * **What it fixes:**

    * It archives expired memories.
    * It adds index entries for memories the index missed, which recall would otherwise never find.
    * It removes index entries that point to files that no longer exist.
    * It splits files that grew too long or that hold several unrelated facts.
    * It merges obvious duplicates, updates facts that a newer memory replaced, and fixes formatting mistakes.
    * When something is unclear, it asks you instead of guessing. Unanswered questions come back at the next cleanup.
  * **Only what was used:**

    * Topics are always checked.
    * A project is checked only if it came up in the current conversation. Ask to "clean up all projects" to include the rest.
    * Only memories changed since the last cleanup get a closer read.
    * This keeps cleanup cheap even with hundreds of projects.
  * **Hands off what's fine:** it never rewrites index entries for style. It changes one only when something is obviously missing or the entry is too long.
* **Why:** Upkeep happens without you thinking about it, and nothing quietly rewrites your memory.

### 9\. Archive instead of delete, unless you say "forget"

* **Problem:** Automatic cleanup could throw away something you still wanted.
* **What it does:**

  * **What gets archived:** expired memories, originals replaced by a split, and merged duplicates move to an archive folder instead of being deleted.
  * **The archive is out of the way:** recall never lists or reads it, so it takes up no working space.
  * **Corrections stay put:** a corrected memory stays where it is, with a "Previously:" note.
  * **Only "forget" deletes:** saying "forget X" is the only way to permanently delete a memory file.
* **Why:** Nothing disappears by accident. Deleting is always your choice.

### 10\. Your choice about sensitive information

* **Problem:** Some people want an assistant that remembers their health or finances. Others never want that written down.
* **What it does:**

  * **You choose at setup** whether memory may hold sensitive topics: health, finances, family, immigration status, and other people's private details.
  * **If you say no,** you can still ask Claude to remember one specific item.
  * **Secrets are never saved,** whatever you choose: passwords, API keys, account and card numbers.
* **Why:** You decide how personal your memory gets.

### 11\. Helpers don't trip over each other

* **Problem:** Two helpers editing the same file at once can corrupt it.
* **What it does:**

  * **One at a time:** within a conversation, only one writer runs at a time, and the cleaner never runs alongside a writer.
  * **Fresh reads:** writers and cleaners re-read a file right before editing it.
  * **Full context:** they read indexes in full before deciding, never just a search result.
  * **Hands off two actions:** they're told never to delete files or edit your `CLAUDE.md`, and the settings don't pre-approve either. They hand those two actions to the main conversation, which does them where you can see. Before deleting an archived file, it checks that the archive copy exists.
* **Why:** Edits stay safe, and you always know what changed.

\---

## Install

You need [Claude Code](https://claude.com/claude-code), with access to the Claude Haiku and Sonnet models.

1. Copy the four folders in `skills/` into your Claude skills folder, `\\\~/.claude/skills/` (`\\\~` means your home folder). If you've set `CLAUDE\\\_CONFIG\\\_DIR`, use `$CLAUDE\\\_CONFIG\\\_DIR/skills/`; if you've never heard of it, ignore it.
2. Start Claude Code and say: **"Set up the memory system."**
3. Claude asks whether memory may hold sensitive information, then shows you a plan:

   * **Settings changes:**

     * It turns off the built-in auto memory, so two memory systems don't compete.
     * It lets the helpers read and write your memory folder.
     * It requires your approval for edits to `CLAUDE.md`.
   * **New rules:** the rules it will add to `CLAUDE.md`.
   * **Moving old memories:** if you have existing memory files, including Claude Code's built-in auto memory, it shows how they'll be moved over and backs them up first.

   Nothing changes until you say yes.

4. Start a new conversation. Memory is now active.

**Upgrading:** copy the new skill folders over the old ones and say "upgrade the memory system".

**Removing:** say "uninstall the memory system". Your memory folder is never deleted for you.

\---

## Everyday use

You don't need to do anything: Claude recalls and saves on its own. You can also say:

* **"Remember that…"** saves something explicitly.
* **"Forget…"** permanently deletes a memory.
* **"Archive…"** moves a memory out of use without deleting it.
* **"Save what you learned"** reviews the whole conversation for things worth keeping.
* **"Clean up my memory"** runs a cleanup now.

Memories are plain text files in `\\\~/.claude/memory/` (or `$CLAUDE\\\_CONFIG\\\_DIR/memory/`). You can open and read them anytime.

\---

## Known limitations

* **Approval prompts.** Claude Code treats `\\\~/.claude/` as a protected folder, and settings can't pre-approve changes there.

  * In the default permission mode, each save or cleanup may ask for your approval.
  * In auto mode, Claude Code's safety checker decides.
  * automem has mostly been tested in auto mode.
* **Rules can be skipped.** Recall and saving are driven by instructions, not code, so Claude occasionally misses one. See decision 1.
* **Recall adds a wait** before each reply, and the helpers use a bit more of your Claude usage.
* **Updates mid-conversation.** If a memory changes after recall already returned it, recall only names it rather than resending it, so the main conversation may still have the old version. The save report shows the change, and a new conversation picks it up.
* **One computer, one conversation at a time.** Memory lives on your computer and doesn't sync between machines. There's no lock between two Claude Code windows, so avoid saving from two at once.
* **Helpers have no memory of their own.** Agents started for other tasks don't run recall.
* **Some things are manual:**

  * **Standing preferences:** "forget" doesn't cover them. To change or remove one, edit `CLAUDE.md` yourself.
  * **Restoring an archived memory:** move it back from `memory/\\\_archive/`, and the next cleanup that covers it adds it back to the index.
  * **Archiving a whole project:** move `memory/projects/<name>/` to `memory/\\\_archive/projects/<name>/` and remove its entry from `memory/index.md`.
* **Conversation history isn't covered.** "Forget" removes the memory file, not the text of past conversations, which Claude Code keeps on your computer for a while.
* **Not yet tested at large scale.** The design aims at thousands of memories, but recall quality and speed haven't been benchmarked at that size. Non-interactive runs (`claude -p`, the Agent SDK) are untested.

\---

## License

No license has been chosen yet.

