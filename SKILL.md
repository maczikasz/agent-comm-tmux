---
name: agent-comm-tmux
description: Coordinate several coding agents that run in separate tmux sessions. Gives each agent a file inbox and outbox, a message format with status codes, tmux notifications instead of polling, and a heartbeat file that shows whether an agent is still alive. Use when you split work across agent sessions and they must hand tasks to each other without a shared chat.
---

# Agent Communication Protocol

File-based messaging for agent coordination, with tmux notifications.

Each agent owns one inbox file and one outbox file. A sender appends a message to the
receiver's inbox, then sends a notification to the receiver's tmux session. No agent polls,
and no agent needs a shared chat service.

## When to use this

Use it when all of the following are true:

- Two or more agents run at the same time, each in its own tmux session.
- The agents must pass work to each other.
- You want a durable record of every task and every answer.

Do not use it for a single agent. One agent needs no protocol.

## Roster

Name each agent, and give each agent one directory or one area that it owns. This skill uses
`lead` for the coordinator and `worker-a`, `worker-b` for the rest. Replace these names with
your own. Keep one name per agent everywhere: the tmux session name, the inbox file name, the
outbox file name and the heartbeat file name must all match.

| Agent | Owns | Edits code? |
|---|---|---|
| `lead` | Task delegation and progress tracking | Only when you say so |
| `worker-a` | One directory and its tests | Yes, inside that directory |
| `worker-b` | One directory and its tests | Yes, inside that directory |
| `monitor` | Staleness checks over the heartbeat files | No |

## Directory Structure

```
.agents/comm/
├── inbox/
│   ├── lead.md           # the coordinator's inbox (agents write here)
│   ├── worker-a.md       # worker-a's inbox
│   └── worker-b.md       # worker-b's inbox
├── outbox/
│   ├── lead.md           # the coordinator's outbox (tasks delegated)
│   ├── worker-a.md       # worker-a's outbox
│   └── worker-b.md       # worker-b's outbox
├── heartbeat/
│   ├── worker-a.json     # worker-a health status
│   └── worker-b.json     # worker-b health status
└── archive/
    └── completed/        # Completed tasks archive
```

## Message Format

Each message follows this format:

```
## [TIMESTAMP] [STATUS] [FROM] -> [TO]

**Task ID**: unique-task-id
**Priority**: high|medium|low
**Type**: task|query|update|complete|blocked

### Summary
Brief description of the task or update.

### Details
- Detailed information
- Context and requirements
- Files affected

### Acceptance Criteria
- [ ] Criterion 1
- [ ] Criterion 2

---
```

Write the timestamp in ISO 8601 with a `Z` suffix. Keep the task ID stable for the whole life
of the task, because it is the only key that joins the inbox entry, the outbox entry, the
heartbeat file and the archive file.

## Status Codes

| Status | Description |
|--------|-------------|
| `PENDING` | New task, not yet acknowledged |
| `ACK` | Acknowledged, work starting |
| `IN_PROGRESS` | Currently being worked on |
| `BLOCKED` | Blocked, needs help |
| `COMPLETE` | Task finished |
| `FAILED` | Task failed, needs review |

## Tmux Notifications

Agents run in separate tmux sessions. Use tmux to notify an agent of a new message instead of
polling.

### Finding an Agent's Tmux Session

```bash
# List all tmux sessions
tmux list-sessions

# Find session by agent name (sessions are named like "worker-a", "worker-b", etc.)
tmux has-session -t worker-a && echo "worker-a session exists"
```

### Sending a Notification

Use `tmux send-keys` to send a message to another agent's session:

```bash
# Notify worker-a of a new inbox message
tmux send-keys -t worker-a "echo '🔔 NEW MESSAGE IN YOUR INBOX - check .agents/comm/inbox/worker-a.md'" Enter

# Or use display-message for a popup (if supported)
tmux display-message -t worker-a "NEW INBOX MESSAGE"
```

### Make the Keystroke Land

A single `send-keys "text" Enter` call is not reliable against an interactive agent. The agent
reads the text and the newline in one burst, and it can submit an empty prompt. Send the text
and the newline as two calls, with a short pause between them:

```bash
notify_agent() {
    local agent=$1
    local message=$2
    tmux send-keys -t "$agent" -l "🔔 $message"
    sleep 0.3
    tmux send-keys -t "$agent" C-m
}

# Usage
notify_agent worker-a "New task in your inbox"
notify_agent lead "Task complete - check inbox"
```

`-l` sends the text literally, so a message that contains `;` or `$` reaches the agent
unchanged.

### Look at the Pane Before You Send

