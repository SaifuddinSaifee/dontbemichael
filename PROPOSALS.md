# Don't Be Michael: One Enhancement, One Fix

## 1. Enhancement: answer Needs you from your phone, by email

### Problem

- **The office waits on the owner.** A card is Blocked only while its question waits on the owner, and that work stops until they answer (house rule 8, `hive.ts:233`).
- **The owner can only answer at the desktop.** Notifications are desktop pop-ups only (`hooks.ts:840`, `index.ts:1781`). Both remote paths are off: `/remote-control` (`useHive.ts:673`) and Slack (`SHOW_SLACK = false`).
- **The product already assumes a phone.** Michael's instructions say "The owner reads each ask on a small card, often on a phone" (`hive.ts:329`). The restaurant pack says "The owner works during service" (`restaurant-food.json`).

### Proposal

An opt-in **Needs you by email**. When a question lands on Needs you, the app emails it to the owner's personal address. The owner replies from any mail app, and the reply becomes the card's answer, exactly as if typed in the app.

### How

1. **Setting.** Settings, Connections, Needs you by email: the owner's address, which connected mailbox sends, and an on/off switch.
2. **Outbound.** For each new open question, main sends it with the existing `MailService.send` (`mail.ts:840`). The subject is `[Needs you] <card title> (ref <token>)`. The body holds the question, its options, and "Reply above this line".
3. **Inbound.** A main-process poller runs every 2 minutes. It uses `MailService.search` and `read` (`mail.ts:547`, `mail.ts:705`) to find replies by live token rather than unread status, keeps only the text above the quoted part, and files the answer.
4. **One answer path.** Today the renderer runs the answer steps: write the card's answer, call `hive:rememberOwnerAnswer`, send Michael a request. Move these into one main-process function that both the Ask me card and the email poller call, so the two paths can't drift.
5. **Scope of version 1.** Questions only. Email approvals and schedule requests stay in the app, because they carry more risk.

### Safety

- **Token.** Each question gets a random single-use token, kept in the app's data folder and never in the hive.
- **Accept only proven replies.** A reply counts only when it comes from the owner's address, its token matches an open question, and the mail server's checks (DKIM and DMARC) pass. Anything else is ignored and logged.
- **Agents don't see these emails.** md-mail search hides messages that carry a live token, so an Executive Admin clearing the same inbox doesn't act on them.
- **Late replies.** A reply to a question that was already answered or withdrawn gets a short "already settled" email back.

### Measure

How long cards stay Blocked, how many questions are answered by email, and Blocked time during the pack's office pauses, such as the lunch rush.

## 2. Fix: a granted connector can change or delete anything

### Problem

- **The promise.** "Spending money, deleting things and changes of scope come to you" (README). "Spending money, deleting things and anything public come to you first" (`docs/FEATURES.md`).
- **The grant is all or nothing.** It is a yes or no list (`connectorGranted`, `claudeConnectors.ts:139`). Once a team member holds a connector, `connectorAccess` (`claudeConnectors.ts:235`) allows every tool on it, deletes and updates included.
- **Claude Code doesn't ask either.** Agents run in auto mode, which skips permission prompts, by default (`config.ts:510`).
- **Only two paths are protected.** QuickBooks has Read only or Can make changes (`quickbooks.ts:111`), and the mail tools have no delete. HubSpot, Google Drive, Calendar and every other connector are open for changes once given.
- **The project's own TODO agrees.** TODOS.md (Enforcement compiler, priority 1): "'Asks before sending' is instructed, not enforced."

### Proposal

Give every connector the model QuickBooks already uses. Per team member, each connector is **Read only** (the default) or **Can make changes**. Under Read only, the app refuses any call that isn't clearly a read. The refusal tells the agent to send the change to Michael, who puts it on Needs you.

### How

1. **Data.** `agentCapabilities[id].connectors` changes from a list of names to a map from each connector to `read` or `change`.
2. **Read check.** Add it in `shared/claudeConnectors.ts`. A call is a read only if its tool name starts with a read verb (get, list, search, read, fetch, find, query, lookup, describe), or if it reads an MCP resource. Everything else counts as a change, including names with no known verb. A per-connector list of exceptions covers cases like QuickBooks' `READ_TOOLS`.
3. **Gate.** Add one more check in `connectorAccess`, after the grant check. Read only plus a call that isn't a read means deny, worded like the QuickBooks refusal.
4. **Interface.** Every connector row on the Access tab gets the Read only / Can make changes choice that the QuickBooks row already has.
5. **Migration without surprises.** Existing grants keep Can make changes. One Needs you card lists who can change what and offers "Make Read only" in one click. The 0.0.15 connector upgrade card (`connectorsUpgradeCard`, `index.ts:4797`) did the same kind of migration.
6. **Tests.** Extend `test/claude-connectors.test.cjs`:
   - reads are allowed;
   - `delete_*` and `update_*` calls are refused under Read only;
   - a call with no known verb is refused;
   - Can make changes allows everything.

### Risks

- **A read misjudged as a change.** It is refused, not run, so the mistake is the safe one, and the agent reports it (house rule 6).
- **A nested Claude session started from Bash** still skips the app's checks (TODOS.md, priority 1). That gap is separate and this fix leaves it as it is.

### Measure

No connector change happens without Can make changes. Refused-change counts per connector in `log.jsonl` show where the list of read verbs needs tuning.
