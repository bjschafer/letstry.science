+++
title = "Getting organized?"
date = 2026-09-28
draft = true

description = """\
    In this very belated and barely anticipated follow-up to
    [_In Search of Organization_](/blog/in-search-of-organization), I share some
    about what I've done that's worked for both work and personal organization.\
    \
    ...as well as what still needs some improvement.\
"""

[taxonomies]
tags = [
    "organization",
    ]
+++

## Introduction

Once upon a time, in [_In Search of Organization_](../in-search-of-organization), I wrote about my ideal organization system. Interestingly, this was well before I was diagnosed with ADHD. I know roughly what I was thinking, and I was looking _entirely_ for a technical solution to a problem that also requires some amount of mental discipline and effort.
That's not to say though, that having the right tools doesn't make that a whole lot easier!

{{< toc />}}

## Work

For work my holy grail really is the fully converged system. The ability to capture notes and tasks/todos in the same place and manage them in one pane is absolutely killer.

### Practice

The big thing here has been habit stacking. When I'm logging off for the night, I hit a macro
key on my Stream Deck that minimizes my chat window before locking my screen. This way, when I
log in the next morning, I have a chance to cook before getting bombarded by messages from all
the folks over in Europe.

When I first log on for the day, I always sit and drink my coffee. I paired that habit with popping open Obsidian, opening a new daily, and running through some (dopamine-inducing) checkboxes. Simultaneously, I hit another button on my Stream Deck to launch what I call the "Gardener" (more about that below in ##Tooling).

(TODO fill in more details about daily note template)

### Tooling

## Personal

Here, I'm still not fully set and still iterating quite a bit. My needs are quite a bit different from work, I've found out. So trying to find one system that fits all of it is a fool's errand. Whereas at work, I'm pretty much always seated at the same computer, at home
I often just have my phone handy and nothing else.

However, being able to work from multiple devices is a big plus if I can add something from my phone and flesh it out at the computer. Additionally I value collaboration more than at work -- it's handy to be able to put tasks up for both my wife and I to see and grab.

I'm also less keen on fully converged here. It's enough to have a shared Obsidian vault with my wife for writing things down or planning trips, and an entirely separate tool for keeping track of things that need to be done.

### Practice

My biggest challenges here are:

1. A lot of tasks are very open ended. "Go look at the downspout." "Figure out why the hose is leaking."
2. Things that I don't *see* one way or another end up forgotten.
3. I need to reduce friction in capturing a task as much as possible.

With these sorts of challenges in mind, I set out to build something to help handle tasks.

### Tooling

For my three challenges, I built three pieces:

1. Open ended tasks are called "Looks". They're time-bucketed to 15 minutes, and completing a task just asks you to write what you've learned. I can then automatically or manually take that look and turn it into more tasks if need be.
2. A recurring review (1) pings me nightly to do it, and (2) pulls up things stuck on the backlog and asks me if they're still worth doing.
3. The front screen lets you just type, and deal with cleaning it up separately.

---

## 🤖 AI-COLLATED NOTES — DELETE BEFORE PUBLISHING

> **Everything from here to the end of the file is AI-written.** It's a collation of facts from the work vault to write from, not text for the post. Much of the vault's own documentation (gardener command, "Working the Vault", the reorg plan, template wording) was also agent-written, so quoting it directly would put AI text in the post too. Only headings, config values, and counts are verbatim. Employer, internal project/ticket names, internal URLs, and hosted-model names are left out. Topic indexes (MOCs) are infra-flavored, so they're called "topic indexes" here.

### Before vs. after: numbers for a "why I rebuilt it" section

From the reorg plan dated 2026-07-16, written as a vault survey:

- In 2026 you wrote **70 dailies but touched only 15 content notes**. Dailies had become write-only.
- **62 outbound wikilinks across 337 dailies.** The "refactor dailies into notes every morning" habit never actually ran.
- One debugging investigation of about **1,900 lines lived in a single daily** with zero inbound links.
- Of 155 content notes, **80% had no outbound links** and **48% were full orphans**.
- **48% of notes had empty or missing tags**, written in three competing tag syntaxes.
- Topics had 3 or more possible homes (one topic spanned 5 folders), plus junk drawers called `Other/` and `Reference/`. Every filing decision was a coin flip, so things stayed in the dailies.
- The Home dashboard's "Active Projects" query matched 1 of 14 projects, and that one was a stale stub.
- What already worked: the daily template (new on 2026-07-01), the task capture pipeline, and git auto-backup.

Current state, recounted 2026-11-25:

- **412 dailies** since 2022-03-29.
- **184 notes** in one flat `Notes/` folder, plus 7 project folders.
- **1,021 commits** in the vault's git history, starting 2022-11-09.
- **74 dailies** carry a daily gardener digest and **18** carry a weekly one. **29** have an agent work log.
- **47 "promoted to" markers** in the dailies. The September count of 58 likely included matches outside the dailies, so don't read this as markers disappearing.

### Timeline

- **2022-03:** first daily.
- **2022-11:** vault goes into git.
- **2026-07-01:** Morning plan / Wind-down daily template.
- **2026-07-16:** reorg. Folders flattened, tags cut down, topic indexes built, orphans cleared, gardener built. The gardener ran propose-only for two supervised dry runs before it could write.
- **2026-07-30:** gardener gets full autonomy to move notes out of dailies and create tasks.
- **2026-08-17 to 08-31:** the daily's `## Notes` section becomes a "durable inbox". Anything written there gets rescanned until it's filed.
- **2026-09-29:** Dataview removed (no release since April 2025). Every query now uses core Bases or core search. The task statuses were trimmed and `Tasks.md` was rebuilt around a status board.
- **2026-10-28:** prev/next links come back to the daily. The Dataview removal had dropped them.
- **2026-11-06:** the gardener chain moves from OpenCode to omp. OpenCode 2 can't run a slash command headlessly.

### Daily note structure

Headings, in order:

1. **☕ Morning plan**
    - Previous/next daily links. A Templater script writes a static `← [[prev]] · [[next]] →` line and fixes up the neighboring dailies' links.
    - Checkboxes: run the Gardener (Stream Deck); yesterday's loose ends ("still worth doing?"); scan open tasks.
    - **Today's focus** is a bare checkbox. It's deliberately *not* tracked as a task. It's a cue, not a duplicate task list. If a focus item is still open on a later day, the gardener asks once whether to `track`, `drop` or `leave` it.
    - Embedded **Needs attention** view: anything due or scheduled today or earlier.
    - Embedded **Pick up work** panel, now a Bases view: projects marked `working: true` with their checkpoint and resume cue, notes linked from recent dailies, and links to the task board, unfiled captures and recently filed notes.
2. **🌙 Wind-down:** "Wins / done", "Open focus items" (track each one as a real task, or strike it) and "On my mind". Existing tasks get *rescheduled* in place, never copied into the next daily.
3. **🤖 Agent work log:** generated automatically, covered below.
4. **Notes:** the durable inbox. Anything written here is guaranteed to get filed.
5. **Scratch:** disposable. It's allowed to stay in the daily forever.
6. **Tasks:** captured tasks land here.
7. **🌱 Gardener** and **🌿 Weekly Gardener** digests get appended at the bottom.

The written morning routine is "~1 minute": open the daily, press the button, set the focus.

### Capture: "zero filing decisions"

- **Knowledge:** a plain bullet under `## Notes`. It needs no title, tag or destination. "Never ask 'which folder?' at capture time."
- **Tasks:** a Raycast script command called **Add Obsidian Task**. It lives in your personal `raycast-scripts` repo, written in TypeScript and run by Bun.
    - Priority shorthand: `!` / `!!` / `!!!` / `~` become Tasks-plugin priority emoji.
    - The date is a *separate* field, parsed by chrono-node (`tomorrow`, `friday`, `in 3 days`). Splitting it out means "debug friday incident" never turns into a Friday due date.
    - The script appends `- [ ] TODO …` under today's `## Tasks`.
- The Tasks plugin's **global filter is `TODO`**. Only lines containing `TODO` count as tasks. The written rule: a `TODO` checkbox is anything that must outlive today, and a bare checkbox is a today-only checklist. Morning checklists, focus items, gardener Review lines and design checklists stay out of the task views.

### Task states

Custom statuses. All except Bookmark have their own command, and so can take a hotkey:

- **Inbox** `[ ]`: captured, not sorted yet. It was called Todo until 2026-09-29.
- **Next** `[+]`: chosen and ready. It doesn't need a deadline.
- **Waiting** `[w]`: blocked on something named. The Scheduled date is the follow-up date.
- **Later** `[L]`: worth keeping, but not a commitment.
- Also **In Progress** `/`, **Cancelled** `-` and **Bookmark** `b`. Nine unused statuses were dropped in September.

Rules you set for yourself:

- **Scheduled** means when to work on it. **Due** means a real deadline. No fake deadlines.
- Status and date are decided *when choosing work*, not at capture.
- Created, done and cancelled dates are stamped automatically.

`Tasks.md` sections: **Today** (overdue, due or scheduled today or earlier), **Board** (Inbox, Next, Waiting and Later as columns), **Needs a decision**, **Coming up**, **Bookmarks**, **Done today**. "Needs a decision" holds inbox items older than 14 days, "not expired commitments".

### Projects

- One rule for filing: an active or paused project goes in `Projects/<Project>/`, and everything else goes in `Notes/`. "When in doubt, `Notes/`."
- Each project has one **hub note** with `status: active|paused` and `working: true|false`. `working: true` puts it on the daily's Pick up work panel.
- Hub fields are frontmatter properties: `outcome`, `checkpoint` and `resume` (the resume field is a "where was I" cue). They used to be Dataview inline fields, but Bases can only read frontmatter. Sections are Next (1–3 actions), Waiting, Later ("coarse checkpoints, not a second backlog"), Links and Documents. Documents is a Bases view that lists the project folder's other files.
- Finished projects move to an archive folder.
- **`/break-down`** is an agent command. You give it a brain dump and it saves a proposed outcome, checkpoint and up to 3 actions. It creates no real tasks until you approve them. It saves your raw dump verbatim first, so the only copy never lives in chat history.

### The Gardener

- **Trigger:** the Stream Deck button `🌱 Gardener`, placed under a "Daily Note" button. It opens a visible Terminal so you can watch it. The chain is button → `.command` launcher → shell script → `omp -p` running each slash command. It used `opencode run` until November, when OpenCode 2 broke headless commands. The script now picks the model for each pass, and the weekly pass gets a higher reasoning level. The physical button was chosen as the "existing dopamine path". A scheduled job was the alternative.
- **One button runs three things:** the work log, the daily gardener, then the weekly gardener if the last weekly was 7 or more days ago. Folding the weekly in means there's nothing separate to remember.
- **Daily steps:**
    1. Move every `## Notes` capture into a real note. It uses Obsidian's own Note Composer and leaves `↪ promoted to [[Note]]` in the daily.
    2. Sweep daily prose for "need to / don't forget / follow up" and turn it into real tasks.
    3. Fix tag metadata.
    4. File strays.
    5. Link new notes from a topic index and flag orphans.
    6. Task hygiene: working projects with no chosen next action, the same broad focus 3 times in 7 days, waits whose follow-up date has passed, focus items left open, inbox items older than 14 days, duplicate tasks, and idle projects.
- **Output is a digest in today's daily.** "Done" is a receipt. "Review" holds **at most 5 decisions**, "not a generated task avalanche". You tick a line and type `next`, `waiting`, `later`, `break down`, `accept`, `track`, `reschedule <date>`, `pause`, `cancel` or `leave`, and the next run carries it out. A bare tick means "seen", not yes.
- **Finding vs. deciding:** the agent finds stale things and carries out your answers. It never decides on its own whether a stale task or idle project still matters.
- **No state file.** "Last run" is the newest daily that has a digest.
- **Safety net:** Home has an "Unprocessed captures" view that lists anything still sitting under a daily's Notes, independent of the gardener.
- **Weekly pass:** merges tags, re-curates topic indexes (it suggests splitting one past about 25 curated links), and proposes archiving dead stubs. A dead stub is untouched for about 6 months, under about 15 lines, and has at most one link each way. It applies only safe, reversible fixes on its own. Judgment calls show up as an exact `- [ ] garden: …` line. Ticking one is the approval, and it makes the next button press run the weekly pass early.

Bugs you hit, which could make good anecdotes:

- **You answered "pause it" four days in a row and got asked again every day.** The fix was splitting "find" from "apply", and searching all dailies for past answers.
- **The run window skipped part of each day.** Anything written after that morning's run was missed forever. It now includes the boundary day.
- Moving notes started as copy-and-mark. It became a real move (cut plus link) so notes don't exist twice.

### Agent work log

- The same button first runs `/work-log`. It reads that day's coding-agent sessions from three tools and writes a short `🤖 Agent work log` into each past daily that lacks one.
- It covers agent work only, so it doesn't replace your own "Wins / done". Meetings, reviews and chat never show up.

### Plumbing

- **Plugins:** Tasks, Templater, Linter (enforces tag format on save), obsidian-git, Calendar, Advanced URI, Local REST API, Tag Wrangler, Iconize, Pandoc, plus a few editor and callout plugins. Dataview is gone. Queries use core **Bases** (`Browse.base` is a table-view index, `Dashboard.base` feeds Home and the daily) and core search. There's also a homemade plugin that publishes a note to an internal wiki. Leave that out of the post or keep it vague.
- **Git:** auto-commits every 5 minutes and pushes every 6 hours. **Dailies are git-ignored**, so only filed notes are versioned.
- Agents use the Obsidian CLI (`obsidian move`, `obsidian eval`) so wikilinks update when notes move.
- There's a private folder that agents are told never to read.

### Mapping to the 2020 post's wishlist

- ✅ Notes and todos together, with inline metadata (Tasks-plugin emoji for dates and priority).
- ✅ Plain text.
- ✅ Git sync.
- ✅ Tags, not categories.
- ✅ CLI access.
- ✅ Free-form structure.
- ❌ **Mobile.** obsidian-git doesn't run on mobile. The reorg left a mobile/voice capture idea parked for exactly that reason.
- The 2020 post ruled out Obsidian because it had "no task management". The Tasks plugin is what changed that.

### Not covered

- **The Stream Deck chat-minimize macro:** it isn't documented in the vault, so it's unconfirmed.
- **The personal side** (the shared vault with your wife, "Looks", the nightly review): none of that is on the work machine.