A stray newline answers whatever question the target pane is showing. When the pane sits on a
confirmation prompt, your newline picks the highlighted option, which can be a destructive one.
Capture the pane first, and send nothing while a prompt is open:

```bash
tmux capture-pane -p -t worker-a | tail -20
```

### Restarting Sessions After Protocol Changes

An agent reads this protocol when its session starts. After you change the protocol, restart
every agent session, or the old rules stay in force for the rest of the run.

## Workflow

### Delegating from the coordinator

1. `lead` writes the task to `outbox/lead.md`
2. `lead` adds the same task to the target agent's `inbox/{agent}.md`
3. `lead` notifies the agent over tmux:
   ```bash
   notify_agent {agent} "NEW TASK IN INBOX - check .agents/comm/inbox/{agent}.md"
   ```
4. The target agent reads its inbox
5. The target agent updates the status, and moves the entry to its outbox

### Reporting back to the coordinator

1. The agent writes the update to `outbox/{agent}.md`
2. The agent adds the update to `inbox/lead.md`
3. The agent notifies `lead` over tmux:
   ```bash
   notify_agent lead "TASK UPDATE - check .agents/comm/inbox/lead.md"
   ```
4. `lead` reads the inbox and coordinates

### When Receiving a Notification

When you see a notification in your tmux session:

```bash
# Check your inbox immediately
cat .agents/comm/inbox/{your-agent-name}.md

# Acknowledge PENDING messages by updating status to ACK
# Begin work and update to IN_PROGRESS
```

## Message Types

### Task
New work to be done. Includes requirements and acceptance criteria.

### Query
Question requiring clarification. Blocking until answered.

### Update
Progress report. Non-blocking, informational.

### Complete
Task finished. Include summary of changes made.

### Blocked
Cannot proceed. Explain blocker and what's needed.

## Evidence Rules

A `COMPLETE` report is only as good as the evidence in it. Require these three things, and
bounce the task back as `BLOCKED` when one of them is missing:

1. The exact command that verified the work, and its exit code or test counts.
2. The files that changed, each with a `path:line` reference.
3. A statement of what was not done, when the task is only partly done.

"The build will now succeed" is not evidence. The exit code of the build is.

## Example Usage

### The coordinator delegates to worker-a:

```markdown
## [2026-03-15T10:00:00Z] PENDING [lead] -> [worker-a]

**Task ID**: add-health-endpoint
**Priority**: high
**Type**: task

### Summary
Add a health endpoint to the API service.

### Details
- Put the handler beside the existing routes
- Follow the pattern of the existing status handler
- Include an integration test

### Acceptance Criteria
- [ ] Endpoint returns 200 with the build version
- [ ] Integration test passes
- [ ] The route is registered in the router

---
```

### worker-a reports complete:

```markdown
## [2026-03-15T11:30:00Z] COMPLETE [worker-a] -> [lead]

**Task ID**: add-health-endpoint
**Priority**: high
**Type**: complete

### Summary
The health endpoint is implemented and tested.

### Details
- Added the handler next to the status handler
- Registered the route
- Added an integration test

### Files Changed
- `src/api/health.ts:1` — new handler
- `src/api/router.ts:42` — route registration
- `test/api/health.test.ts:1` — integration test

### Verification
- `npm test -- health` → 4/4 passed, exit 0

---
```

## Archive Process

When a task is fully complete and acknowledged:

1. Move the message to `archive/completed/{date}-{task-id}.md`
2. Remove it from the inbox and outbox files
3. Keep the archive for reference

## Heartbeat System

Agents write heartbeats to show that they are active. The `monitor` agent reads them.

### Heartbeat Format

Location: `heartbeat/{agent}.json`

```json
{
  "agent": "worker-a",
  "status": "IN_PROGRESS",
  "task_id": "add-health-endpoint",
  "last_heartbeat": "2026-03-15T10:30:00Z",
  "message": "Working on integration tests"
}
```

### Status Values

| Status | Meaning |
|--------|---------|
| `IDLE` | Waiting for tasks |
| `ACK` | Acknowledged, starting work |
| `IN_PROGRESS` | Actively working |
| `BLOCKED` | Cannot proceed |
| `COMPLETE` | Task finished |

### Staleness Thresholds

When no heartbeat arrives inside the threshold, `monitor` alerts `lead`:

- `ACK`: 30 minutes
- `IN_PROGRESS`: 1 hour
- `BLOCKED`: 2 hours

A silent agent is not always a dead agent. An agent that retries against an exhausted API quota
looks identical to an agent that is working. Read the pane before you declare an agent dead.

### Agent Responsibilities

1. Update the heartbeat at task start
2. Refresh it every 15 to 30 minutes while working
3. Update it on every status change
4. Set it to `IDLE` when the task is complete
