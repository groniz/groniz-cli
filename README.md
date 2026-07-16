# groniz-cli

The agent skill for [Groniz](https://groniz.com) — teaching coding agents to drive
the Groniz CLI, which schedules social media and chat posts across 28+ channels
(X, LinkedIn, Reddit, Instagram, Threads, YouTube, TikTok, Discord, Slack, Mastodon,
Bluesky, Telegram, and more).

The CLI itself is distributed as a self-contained native binary and installed
separately; this repo ships the skill that tells an agent how to use it correctly.

## Install the CLI

```bash
curl -fsSL https://groniz.com/install.sh | sh
```

No Node.js required. Update with `groniz update` (`groniz update --check` exits 10
when a newer version exists).

## Install the skill

```bash
npx skills add groniz/groniz-cli
```

This detects your agent and installs into its skills directory — `.claude/skills/`
for Claude Code, `.agents/skills/` for Cline, Cursor, and OpenCode. To skip the
prompt, name the targets:

```bash
npx skills add groniz/groniz-cli -a claude-code -a opencode
```

## What the skill covers

`groniz --help` already documents every command and flag, so the skill deliberately
does not restate them. It covers what `--help` cannot:

- **Authenticate first** — every command fails without credentials, and the stored
  API key survives across shells and agents where `export` does not.
- **Upload media before posting** — raw paths and external URLs are rejected
  server-side; `groniz upload` returns the `.path` to pass back.
- **Discover channel settings at runtime** — `integrations:settings <id>` is the
  source of truth for required settings and character limits, so agents don't
  hardcode platform assumptions and eat 400s.
- The traps behind the common failures: strict JSON mode shape, differing `jq`
  paths, threads via repeated `-c`, drafts skipping validation, and the
  missing-release-id flow that leaves analytics unavailable.

## Documentation

Public API reference: https://docs.groniz.com/public-api/introduction

## License

Apache 2.0 — see [LICENSE](LICENSE).
