# agent-comm-tmux

A skill that lets several coding agents coordinate when each one runs in its own tmux session.

Each agent owns an inbox file and an outbox file. A sender appends a message to the receiver's
inbox, then sends a tmux notification to the receiver's session. Nobody polls, and every task
and answer stays on disk.

The protocol covers:

- a message format with a task ID, a priority and a type
- six status codes, from `PENDING` to `COMPLETE`
- tmux notification commands, and the two-step keystroke that makes them land
- a heartbeat file per agent, with staleness thresholds for a monitor agent
- evidence rules that a `COMPLETE` report must satisfy

## Install

Copy the skill into your agent's skills directory. For Claude Code:

```bash
mkdir -p ~/.claude/skills/agent-comm-tmux
cp SKILL.md ~/.claude/skills/agent-comm-tmux/
```

Then create the message directories in the repository that the agents work on:

```bash
mkdir -p .agents/comm/{inbox,outbox,heartbeat,archive/completed}
```

## Adapt it

The skill uses `lead`, `worker-a`, `worker-b` and `monitor` as agent names. Replace them with
your own. One agent needs the same name in four places: its tmux session, its inbox file, its
outbox file and its heartbeat file.

## Read it

[SKILL.md](SKILL.md) holds the whole protocol.
