# Don't Be Michael: Product and Architecture Report

_Written 2026-10-13 against `main` at `c7a191ad` (app version 0.1.9). Sources: the code in `src/`, the
bundled packs in `resources/`, the design docs in `docs/` and `docs/designs/`, `HIVE.md`, `SPEC.md`,
`TODOS.md`, `CHANGELOG.md` and git history. When this report infers something rather than reading it
directly from code or docs, it says so._

---

## Contents

1. [Executive summary](#1-executive-summary)
2. [What the product is](#2-what-the-product-is)
3. [Who it is for](#3-who-it-is-for)
4. [The problems it solves](#4-the-problems-it-solves)
5. [Origins and history](#5-origins-and-history)
6. [Core concepts (the mental model)](#6-core-concepts-the-mental-model)
7. [The owner's journey, end to end](#7-the-owners-journey-end-to-end)
8. [System architecture](#8-system-architecture)
9. [Subsystems in depth](#9-subsystems-in-depth)
10. [Where data lives](#10-where-data-lives)
11. [Codebase structure](#11-codebase-structure)
12. [User interface structure](#12-user-interface-structure)
13. [Build, test and release](#13-build-test-and-release)
14. [What this build hides, and multi-engine status](#14-what-this-build-hides-and-multi-engine-status)
15. [Why it is built this way: key design decisions](#15-why-it-is-built-this-way-key-design-decisions)
16. [Security model and known limitations](#16-security-model-and-known-limitations)
17. [Roadmap and open work](#17-roadmap-and-open-work)
18. [Assessment: strengths, risks, observations](#18-assessment-strengths-risks-observations)
19. [Where to read next](#19-where-to-read-next)

---

## 1. Executive summary

**Don't Be Michael** is an open-source (MIT) desktop app for macOS and Windows 11 that gives a small
business owner an **office of AI employees** running on their own computer. Each employee is a
real **Claude Code** process with a job (Finance, Customer Support, Sales and so on), its own folder,
its own memory, and optionally its own email inbox. One of them, **Michael**, is the office manager.
The owner talks only to Michael. He routes work to the right teammate, keeps a task board, runs an
hourly standup, and brings the owner only the decisions that need them, on a **Needs you** board.

Under the hood it is an Electron + React + TypeScript app (about 90,000 lines of TypeScript) that:

- spawns each agent as a `claude` CLI process in a pseudo-terminal (`node-pty`);
- coordinates the agents through **the hive**, a folder of plain JSON and Markdown files (per-agent
  inbox and outbox, a task ledger, a roster, an event log) that only the app commits to git;
- observes and governs every agent through **Claude Code lifecycle hooks**, which call back into the
  app over a local socket so it can track status, inject context, and allow or refuse each tool call;
- enforces business rules in code, not just in prompts: who may assign work, which folders each agent
  may open, which mailbox and Claude connectors each agent may use, and how its email may leave
  (send, send on approval, or draft only);
- draws the whole thing as a calm isometric office with characters from *The Office* (US TV show),
  animated by real events.

The product runs on the owner's **existing Claude plan** (Max is recommended) or an Anthropic API
key. It sends no usage data. It grew out of **Munder Difflin**, a developer-oriented agent harness,
and rebuilt that for non-technical owners. Version 0.0.1 shipped on 2026-09-24; 0.1.9 on 2026-10-11.

---

## 2. What the product is

### One line

> An AI office for your small business. Talk to Michael, your office manager, and a team of AI
> employees does the work on your computer.

### The shape

| Aspect | What it is |
|---|---|
| Form factor | Electron desktop app. macOS (Apple Silicon and Intel) and Windows 11 x64. Linux "will follow" (build targets exist). |
| Runtime engine | Claude Code CLI, one interactive process per agent, signed in to the owner's Claude account, or an Anthropic API key. |
| The owner's interface | An office floor (pods per department around Michael's glass office), a **Talk to Michael** box, a **Needs you** board, a **Tasks** kanban, a **Who talks to whom** graph, and per-agent panels (Profile, Access, Messages, Memory, Work). |
| Coordination | A local file-based message bus (the hive) plus a single orchestrator agent (Michael). |
| Where work happens | On the owner's disk: `~/Documents/<Business>/` holds a folder per team member. |
| Cost model | Free, MIT. No paid tier inside the app. The design doc mentions a possible separate Pro plan (support, training, consulting) sold outside the app. |
| Data posture | Local first. No telemetry is sent (`COLLECT_USAGE_STATS = false`). Agents' model calls go to Anthropic through the owner's own Claude Code. |

### What it is not

- Not a chatbot. The owner hands work to an office that keeps working, on schedules, across days.
- Not a cloud service. Agents run only while the app is open and the computer is awake.
- Not a developer tool any more. Git views, the code editor, raw terminals as the main view,
  temporary helper agents and engine pickers are switched off in this build
  (`src/shared/buildFeatures.ts`).

---

## 3. Who it is for

### Primary audience

**Owners of small, often single-person, businesses with no technical background**: restaurants and
food businesses, retail shops, professional services firms, home services trades, small SaaS or
consulting firms, or "something else".

### The wedge, from the design record

`docs/designs/business-mode-office-packs.md` (approved 2026-09-15) is explicit:

- **Demand evidence:** the founder runs an "AI for small business" program for the **Greater Austin
  Asian Chamber of Commerce**: "there is a genuine demand by small businesses to leverage AI to do
  their work as most of them are single person run businesses." The doc also records an honest gap:
  no single named owner had been observed yet, only program-level interest.
- **Who the owners are today:** "some are pure chatgpt people or worse using chatgpt like google,
  some know a bit of tools … some knows claude code but not technical enough to run end to end
  automation like this project can do."
- **The narrowest wedge:** a solo **restaurant or food** owner in that program who **already pays
  for Claude or Claude Code** but cannot wire up end-to-end automation. First job: social posts and
  daily specials (low risk, visible daily, easy to approve).
- **Explicitly not the first target:** people who use ChatGPT like a search engine. They would need a
  Claude plan, an install and trust in agents all at once.

### Secondary audiences

- **Contributors and developers** who want to study or extend a working multi-agent system
  (`CONTRIBUTING.md`, `docs/ARCHITECTURE.md`, and a blog in `blog/` aimed at developers).
- **Developers wanting coding agents** are pointed back to Munder Difflin in the README: "If you
  write code and want a team of coding agents, Munder Difflin is the better fit."

### Assumptions about the user baked into the product

- They never open a terminal. Setup even installs Claude Code for them (**Get Michael ready**).
- They read on a phone-sized card, so asks are short (about 700 characters), bold-led, with options.
- They prefer plain business words: house rule 7 forbids tool names, file names, message ids and
  time zones in anything the owner reads; the owner "prefers text without dashes".
- They want control over money, public actions and deletion, but not over every routine step.

---

## 4. The problems it solves

| Owner's problem | How the product answers it |
|---|---|
| "AI tools do one part of my business; I stitch them together by hand." | A team of specialised agents, one per department, coordinated by one manager. The design record quotes this as the key signal: "not to solve one part of the problem of a business but using multiple agents specialized for each part of the business." |
| "I know Claude Code is powerful but I can't set up automation." | Pick a business type, pick a suggested team, done. Each role comes pre-written (job, routing line, work style, starter schedules) in an **Office Pack**. |
| "I don't want to manage ten bots." | You talk to one person. Only Michael assigns work (enforced by the router), only Michael sends desktop notifications, and only the decisions he cannot settle reach you. |
| "AI makes things up." | House rules in every agent's system prompt: trace every fact to a source, say when you don't know, never invent people, customers or quotes, report outcomes as they are. |
| "I'm scared of an AI emailing my customers or spending money." | Per-agent mailbox with **Can send / Send on approval / Draft only**, enforced by a broker the agent cannot bypass. Spending, deleting, public actions and scope changes go to the owner. Nothing in the mail tools can delete mail. |
| "I don't want every agent reading everything." | Private folders per agent, enforced by Claude Code's OS sandbox (Mac/Linux) and by the app's hook on every file tool call. Connectors (HubSpot, Drive, QuickBooks…) are off until the owner grants them to a specific agent. |
| "Work should happen without me remembering to ask." | Schedules per agent ("Follow up on unpaid invoices, weekdays at 9"), an hourly standup for Michael, and an Executive Admin that keeps the inbox at zero every two hours. |
| "The AI forgets what I told it." | Per-agent memory index, tidied in the background so it improves instead of piling up; owner answers go straight into the asker's memory. Shared company profile and company knowledge (documents, searchable by words and by meaning). |
| "A runaway agent could burn my plan." | A circuit breaker (steer, then constrain, then optionally stop), a hop cap on message ping-pong, and an **office break** that pauses typing into terminals when the Claude plan hits its usage limit. |
| "I need to close the laptop without losing work." | **Closing time**: Michael tells everyone to save and report where each card stands, then the app quits; the office reopens next launch and catches up missed jobs once. |

---

## 5. Origins and history

### Lineage

- **Munder Difflin** (Chaitanya Giri, first commit 2026-05-31): an agent harness for developers. Each
  agent is a real coding CLI in a terminal, rendered as a pixel-art avatar on a 2D floor, coordinated
  through the hive. `SPEC.md` is its original spec (tmux, Pixi.js, "Sims metaphor"); `HIVE.md` is its
  multi-agent design.
- **Don't Be Michael** (Vivek Kumar, repo `agentvivekkumar/dontbemichael`): a fork that keeps the
  core (real CLI processes, the hive) and rebuilds everything the user sees for business owners. It
  keeps its own version line starting at 0.0.1. `LICENSE` retains Munder Difflin's MIT notice.

### Timeline (from `CHANGELOG.md` / `docs/FEATURES.md`)

| Version | Date | Headline |
|---|---|---|
| 0.0.1 | 2026-09-24 | First release: setup by business type with Office Packs, folder per team member, company knowledge, Office/Tasks/Graph views. |
| 0.0.3 | 2026-09-25 | Private folders enforced; company profile; memory that improves; house rules. |
| 0.0.7 | 2026-09-26 | A mailbox per agent (IMAP), Capabilities tab. |
| 0.0.8 | 2026-09-27 | Four-step hire wizard; distinct-job check; only Michael assigns. |
| 0.0.11 | 2026-09-30 | QuickBooks through the Claude account. |
| 0.0.14 | 2026-10-01 | **Studio** redesign: the pixel floor is replaced by an isometric SVG office. |
| 0.0.15 | 2026-10-02 | Claude connectors listed and gated per agent. |
| 0.0.16 | 2026-10-03 | Card lifecycle overhaul (Blocked means waiting on you; Done is the only end). |
| 0.1.1–0.1.3 | 2026-10-05 | Windows 11 x64 beta. |
| 0.1.4 | 2026-10-07 | Send on approval; Michael answers in the app; Get Michael ready. |
| 0.1.8 | 2026-10-11 | Office break at the plan limit; file chips and file tabs. |
| 0.1.9 | 2026-10-11 | Michael alone writes the task board (triage queue); Done is final. |

### People and activity

- 1,964 commits in total. Most from Chaitanya Giri (Munder Difflin era) and Vivek Kumar (Don't Be
  Michael era: ~446 of the commits since 2026-09-15), plus a long tail of outside contributors.
  `CONTRIBUTORS.md` is generated, and a GitHub Action grants a Discord role on merge.
- **Inference:** code comments credit work to "Lane A", "Oscar/#7", "Kevin", "Pam guardrail", "god
  sign-off". These are the cast names the app gives its own agents, which suggests much of the
  codebase was written by a hive of Claude Code agents running in the predecessor app.

---

## 6. Core concepts (the mental model)

| Term | Meaning |
|---|---|
| **Owner** | The human. Never an agent. Talks only to Michael (or to one agent in a 1:1). |
| **Michael** | The office manager. Internal id `god`, `isGod: true`. Always on, always spawned first, works in the business folder. The only agent that assigns work, writes the task board, decides schedule requests, and notifies the owner. Renameable. |
| **Team member** | Any other agent: a Claude Code process with a role, a character from *The Office*, a folder, a memory, optional mailbox, connectors and schedules. |
| **Office Pack** | A JSON template for a business type: the suggested cast, each role's job text and work style, default picks, office hours, and starter schedules. Six ship: core, restaurant-food, retail-shop, pro-services, home-services, saas-consulting. |
| **Role / routing line / work style** | Role is the title ("Finance"). The routing line tells Michael what to send there. The work style is the agent's own instructions, editable in plain words and converted to agent instructions by a hidden Claude call. |
| **The hive** | `<harnessHome>/hive/`, a git repo of plain files: registry, per-agent inbox/outbox/memory, task ledger, triage folder, board, event log. Only the app commits. |
| **Message** | One JSON file with a "speech act": `request`, `inform`, `propose`, `query`, `agree`, `refuse`, `done`. Agents write to their outbox; the router delivers to inboxes. |
| **Ask me / Needs you** | The owner's decision board. Questions live on task cards (`humanQA` entries). The **Needs you** count is the single "something waits on you" signal. |
| **Task card** | An entry in `tasks.json`: To do, Doing, Waiting (on someone named), Blocked (on the owner, only), Done (final). Michael alone writes it. |
| **Triage** | Team members file needs as one write-once JSON file in `hive/triage/`. Michael works them oldest first into cards, answers or asks. |
| **Report card** | A Done card with a `kind: report` entry: a scheduled outcome (weekly money summary) the owner clears with Got it. |
| **Michael's dock** | The conversation above the Talk to Michael box: the owner's questions with live status (Sent, Michael has it, Waiting for you, Answered…) and his replies. |
| **Schedule (mission)** | A job on a clock for one agent, with one or more "when" lines and a focus area. |
| **Capabilities / Access** | Per-agent grants: one watched mailbox, one send-only mailbox, a Sending choice for each, Claude connectors, QuickBooks level, schedules. |
| **Memory** | `memory.md` (an app-maintained index) plus `memory/inbox.md` (agent notes), `memory/procedures/`, `memory/archive.md`. |
| **Company profile / knowledge** | Profile: short facts every agent gets. Knowledge: uploaded documents, searchable by every agent. |
| **1:1 (on hold)** | The owner talks to one agent directly; Michael sends it no work until the 1:1 ends. |
| **Closing time** | Graceful shutdown protocol run through the hive. |
| **Office break** | State entered when the Claude plan hits its usage limit; nothing is typed into Claude terminals until reset. |
| **House rules** | Eight rules in every agent's system prompt (sources, honesty, plain words, board semantics). |

---

## 7. The owner's journey, end to end

### 7.1 Install and setup

1. Download the `.dmg` or `-win-x64-setup.exe` from GitHub releases. The builds are not yet signed
   with an Apple Developer ID or a Microsoft certificate, so the OS warns once.
2. The onboarding wizard (`OnboardingWizard.tsx`) runs these steps:
   `business → details → team → welcome → home → orchestrator → permissions → ready`.
   - **Business:** business name, kind of business (picks the Office Pack), owner, headquarters address.
     Contact details, hours and prices are optional (Settings later).
   - **Team:** the pack's suggested cast, pre-ticked by `defaultPicks`; the office picture fills with a
     pod per picked person.
   - **Home:** the app's own data folder (default `~/HarnessAgents`), distinct from the business folder.
   - **Ready (Get Michael ready):** installs Claude Code with Anthropic's standalone installer (no Node,
     no password) and signs in where the owner can see it, or takes an Anthropic API key, checked with
     Anthropic and stored encrypted. Skipping leaves a card on Needs you.
3. Michael spawns. The picked team starts once he is up, each in its own folder under
   `~/Documents/<Business>/`. Starter schedules from the pack are seeded (for example Pam's "Inbox to
   zero" every 2 hours in office hours, Oscar's Monday weekly money summary).

### 7.2 Daily operation

```
Owner types in "Talk to Michael"
        │  (app files it as a tracked question with an id)
        ▼
Michael (Claude Code)  ── reads roster, board, open owner requests (injected by hooks)
        │ decides: answer himself, or route
        ▼ writes JSON to his outbox: {to: "kelly", act: "request", ...}
Router (main process, ~1.5 s poll) ── delivers to agents/kelly/inbox/
        │
        ▼ renderer sees new inbox mail → queues a nudge → types it when Kelly is idle
Kelly works in ~/Documents/<Business>/Customer Support/ with her tools
        │ needs the owner? files hive/triage/<time>-kelly.json
        │ done? writes {to: "michael", act: "done", ...}
        ▼
Michael turns triage into a Blocked card with one short ask → shows on Needs you
        │
Owner answers on the card ─▶ answer goes into Kelly's memory AND to Michael as a request
        │
Michael routes the follow-up, closes the request; replies to the owner's question in the dock
```

Meanwhile:

- **Schedules** fire messages from `scheduler` into agents' inboxes (weekly, at a time, or every N
  hours within office hours). Michael gets an **hourly ops standup**.
- The **office floor** animates: envelopes fly between pods, the wall clock rings for scheduled runs,
  mail flags rise on mail tool calls, a check bursts when a card reaches Done, a paper plane goes to
  Needs you when Michael asks the owner something.
- **Email**: an agent with a watched mailbox searches, reads, archives, drafts, proposes or sends,
  according to its Sending setting. Proposed emails appear on Needs you to edit and approve.

### 7.3 Hiring later

The **hire wizard** (`AddAgentModal.tsx`) has four steps: **Who** (pick a character), **Job** (copy a
teammate's job, any pack's job, or write a new one), **Role** (work style in plain words; "Suggest
me" drafts one), **Finalize**. Before hiring, a **distinct-job check** (a hidden Haiku call, with rule
fallback) compares the new job to every teammate's; an overlap must be tied to its own mailbox or topic.

### 7.4 End of day

Quitting with agents running starts **closing time** on the floor: Michael broadcasts it, each team
member saves and replies `CLOSING-TIME-ACK` with where its cards stand, Michael updates the board and
sends `CLOSING-TIME-COMPLETE`, and the app quits. The owner can Remind one agent, Close without one, or
Force quit. Next launch, the office reopens, each agent gets an "office open" message, and a missed
schedule runs once (never a backlog).

---

## 8. System architecture

### 8.1 Technology stack

| Layer | Choice |
|---|---|
| Shell | Electron 32, built with `electron-vite`, packaged with `electron-builder` (DMG/zip for Mac, NSIS for Windows, AppImage target for Linux). |
| UI | React 18, TypeScript, zustand for state, i18next (English, Simplified Chinese, Arabic; picker hidden). |
| Office scene | Hand-built isometric SVG in React (`scene/studio/`). The old Pixi.js pixel floor is gone. |
| Terminals | `node-pty` in main; xterm.js 5 (+ fit, unicode11, webgl addons) in the renderer, pooled per PTY. |
| Persistence | Plain JSON/Markdown files (hive, roster, config), SQLite via `better-sqlite3` (window bounds, command history, kv), Electron `safeStorage` for secrets. |
| Email | `imapflow` (IMAP), `nodemailer` (SMTP), `mailparser`. |
| Documents | `fflate` (Office zip containers), `unpdf` (PDF), macOS `textutil` and Vision OCR (`macOcr.ts`). |
| Updates | `electron-updater` against GitHub releases; download in background, install only on user click. |
| Optional / hidden | Monaco and CodeMirror (IDE), `@openai/agents-realtime` (voice Michael), `posthog-node` (analytics, off), `tunnelmole` (public tunnel for webhooks/Slack), Groq (dictation). |
| Tests | Node's built-in `node:test`, ~300 `test/*.test.cjs` files, run by `tools/run-tests.cjs`. |

### 8.2 Process model

```
┌──────────────────────────── Electron app ─────────────────────────────────────┐
│                                                                               │
│  Renderer (React)                         Preload                             │
│  ├─ Studio office scene (SVG)             typed `window.cth` bridge           │
│  ├─ Needs you board, Michael's dock       (~500 methods; fs/git sandboxed)    │
│  ├─ Tasks kanban, Who-talks-to-whom                                           │
│  ├─ Agent panels + pooled xterm.js        ◀──── IPC (contextBridge) ────▶     │
│  └─ useHive: event loop, MD queue drain                                       │
│                                                                               │
│  Main process (Node)                                                          │
│  ├─ PtyManager ─────────────── spawns `claude …` per agent (node-pty)         │
│  ├─ HiveManager ────────────── hive files, router, registry, ledger, git      │
│  ├─ HookServer ─────────────── Unix socket / named pipe for hook payloads     │
│  ├─ IntegrationBroker ──────── 127.0.0.1 HTTP: mail + secrets, token-gated    │
│  ├─ Scheduler, heartbeat, breaker, office break, closing time, safe clearer   │
│  ├─ MemoryTidy, MemoryManager (MemPalace), KnowledgeManager (kg-core)         │
│  ├─ TelemetryCollector (loopback OTLP), cost ledger                           │
│  └─ Updater, webhook server, (Slack, voice: hidden)                           │
└───────────────┬───────────────────────────────▲───────────────────────────────┘
                │ argv, env, --settings file    │ hook frames (JSON + newline)
                ▼                               │
     ┌───────────────────────────────────────────────────────────────┐
     │  One Claude Code process per agent (Michael + team)           │
     │   ├─ hooks + status line → `cth-hook` shim → HookServer socket │
     │   ├─ md-mail MCP server (stdio) → IntegrationBroker (HTTP)     │
     │   ├─ OTel → TelemetryCollector (loopback)                      │
     │   └─ reads/writes: its folder, its hive inbox/outbox/memory    │
     └───────────────────────────────────────────────────────────────┘
```

### 8.3 Two data planes

The original design decision from `SPEC.md` still holds (`docs/ARCHITECTURE.md`):

- **Terminal plane.** Raw PTY bytes. Authentic terminal output for the Work tab, and the only way to
  type into an agent. Each PTY streams over per-id IPC (`pty:data:<id>`).
- **Event plane.** Structured lifecycle events from Claude Code hooks. These drive status, avatars,
  gating, context injection, cost and the office break.

Why both: hooks alone don't give the authentic stream; the PTY alone can't say reliably which tool is
running without fragile parsing.

---

## 9. Subsystems in depth

### 9.1 The hive (multi-agent coordination layer)

`src/main/hive.ts` (5,271 lines), design in `HIVE.md`. Layout under `<harnessHome>/hive/`:

```
hive/
  PROTOCOL.md          agent-facing message contract (written by the app)
  COMMANDS.md          Claude Code command reference for Michael
  registry.json        hive identity: id, name, role, cwd, provider, sessionId, status, onHold, archived
  fleet.json           live snapshot: per-agent tokens, cost, status, last tool, breaker level, backlog
  tasks.json           task ledger, written by Michael only (app guards it)
  board.md             Michael's notes on plans and priorities
  triage/              team members' write-once filings; triage/.done after handling
  log.jsonl            append-only event log (spawns, messages, drops, drains, compactions…)
  cost-ledger.jsonl    per-sample cost rows (untracked from git)
  bin/                 the hook shim and a bundled-node launcher (locked against agent writes)
  agents/<id>/
    identity.md        who I am (refreshed each spawn)
    memory.md          memory index (app writes only)
    memory/inbox.md    candidate notes (agent writes)
    memory/procedures/ step-by-step for repeated jobs
    inbox/ inbox/.done/   messages delivered to me, and handled ones
    outbox/ outbox/.sent/ messages I send; router archives after delivery
    cursor.json
  agents/human/        the owner's sent questions and dock state (app writes only)
```

**Design rules** (from `HIVE.md` and the code):

1. **Single committer.** Only the main process runs git, with retry, backoff and stale-lock cleanup,
   so many agents never corrupt `.git/index.lock`. Agents never call git.
2. **Single writer per file.** Agents write only inside their own folder; the router moves messages.
   `tasks.json` and `board.md` have one scribe: Michael.
3. **One JSON file per message**, written atomically (temp file + rename).
4. **Untrusted ids.** A message id from an agent must match `^[\w.-]+$` or it is replaced, because it
   becomes a filename and is typed into another terminal.

**Message schema** (FIPA-lite): `id, conversation, in_reply_to, from, to, act, subject, body, hops,
requires_reply, needs_human, created_at`, plus `expect_by` and `waiting_on` for Michael's holding
replies to the owner.

**The router** (`routeOnce`, polled about every 1.5 s) for each agent's outbox:

- Parses each JSON file, repairing literal line breaks inside strings. A file still being written
  (younger than 10 s) is retried; an unreadable one is quarantined to `.sent/bad-*` and the sender is
  told once per 10 minutes.
- The sender is always the owning directory (`msg.from = id`), never what the file claims.
- `to: "scheduler"` with a `schedule` object is a **schedule request**, handled by main, not mail.
- **Hop cap 12:** beyond it the message is dropped and Michael is told (anti ping-pong).
- `human`, `god`, `michael` and Michael's display name all resolve to Michael.
- **Only Michael assigns:** a `request`/`propose` from one team member to another (or broadcast) is
  rerouted to Michael, and the sender is told (`shared/handoffRule.ts`). `query`/`inform` pass, so
  teammates can still ask each other for facts.
- Replies to `scheduler`, `heartbeat` or `mail` are dropped quietly.
- Unknown recipients bounce to Michael with an "undeliverable" subject, never vanish.
- Non-Claude engines without an inbox path get a **terminal work order** instead.
- Every routed message is appended to `log.jsonl`, emitted to the renderer (to fly an envelope), and
  committed.

**Note on `HIVE.md` vs current code:** `HIVE.md` describes agents draining their inbox via a `Stop`
hook that returns `decision: block`. The current `hooks.ts` deliberately does **not** block at Stop
("That old path bypassed terminal-draft/HITL safety and could spend credits while a user was
answering a question"). Delivery now happens through the renderer's guarded message queue (9.4) and a
main-process watchdog (`workerWake.ts`).

### 9.2 Spawning an agent

`spawnAgentCore` in `src/main/index.ts` is the single door for every spawn. In order:

1. Waits for the hook channel; refuses to spawn without it (an agent must never run unwatched).
2. Expands `~`; forces Michael's cwd to the business folder.
3. Resolves the provider (only `claude` is offered in this build).
4. If Claude Code is missing or signed out, prints a short explanation in the terminal and raises the
   **Get Michael ready** card; blocked agents are remembered and started once Claude is ready.
5. Optional git worktree isolation (developer feature, not used by business offices).
6. Computes the agent's **connector plan** (which Claude connectors it may load).
7. Calls `hive.ensureAgent`, which:
   - creates the agent's hive folder, `identity.md`, memory index, cursor;
   - upserts `registry.json` (keeping the prior `sessionId` for `--resume`);
   - builds the **system prompt** and passes it with `--append-system-prompt`;
   - writes a **per-agent settings file** to `<harnessHome>/agent-settings/<id>.json` (outside the
     hive, so agents cannot rewrite it) and passes `--settings <file>`;
   - sets env: `AGENT_ID`, `AGENT_NAME`, `HIVE_ROOT`, `AGENT_DIR`, `HIVE_SOCK`, `HIVE_NODE`, OTel
     variables pointing at the loopback collector, MemPalace and knowledge-store paths;
   - with no connector grants: `ENABLE_CLAUDEAI_MCP_SERVERS=false` and `--strict-mcp-config`.
8. Adds Claude flags: permission mode (auto mode, which is `bypassPermissions`, is the default),
   `--resume <sessionId>` when restoring, the model, and `--mcp-config` for the md-mail server when the
   agent has a mailbox.

**What the per-agent settings file contains** (`hookSettings` in `hive.ts`):

- `hooks` for `Stop`, `SubagentStop`, `PreToolUse` (`*`), `PostToolUse` (`*`), `UserPromptSubmit`,
  `Notification`, `SessionStart`, `PreCompact`, `PostCompact`, `StopFailure` (`rate_limit`), all
  running the `cth-hook` shim with the bundled Node.
- A `statusLine` command (the same shim with `--status`) that forwards exact context-window use and
  plan rate-limit windows after every response.
- `sandbox` (Mac/Linux): writable dirs = the agent's folder, its hive folder and the shared palace;
  `denyWrite` on the hook program's folder; `denyRead`/`allowRead` for folder privacy.
- `permissions.additionalDirectories` and `permissions.deny` (folder rules plus denied connectors).
- `autoContinueAtUsageLimit: true`, `theme: 'auto'`, and the owner's own plugins turned off for this
  session only.

Nothing is written to the owner's `~/.claude` or to any repo.

### 9.3 The hook server: the app's control point

`src/main/hooks.ts`. The `cth-hook` shim sends one newline-terminated JSON frame per connection to a
Unix socket at `hive/hooks.sock` (a named pipe on Windows); frames over a size cap are rejected. `handle()` returns the
hook's answer. In order:

1. **Status-line ticks** update the context gauge and the office break's rate-limit view. Pure
   telemetry, returned early.
2. **`StopFailure` with `rate_limit`** starts the office break.
3. **Operator halt** returns `{continue: false}` (stops cleanly instead of killing the PTY).
4. Records the session id (for `--resume` and cost de-duplication).
5. Feeds the **circuit breaker** (repeated identical tool calls, user prompts, compaction windows).
6. **Stop:** notes the turn end; if Michael was handed an owner question and sent nothing, files his
   last words as "From Michael's notes"; notifies (Michael only).
7. **PreToolUse gates.** Any of these can return `permissionDecision: 'deny'` with a reason the agent
   reads and acts on:
   - operator pause or tool gate (`control.ts`);
   - **Claude connectors**: every MCP tool or resource call is refused unless it is a connector the
     owner turned on and gave this agent, or the app's own md-mail. Fails **closed** on error;
   - **QuickBooks**: off, Read only (a fixed allowlist of reads) or Can make changes;
   - **md-mail**: `mailAccess` checks the mailbox, operation, Sending setting, approved proposal or
     standing approval, and forbids forwarding or attaching across mailboxes;
   - **harness guard**: no writing work files into the app's plumbing folder;
   - **`AskUserQuestion` refused** for team members ("nobody answers your terminal; file it in triage");
   - **folder privacy**: every Read/Write/Edit/Glob/Grep-style call is judged against the folder
     layout, on both the path as asked and its real path (symlinks).
8. **Context injection** (`additionalContext`) on `SessionStart`/`UserPromptSubmit`, each sent only
   when new or changed so it stays cheap and cache-friendly:
   - the **team roster** (Michael only), with each member's real access;
   - the **company profile**;
   - the agent's **memory index** (once per session);
   - a **handoff** after a safe clear;
   - the **standing goal / work style**;
   - **company knowledge on/off** notices;
   - for Michael: **open owner requests**, **owner questions in the dock**, **cards to tidy**
     (new triage filings, Blocked with nothing asked);
   - any queued operator **steer**.
9. **Notification:** an idle notice becomes a light, rate-limited desktop line from Michael (at most
   once every 3 hours: "World's best boss, currently underused."); a team member stuck at a real
   terminal prompt is relayed to Michael, who must put it on Needs you.
10. Everything else is forwarded to the renderer as `hive:hookEvent` to drive status and animations.

### 9.4 Typing into terminals: the message queue

`docs/message-queue.md`, `useHive.ts`, `terminalPool.ts`. The CLI has one input line and both the
owner and the app want it, so there is **one gate**: the drain loop. Every automatic message (inbox
nudges, owner messages, terminal work orders) is enqueued per agent; the drain types the front item
only when all of these hold:

- the agent's status is `idle`;
- delivery is not paused and there is no office break (unless the owner released that message);
- the agent is past its boot grace (35 s);
- `isTerminalAutomationSafe`: no unsubmitted owner draft on the prompt, no open picker menu, not exited;
- 4.5 s since the last delivery to that agent.

Automation never erases the owner's text and never closes their menu. Draft detection reads the
xterm buffer, but only ever to *clear* a suspected draft, never to invent one ("the cheap mistake is
the one we allow"). Inbox nudges carry an `inbox-nonempty` precondition so an agent that already
drained its mail is not woken for nothing. A PTY-quiescence fallback flips a `working` agent idle
after 12 s of silence, and `workerWake.ts` covers a throttled background window.

### 9.5 Prompt architecture

Every agent's system prompt (`injectedPrompt`) is assembled once at spawn:

- **Michael:** `michaelInstructions`: who he is, the business briefing from the pack, routing rules,
  when no one fits (offer 4 options to the owner), the Ask me board format (bold first sentence,
  about 700 characters, title under 60 characters), triage, keeping the board accurate, schedule
  requests, the owner's conversation, scheduled runs and report cards, "staying cheap", file
  locations.
- **Team members:** `teamMemberInstructions`: Michael is the only link to the owner; file needs in
  triage; ask teammates only for facts; anything hard to undo, public or costly goes to Michael;
  follow the email Sending setting; report in a few plain sentences.
- Then for everyone: the **house rules**, the folder line, memory rules, company knowledge commands
  (absolute paths, so they work in PowerShell too), and MemPalace when installed.

Two invariants are commented in the code:

- **Prompt-cache invariant.** The system prompt contains nothing volatile (no dates, counters or board
  state), so Anthropic's prompt cache holds across turns. Volatile context goes through hooks.
- **No shell syntax.** Paths are absolute and joined with native separators; no `$VAR`, so the same
  instructions work on Windows.

The work style (each agent's job specifics) arrives through the hook as a `<goal>` block, re-read
each prompt, so editing it in the app takes effect without a restart.

### 9.6 The task board and the owner loop

`tasks.json` cards: `id, title, description, assignee, status (triage|todo|doing|waiting|blocked|done),
waitingOn, nextAt, dependsOn, priority, humanQA[], result, closedBy/closedAt/closedReason`, plus
Slack or webhook origin fields.

Rules (house rule 8, `michaelInstructions`, `shared/cardGuards.ts`, `shared/ownerRequests.ts`):

- **Blocked** means waiting on the owner and only that, with an open question on Ask me.
- **Waiting** names who else it waits on (a customer, a teammate), never its own assignee.
- **To do** with `nextAt` is a parked step; at that time the assignee gets a "Next step due" message,
  and Michael hears once if it is still To do four hours later.
- **Done** is final. Only the owner reopens a card, with a reason. The app restores a Done card an
  agent reopens or deletes, from a private copy it keeps, and tells Michael.
- One open ask per card; questions from one activity go in one numbered ask; unanswered parts come back.
- Scheduled routines get no card; their outputs arrive as **report cards**.
- An owner's answer goes into the raiser's memory and to Michael as a **request**. It stays in his
  "open requests from the owner" (re-injected every turn) until he replies `done` with `in_reply_to`.
- The Tasks view shows cards "With Michael" and flags ones not moved after a working day of office hours.

The **dock** (Talk to Michael) is a separate loop: each owner question gets an id; Michael must answer
`to: "human"` with `in_reply_to`, using `done`, `refuse`, `query`, or `inform` with `expect_by`/
`waiting_on`. A reply naming no question is bounced back to him to resend with the id.

### 9.7 Scheduling

`shared/missions.ts`, `shared/scheduleTimes.ts`, scheduler in `index.ts`:

- A schedule belongs to one agent and has a label (the job), a **focus** (what to concentrate on this
  run) and one or more "when" lines: `every N` (optionally limited to days and hours) or `at HH:MM`
  on given days. Local time, built from calendar fields.
- A run sends an `inform` from `scheduler` into the agent's inbox. During an office break the job still
  lands; only its delivery waits.
- Missed slots after sleep or closing time run **once**, never as a backlog.
- Agents cannot change their schedules. They send a schedule request to `scheduler`; **Michael**
  approves or declines, and asks the owner only when facts can't settle it or it is sensitive.
  Undecided for 12 hours goes to the owner.
- Built-ins: the **hourly ops standup** for Michael (enabled), a context-aware **heartbeat** (seeded
  disabled), and pack **starter jobs** per hire, timed to the pack's office hours.
- `powerMonitor` resume/unlock re-arms everything after sleep.

### 9.8 Memory and knowledge

**Per-agent memory** (`shared/memoryIndex.ts`, `main/memoryTidy.ts`):

- The agent writes at most three short notes per task to `memory/inbox.md` (owner corrections at once).
- Every 30 minutes, if notes are waiting and a trigger is met (10+ notes, 24 h, office idle, index
  over 90% of budget), a **hidden Haiku call with no tools** returns itemised changes (add, update,
  delete, procedure), applied by code. The model never rewrites the whole file, citing "context
  collapse" research (ACE) and Mem0's add/update/delete shape. Old entries go to `archive.md`; nothing
  is forgotten for being unused. Estimated cost: about a cent per agent per day.
- The index is delivered once per session at `SessionStart`, so it caches.

**Semantic memory** (`memory.ts`): optional **MemPalace** CLI. One shared palace; each agent's memory
is mined into its own "wing"; agents can `mempalace search`. Silent no-op if not installed.

**Company profile** (`shared/companyProfile.ts`): short, definite facts (name, owner, address, hours,
time zone, currency…), modelled on schema.org Organization and LocalBusiness. Tax ids and bank details
are deliberately absent because every agent sees them.

**Company knowledge** (`knowledge.ts`, `kg-core.cjs`, `resources/kg.cjs`): the owner uploads Word,
Excel, PowerPoint, PDF and, on a Mac, scans and photos (Vision OCR). `docText.ts` converts them to text
with pure JS. Every agent can search by keyword through the `kg` CLI, and by meaning through MemPalace
when installed. Large Office files over 200 MB of text parts or 5,000 parts are refused up front.

### 9.9 Email

Design docs: `multi-mailbox.md`, `send-on-approval.md`, `shared-mailboxes.md`, `inbox-zero.md`.

```
Agent (Claude Code) ──stdio MCP──▶ md-mail-mcp.cjs ──HTTP+token──▶ IntegrationBroker (127.0.0.1)
                                   (no passwords,                     │ authenticates per-agent token
                                    no rules)                         │ mailAccess() again
                                                                      ▼
                                                              mail.ts (IMAP / SMTP)
                                                              password from safeStorage
```

- Mailboxes are connected once in **Settings → Connections → Mailboxes**: Gmail, Google Workspace,
  iCloud, Yahoo, Zoho or any IMAP with an app password. The login is tested before saving. Microsoft
  365 / Outlook are not supported yet (a custom domain on M365 is detected and explained).
- Tools: `list_mailboxes, search, read, archive, mark_read, mark_junk, draft, propose, send`. **No
  delete.** Archive never targets trash, junk or system folders.
- Per agent: at most one **watched** mailbox (one watcher per mailbox) and one **send-only** mailbox,
  each with **Can send**, **Send on approval** or **Draft only**. New hires start on Draft only.
- **Send on approval:** the agent `propose`s; the proposal lives in the app's own
  `mail-proposals.json` (not in a file agents write); the owner edits, approves, asks for changes or
  stops it on Needs you; the agent then sends exactly the approved version, once, idempotent across
  crashes and double calls.
- **Standing approvals:** after repeated unchanged approvals of one kind of email, the agent may offer
  "send this kind without asking". Each such email is still checked by a separate no-tools model call
  (`standingCheck.ts`); anything that doesn't clearly fit goes to the owner.
- Enforced twice: once in the PreToolUse hook and again in the broker.

### 9.10 Claude connectors and QuickBooks

- `claude mcp list` is read in the background to list every connector on the owner's Claude account
  (HubSpot, Google Drive, Gmail, QuickBooks…). Each is **off** until the owner turns it on, then granted
  per agent on its Access tab.
- At spawn: an agent with no grants starts with no account connectors at all; one with grants gets
  deny rules for everything else. At every call: the hook re-checks (fails closed).
- The owner's own Claude Code MCP servers and plugins never reach the team.
- QuickBooks has its own switch and a Read only / Can make changes level; roles that read the books
  (Oscar) default to on and Read only.

### 9.11 Folder privacy and sandboxing

`shared/folderAccess.ts`:

- `~/Documents/<Business>/` is Michael's folder; each team member's folder sits inside it.
- A team member opens only its own folder. Michael can **read** his team's folders but not change them.
- Enforced by two layers: Claude Code's **OS sandbox** for shell commands (macOS Seatbelt, Linux
  bubblewrap), and **permission deny rules plus the hook** for Claude's file tools.
- On **Windows** Claude Code has no sandbox, so shell commands are not limited; file tools still are.
- The hook program and every agent's settings file are write-locked for all agents, Michael included.

### 9.12 Guardrails

| Guardrail | Mechanism |
|---|---|
| House rules | System-prompt text for every agent (8 rules). |
| Only Michael assigns | Router reroutes peer `request`/`propose` to Michael. |
| Hop cap | Messages beyond 12 hops dropped; Michael told. |
| Circuit breaker (`breaker.ts`) | Inputs: cost and token velocity (from OTel), repeated identical tool calls, no-progress by file mtimes. Ladder: **steer** (message) → **constrain** → **stop**. One level per beat, de-escalates when healthy. `hardStop` is off by default, so it caps at constrained. |
| Operator control (`control.ts`) | Pause, gate a tool, steer (inject context once), halt at the next hook. Mostly hidden in this build. |
| Office break (`officeBreak.ts`) | At the plan's usage limit: nothing typed into Claude terminals; Claude Code's own auto-continue resumes at reset; Michael is told; cut-off members are told to carry on. Persisted across restarts. |
| Safe clearer (`safeClearer.ts`) | An idle agent with a long conversation writes a handoff, then `/clear`s; the handoff arrives at the new session's start; the old conversation can be restored. |
| Auto-compaction | Claude Code's own, with a per-agent window (`AUTO_COMPACT_WINDOW_TOKENS = 300,000`). |
| Closing time (`closingTime.ts`) | Graceful shutdown through the hive (section 7.4). |
| Imported pack cap | A pack from a link or file has every outward capability forced to "ask" at load time. |

### 9.13 Observability and cost

- Every Claude agent is launched with OpenTelemetry pointed at an embedded **loopback OTLP/HTTP-JSON
  collector** (`telemetry.ts`). Only an allowlist of keys is read, so account email and ids never
  persist.
- A transcript reader (`transcript.ts`) is the fallback, with a per-model price table (`pricing.ts`).
- `cost-ledger.jsonl` keeps every sample; `costLifetime.ts` recovers true lifetime cost across app
  restarts (a reset counter otherwise hid 59% of spend in their measurement).
- `fleet.json` gives Michael a live per-agent view of tokens, cost, status, last tool and breaker level.

### 9.14 Hidden Claude calls (background helpers)

`hiddenClaude.ts` runs a short-lived, invisible **interactive** Claude session in a PTY and reads the
answer from its transcript. It deliberately avoids `claude -p` so these calls draw on the owner's
interactive plan quota rather than a separate Agent SDK credit pool. Used by: memory tidy, the
standing-approval check, the distinct-job check, work-style conversion and "Suggest me". Each has a
rules-based fallback, so a sign-in problem never blocks hiring.

### 9.15 Other inbound channels

- **Webhooks** (`webhook.ts`): opt-in endpoints, each with its own secret, constant-time comparison,
  non-enumerable ids, optional approval gating, and a per-request token the caller uses to poll its
  task's status. Exposed through a tunnelmole tunnel.
- **Slack** (`slack.ts`): signed Events API receiver plus reply helper. **Hidden** in this build.
- **Voice Michael** (`realtime*.ts`): OpenAI Realtime speech-to-speech with an ephemeral token minted
  in main. **Hidden** in this build.
- **Deep links** (`dontbemichael://`): shareable hire manifests that pre-fill the hire form but never
  spawn by themselves.

---

## 10. Where data lives

| Location | Contents |
|---|---|
| App data folder (`~/Library/Application Support/dontbemichael` on macOS; Electron `userData`) | `config.json` (all settings, team, missions, capabilities), `integration-secrets.json` (safeStorage-encrypted, mode 0600, written atomically), `mail-proposals.json`, the SQLite store (window bounds, command history, kv), owner dock keys, update log. |
| Harness home (default `~/HarnessAgents`) | `hive/` (section 9.1, a git repo), `roster.json` plus append-only `roster-backups/`, `office.json`, `agent-settings/<id>.json`, the MemPalace palace, the knowledge store, worktrees (developer feature). |
| Business folder (`~/Documents/<Business>/`) | Michael's working folder; one subfolder per team member (`Finance/`, `Customer Support/`…). The owner's real files and the agents' deliverables. |
| Claude's own folder (`~/.claude/projects/…`) | Session transcripts, written by Claude Code itself. The app reads them; it never writes `~/.claude`. |

Durability practices visible throughout: atomic temp-file-and-rename writes, a damaged `config.json`
copied aside before any save, a secrets file that is unreadable is never overwritten, roster backups
never pruned, and a missing office folder shows a "can't find your office" screen instead of silently
recreating an empty office.

---

## 11. Codebase structure

```
src/
  main/        ~35,000 lines   Electron main process: index.ts (7.6k, wiring + ~225 IPC handlers),
                               hive.ts (5.3k), mail.ts, config.ts, hooks.ts, pty.ts, and ~75 modules
                               listed in docs/ARCHITECTURE.md. Most are electron-free for testability.
  shared/      ~12,700 lines   Pure rules shared by main and renderer: missions, mailboxes,
                               mailProposals, folderAccess, claudeConnectors, officePack, teamPlan,
                               hireTemplates, ownerRequests, cardGuards, handoffRule, memoryIndex,
                               agentProvider, buildFeatures…
  preload/     ~1,800 lines    contextBridge → typed `window.cth` API.
  renderer/src ~41,000 lines
    App.tsx                    layout and top-level wiring
    hooks/useHive.ts           the event loop: Michael boot, hook events → status, inbox nudges,
                               MD-queue drain, quiescence fallback, restart-to-apply
    store/                     zustand store, config mirror, roster source, file tabs
    shell/                     TopBar, BottomBar, Needs you board, Michael's dock, dialogs
    components/                ~80 components: OnboardingWizard, AddAgentModal, SettingsModal,
                               CommandCenterPanel (Michael), AgentDetailPanel, CapabilitiesTab,
                               AskMeTab, MailProposalCards, TasksKanban, MemoryGraphPanel,
                               terminalPool, PtyTerminalView, ClosingTimeBar…
    scene/studio/              isometric SVG office (StudioStage, StudioArt, life.tsx animations)
    scene/office/              the cast, idle "break room" lines, props
    design/                    tokens.css / tokens.ts / global.css
    i18n/                      en, zh-CN, ar
    ide/, realtime/, freeflow/ IDE, voice and dictation (hidden)
resources/
  packs/                       six Office Packs (JSON)
  md-mail-mcp.cjs              mail MCP server
  kg.cjs                       company knowledge CLI for agents
  md-slack-reply.cjs           Slack reply helper
  skills/                      bundled read-only skills copied into each agent (date helpers,
                               md-audit, md-fetch-summarize, md-hive-sync, capabilities)
test/                          ~300 node:test files (.cjs), loading TS via test/load-ts.cjs
tools/                         test runner, postinstall fixes for node-pty, studio lab, screenshot tool
docs/                          ARCHITECTURE, FEATURES, message-queue, designs/ (≈30 design docs),
                               the built blog, model-catalog.json and hero.json (fetched at runtime)
branding/                      DESIGN.md (1,600-line design system), logos, fonts, reference screens
blog/                          Eleventy source for the developer blog (~110 posts)
landing-remotion/              Remotion clips for the landing page (Munder Difflin era)
SPEC.md, HIVE.md, MEMORY_GRAPH_SPEC.md, TELEMETRY.md, TODOS.md, CHANGELOG.md, RELEASE*.md
```

---

## 12. User interface structure

- **Top bar:** view tabs (Office, Tasks, Who talks to whom, then any open file tabs), clock pill,
  version and Beta chips, update chips, the **Needs you** button (the one count).
- **Office view:** the studio. A pod per department around Michael's glass office; label cards show
  each person's status and action; Michael's hub card; mailbox tags; animations driven by real events.
  Clicking a pod opens that person's panel.
- **Right column:** the Needs you board when anything waits (it cannot be closed while it does);
  otherwise the selected person's panel.
- **Person panel tabs:** Profile (job, folder, full instructions), Access (Email, Claude connectors,
  On a schedule), Messages (day-grouped history), Memory (readable notes), Work (the live terminal).
  Michael's panel adds Office schedule (every enabled job) and Advanced (Monitor, Activity).
- **Bottom bar:** the Talk to Michael composer (accepts pasted screenshots and files) and Michael's
  dock above it.
- **Tasks view:** a dependency-aware kanban with To do, Doing, Waiting (yellow), Blocked (coral) and
  Done columns; task detail overlay with the card's question history and result.
- **Who talks to whom:** an SVG graph of agents and the messages between them over a time range.
- **Settings:** General (About and Updates), Company profile, Prerequisites, Agents (default model,
  approvals, circuit breaker), Skills, Connections (Mailboxes, Claude connectors, Webhooks), and
  Memory & Knowledge. A Voice section exists but is hidden in this build.
- **Design system "Studio" v2** (`branding/DESIGN.md`): Sora for the interface, JetBrains Mono for
  data, light and dark themes, department colors, a "Struck M" logo. Characters are drawn as
  department-colored avatars with a signature prop each.

---

## 13. Build, test and release

- **Dev:** `npm install` (rebuilds `node-pty` for Electron and patches its Windows ConPTY), `npm run dev`.
- **Checks:** `npm run typecheck` (node and web projects), `npm run test:focused` (all
  `test/*.test.cjs` through `node --test`).
- **CI** (`.github/workflows/ci.yml`): Typecheck (plus a release-link consistency check) and Build on
  macOS, and the test suite on both macOS and Windows; all are required checks on `main`.
- **Release** (`release.yml`): builds a universal Mac DMG and zip and a Windows x64 NSIS installer
  (Windows is a beta whose failure never blocks the Mac release; the Linux AppImage job is commented
  out) and uploads them to GitHub releases with `latest*.yml`. It signs and notarizes Mac builds when
  Apple credentials are present; today they are not, so Mac builds are ad-hoc signed. Release notes
  for the update toast are baked into `build/release-notes.md`.
- **Blog** (`blog.yml`): verifies the committed built blog in `docs/blog` matches its source.
- **Auto-update:** `electron-updater` downloads in the background; installing is always a user click.
  An ad-hoc-signed Mac app cannot update in place, so it falls back to "download the new version".
- **Runtime fetches:** `docs/model-catalog.json` (model list with a baked-in fallback) and
  `docs/hero.json` (a Settings card), both served from GitHub Pages.

---

## 14. What this build hides, and multi-engine status

### Hidden surfaces (`src/shared/buildFeatures.ts`)

Each is a one-line switch; the code behind it is still wired:

| Off | What it hides |
|---|---|
| `SHOW_GIT`, `SHOW_IDE`, `SHOW_TRACES`, `SHOW_DEV_TOOLS`, `SHOW_FOCUS_MODE`, `SHOW_OPEN_TERMINAL` | Developer views: git, code editor, tool traces, Node/git prerequisites, full-window terminal, "open Terminal here". |
| `ALLOW_TEMP_WORKERS` | Michael spawning ephemeral helper agents (an unseen spend). |
| `SHOW_VOICE`, `ALLOW_VOICE_HIRE` | Voice Michael and dictation. |
| `SHOW_SLACK`, `SHOW_ORG_TRIGGER` | Slack and org-to-org messaging. |
| `COLLECT_USAGE_STATS` | Anonymous PostHog analytics (described in `TELEMETRY.md`, sends nothing while off). |
| `SHOW_ENGINE_PICKER`, `SHOW_IMPORT_HIRE`, `SHOW_MCP_DEFAULTS`, `SHOW_READONLY_SERVERS` | Engine choice, hire import, default MCP servers (which Claude Code ignores in `--settings` anyway). |
| `SHOW_CLOSE_AGENT`, `SHOW_AGENT_BRAKES`, `SHOW_DELIVERY_SWITCH`, `SHOW_AUTO_MODE_LABEL` | Developer-style controls on agents and delivery. |
| `SHOW_LANGUAGE_PICKER`, `SHOW_ARABIC_TERMINAL_SWITCH`, `SHOW_SIMPLE_MODE_SWITCH`, `SHOW_OFFICE_THEME` | Language, RTL terminal, "explain simply", TV-show office maps. |
| **On:** `SHOW_MAILBOXES` | Mailboxes and the Email section. |

### Engines

`shared/agentProvider.ts` carries presets for Claude Code, Codex, Gemini CLI, Antigravity, Grok, Kimi,
Qwen, OpenCode, Crush, pi, Copilot and Cursor, each with its own bridge into the hive:

- **hooks bridges** (Codex, Antigravity, Gemini, Grok, pi, OpenCode): small shims that translate the
  CLI's lifecycle events into `cth-hook`-shaped payloads;
- **proxy bridge** (Qwen, Crush): a loopback reverse proxy that watches the CLI's LLM traffic and
  synthesises the same events and cost samples.

But `BUILD_ENGINES = ['claude']`. A 2026-09-24 audit in `TODOS.md` lists 11 gaps before any second
engine can join an office: folder privacy and connector deny reach only Claude; md-mail is passed only
to Claude; background helpers call Claude only; hook frames are not authenticated; and others.

---

## 15. Why it is built this way: key design decisions

1. **Real CLI processes, not an SDK loop.** Each agent is the actual `claude` TUI. The app inherits
   Claude Code's tools, sandbox, permissions, compaction, resume and model updates for free, and the
   owner's existing plan pays for it. The cost is complexity in typing into TUIs safely (section 9.4).
2. **Interactive sessions on the plan quota.** Even background helpers avoid `claude -p`, so usage
   draws on the owner's Claude plan rather than separately billed SDK credit (`hiddenClaude.ts`).
3. **Files as the message bus.** Plain JSON and Markdown in a git repo are inspectable, durable across
   crashes, auditable, need no server, and are something any CLI agent can already read and write.
   A single committer and single writer per file avoid conflicts.
4. **A single orchestrator the owner talks to.** One point of contact matches how a small business
   owner already manages a person, and routing all assignment through Michael prevents agents
   bouncing work between each other. The router enforces it so a misbehaving agent cannot break it.
5. **Hooks as the control point.** Claude Code's hook protocol lets the app allow or deny each tool
   call and inject context without typing into the terminal, race-free, per call. Rules read config
   live, so a revoked grant applies on the next call without a restart.
6. **Enforce in code, instruct in prompts.** Prompts say what to do; the hook, the broker, the sandbox
   and the router make the important rules hold even when a model ignores them (mail sending,
   connectors, folders, assignment). The project is explicit about where this is not yet true
   (section 16).
7. **The agent never holds a credential.** Mail passwords live only in the main-process broker; the
   MCP server the agent talks to has no secrets and no rules.
8. **Prompt-cache discipline.** System prompts are stable for the agent's life; volatile state goes
   through hooks and is sent only when it changes. This is a cost decision for an always-on office.
9. **Memory by itemised edits.** A cheap model proposes add/update/delete changes that code applies,
   never a full rewrite, so memory improves instead of collapsing; nothing is forgotten silently.
10. **Owner-centric language everywhere.** Plain words, no dashes, no ids, short card-sized asks, one
    notification source. Tests enforce some of it (`no-dashes.test.cjs`, `office-words.test.cjs`,
    `no-names-in-job-text.test.cjs`).
11. **Characters from *The Office*.** Each department maps to a fixed character (Oscar is always
    Finance, Dwight always Sales), giving owners a memorable, friendly cast. The name "Don't Be
    Michael" plays on Michael Scott being a famously bad manager. The legal line: not affiliated with
    NBC, *The Office* or Dunder Mifflin.
12. **Hide, don't delete, developer features.** Switches in `buildFeatures.ts` keep the upstream code
    alive (and mergeable from Munder Difflin) while the owner sees a simple product.
13. **Local first, no telemetry.** Trust is the product's barrier to adoption; the analytics code
    exists with a strict allowlist but is off.

---

## 16. Security model and known limitations

### What holds today

- Mail passwords and API keys: encrypted with OS-backed `safeStorage`, held only in main, injected
  upstream by the broker; agents never see them. Per-agent capability tokens for the broker.
- Mail sending rules enforced in two places (hook and broker); approved emails sent exactly as approved.
- Connectors fail closed; the owner's own MCP servers and plugins are excluded.
- Folder privacy on Mac/Linux via the OS sandbox plus deny rules; symlink-aware hook checks.
- The hook program and settings files are locked against every agent.
- Untrusted message ids sanitised; outbox sender taken from the folder, not the file.
- Webhooks: per-endpoint secrets, constant-time compare, non-enumerable.
- Imported packs are capped to "ask" for outward capabilities.

### Gaps the project itself documents (`TODOS.md`, README, CHANGELOG)

| Gap | Effect |
|---|---|
| **Hook events are not authenticated** (issue #61, P2) | Any agent could forge a hook frame naming another agent (for example, overwrite its saved session id or add cost rows). `agents/human` is still inside the writable hive. |
| **Auto mode is `bypassPermissions` by default** and "asks before spending/deleting" is mostly instructed, not enforced (the "Enforcement compiler" item, P1) | Outside email, connectors and folders, outward actions rely on prompts and the house rules. |
| **Nested Claude** (P1) | An agent with at least one connector grant could start its own `claude` from Bash, which would load account connectors without the app's hook. |
| **Windows has no shell sandbox** | Shell commands on Windows are not limited to the agent's folder; only file tools are. |
| **PreToolUse fails open for file tools** if the shim can't reach the socket | Harness-guard and folder checks for file tools are strong but not airtight (MCP calls fail closed). |
| **Unsigned builds** | OS warnings on first open; no in-place auto-update on Mac; macOS may forget folder grants between versions. |
| **No OAuth broker; no Outlook / Microsoft 365** | Mail is IMAP with app passwords; account apps come through Claude connectors. |
| **Always-on host** | Agents run only while the app is open and the computer is awake; smaller Claude plans hit usage limits during the day (the office break). |
| **Single engine** | Everything depends on Claude Code and the owner's Anthropic account. |

---

## 17. Roadmap and open work

From `TODOS.md` (P1 first):

- **Enforcement compiler and a default-deny hook:** compile each agent's capability levels into real
  permissions.
- **Approvals inbox** for outward actions beyond email (posts, spending), with draft-to-post.
- **Stop nested Claude** from reaching account connectors.
- **Sign and notarize** the Mac build (needs an Apple Developer account).
- **Authenticate hook events** with per-agent tokens; move owner state out of the hive.
- Third-party rate-limit backoff and never-twice writes for connected apps.
- Michael turning repeated work into a routine; hire through Michael.
- A second engine (Codex first) and a **backup engine during a plan limit**.
- Parse documents off the main thread; durable pasted screenshots.
- Business-mode items from the original plan: office hours enforcement, attention strip, tray/close
  behaviour, first-success confirmation and morning report, narrow layout.
- Repo hygiene: remove the old project's website files, record demo videos.

---

## 18. Assessment: strengths, risks, observations

### Strengths

- **A clear, opinionated product.** It picks one user (a small business owner already paying for
  Claude), one interaction model (talk to one manager), and removes everything else.
- **Real enforcement where it matters most.** Email, connectors, folders and assignment are enforced
  by code at the hook, broker, sandbox and router, not just requested in prompts.
- **Unusually thorough failure handling.** Nearly every module documents a past failure and the guard
  added for it: torn writes, half-written outbox files, phantom drafts, timers frozen by sleep, reset
  cost counters, broken packs, a moved office folder.
- **Cost-aware design** for an always-on, multi-agent system: prompt-cache invariants, deduplicated
  context injection, cheap background models, plan-quota-friendly hidden sessions, the office break.
- **Testability.** Most rules live in pure, electron-free modules with about 300 tests.
- **Documentation culture.** About 30 design docs, a feature log per release, a large design system,
  and comments that record the owner's decisions with dates.

### Risks and open questions

- **Platform dependency.** The product rides on Claude Code's CLI behaviour (hook schema, TUI,
  settings keys, transcript format, `claude mcp list`). Several comments note probes against specific
  Claude Code versions; an upstream change can break the office.
- **Complexity of driving TUIs.** Typing into live interactive terminals needs a lot of heuristics
  (idle detection, draft detection, boot grace, cooldowns, watchdogs). It works, but it is the most
  fragile layer.
- **Large central files.** `index.ts` (7.6k lines) and `hive.ts` (5.3k lines) mix many concerns; the
  design doc itself rejected extracting a `hive-core` for now as too risky.
- **Security gaps above** (unauthenticated hooks, bypass-permissions default, Windows sandbox) are
  material for a product that touches a business's email and books, even though the most sensitive
  paths are covered.
- **Adoption barriers for the target user:** a paid Claude plan (ideally Max), unsigned installers
  with OS warnings, a computer that must stay on, and IMAP app passwords.
- **Demand validation.** The design record itself notes the evidence is program-level interest in one
  chamber of commerce, with no single named owner yet observed.
- **Trademark proximity.** The branding leans on *The Office* character names and a parody title; the
  repo carries a non-affiliation line.

### Observations

- `HIVE.md` and `SPEC.md` are partly historical (Stop-hook draining, tmux, Pixi.js); the code is the
  source of truth, and `docs/ARCHITECTURE.md` is the most current map.
- Some leftover upstream naming remains (`window.cth`, `munder-difflin-broker/1` user agent,
  `HarnessAgents` default folder, the `landing-remotion` package name).
- The multi-engine and developer machinery is substantial and still maintained even though this build
  hides it, which keeps upstream merges possible but adds code the business product does not use.

---

## 19. Where to read next

| To understand… | Read |
|---|---|
| Every feature, release by release | `docs/FEATURES.md`, `CHANGELOG.md` |
| The module map | `docs/ARCHITECTURE.md` |
| The multi-agent design | `HIVE.md`, then `src/main/hive.ts` (`ensureAgent`, `routeMessage`, `injectedPrompt`) |
| How the app governs agents | `src/main/hooks.ts` (`handle`) |
| How messages reach terminals | `docs/message-queue.md`, `src/renderer/src/hooks/useHive.ts` |
| Why business mode exists, and its decisions | `docs/designs/business-mode-office-packs.md` |
| The task board rules | `docs/designs/card-lifecycle.md`, `src/shared/cardGuards.ts`, `src/shared/ownerRequests.ts` |
| Email | `docs/designs/multi-mailbox.md`, `send-on-approval.md`, `shared-mailboxes.md` |
| Connectors | `docs/designs/claude-connectors.md`, `src/shared/claudeConnectors.ts` |
| A business type's team | `resources/packs/*.json`, `src/shared/officePack.ts` |
| The visual design | `branding/DESIGN.md` |
| What is not done yet | `TODOS.md` |
