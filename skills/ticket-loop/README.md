# ticket-loop — a coding agent you manage from a group chat

An autonomous ticket loop built from plain Claude Code primitives — **no agent
framework, no orchestrator service, nothing to host.** A Claude Code session
works your Linear board: it takes approved tickets, asks clarifying questions in
a Telegram group, implements each ticket in an isolated git worktree, and opens
one reviewable PR per ticket. Anyone on the team — including non-engineers —
reports bugs and features by typing in the group.

The entire system is the two files in this folder:

| File | What it is |
|---|---|
| `SKILL.md` | The orchestrator — a Claude Code skill. Drop into `.claude/skills/ticket-loop/` in your repo and start it with `/loop /ticket-loop`. |
| `telegram.py` | The Telegram bridge — ~180 lines, Python stdlib only. `send` / `poll` / `discover` subcommands wrapping `sendMessage` + long-polled `getUpdates`. |

## The group-chat grammar

| You type in Telegram | What happens |
|---|---|
| `bug: <what's broken>` | Linear issue created (labeled Bug, reporter credited) + a `take it? (go/skip)` proposal |
| `feature: …` / `ticket: …` | Same, labeled Feature / unlabeled |
| `go` (reply to a proposal) | The `agent` label is applied — approved to build |
| `take NIP-123` | Green-light an existing ticket directly |
| Reply to a ❓ question, or `NIP-123 <answer>` | Answer recorded on the ticket; it unblocks |

The agent posts back: ❓ clarifying questions, 🔨 when it starts a build,
✅ with the PR link, ⚠️ on failures.

## Safety model (the part that matters)

- **Nothing is built without an explicit human `go`.** The `agent` label means
  *approved*, and it is only ever applied through the group. A `manual` label
  fences a ticket off entirely.
- **Every change is a PR** into your integration branch — the agent merges nothing.
- **One ticket at a time**, in a fresh isolated worktree per ticket.
- **State lives on the tickets** (labels + comments), not in the session — kill
  the loop and restart cold, nothing is lost. The only local state is the
  Telegram poll offset in a JSON file.

## Setup (~half a day, including the gotchas below)

1. Create a Telegram bot (@BotFather) and a dedicated group; add the bot.
2. **Disable the bot's privacy mode** (@BotFather → `/setprivacy` → Disable, then
   remove + re-add the bot to the group). Without this, the bot cannot see plain
   group messages — they vanish silently.
3. `python3 telegram.py discover` → grab the group's chat id. Set
   `TELEGRAM_BOT_TOKEN` and `AGENT_TELEGRAM_CHAT_ID` in your repo's `.env`.
4. Wire the Linear MCP in Claude Code (or adapt the skill to your tracker), and
   create three labels: `agent`, `agent-blocked`, `manual`.
5. Drop `SKILL.md` into `.claude/skills/ticket-loop/`, put `telegram.py` where
   your scripts live (adjust the paths in SKILL.md), and adjust the issue-key
   regex in `telegram.py` (`NIP-\d+` → your team key).
6. First run supervised: `/ticket-loop --dry-run`, then `/loop /ticket-loop`.

## Things that will bite you

- **Telegram bot privacy mode** (step 2 above) — the silent killer.
- **`getUpdates` retains messages ~24h.** A loop that's off overnight is fine;
  off for a weekend loses messages sent in the gap.
- **People reply to the bot's *latest* message, not the question.** The
  `NIP-123 <answer>` prefix convention is the workhorse; reply-matching is the
  bonus.

From the team at [Niptao](https://niptao.com) — the write-up with the full story
is on [our blog](https://niptao.com/blog/an-engineer-you-manage-from-a-group-chat/).
