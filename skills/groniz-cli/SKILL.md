---
name: groniz-cli
description: Queues immediate publication, schedules, and manages social media, chat posts, and articles with the Groniz CLI — authenticate, discover channels and their required settings, upload media, create immediate or scheduled posts, drafts, or threads, read post and platform analytics, and research any public X account (profile, posts, search, engagement metrics) without connecting it. Use when a user wants to publish now, schedule, draft, or analyze a post, research a competitor or a topic on X, or mentions Groniz or one of its 32+ channels, including X, LinkedIn, Instagram, Facebook, Threads, YouTube, TikTok, Reddit, Pinterest, Discord, Slack, Telegram, Bluesky, Mastodon, and WordPress.
license: Apache-2.0
metadata:
  openclaw: '{"emoji":"🌎","requires":{"bins":[],"env":[]}}'
---

# Groniz CLI

Groniz CLI schedules posts across 32+ social channels, and reads public data from a platform.
Mainstream channels include X, LinkedIn, Instagram, Facebook, Threads, YouTube, TikTok,
Reddit, Pinterest, Discord, Slack, Telegram, Bluesky, Mastodon, and WordPress.
Commands are grouped into domains — `groniz <domain> <command>` — but `groniz --help` still
names every one of them in a single screen, so one call gets you the whole map;
`groniz <domain> <command> --help` then adds flags, examples, and a `Risk · Cost` footer.
This guide covers only what `--help` can't: the two hard rules, the workflow, and the
non-obvious traps that cause 400s. The HTTP API behind the CLI is documented at
<https://docs.groniz.com/public-api/introduction>.

Requires network access and `jq`. Install if missing (self-contained native binary,
no node required):
`curl -fsSL https://groniz.com/install.sh | sh`. Self-update: `groniz update --check`
(exit 10 = behind) then `groniz update`.

## Two hard rules

**1. Authenticate first.** Every command fails without credentials. Check with
`groniz whoami`. To set up, prefer the STORED key — it survives across shells
and agents (`export` only affects the current shell):

```bash
groniz auth apikey set --api-key <key>
groniz auth apikey show                          # what's stored + which source wins
```

Alternatives: `groniz auth login` (OAuth device flow) or `export GRONIZ_API_KEY=<key>`
(this shell only). Precedence: **credentials file > env > default**.

**2. Upload media before posting.** Never pass a raw path or external URL to `-m`.
Run `groniz upload <file>` first and pass back its `.path`:

```bash
MEDIA=$(groniz upload image.png | jq -r '.path')
groniz posts create -c "Content" -m "$MEDIA" -s "2030-12-31T12:00:00Z" -i "<id>"
```

Enforcement is server-side (`RESTRICT_UPLOAD_DOMAINS`) and, where enabled, rejects
non-Groniz URLs at create time — drafts included. Upload first regardless. Wherever
this guide shows `-m "something.jpg"`, read it as "the `.path` from `groniz upload`".

## Workflow

```bash
# 1. Auth — see rule 1
groniz whoami

# 2. Discover — get the channel id AND its required settings/limits (mandatory).
groniz integrations list                       # ids + identifiers (X = "x"); bare JSON array
groniz integrations settings <id>              # read .output.settings.required, .output.maxLength, .output.tools

# 3. Fetch dynamic data (only if .output.tools is non-empty)
groniz integrations trigger <id> <method> -d '{"key":"value"}'

# 4. Upload media (rule 2)
MEDIA=$(groniz upload image.png | jq -r '.path')

# 5. Post — include EVERY required setting for the channel
groniz posts create -c "Content" -m "$MEDIA" -s "2030-12-31T12:00:00Z" \
  --settings '{"who_can_reply_post":"everyone"}' -i "<id>"

# Or queue immediate publication — `now` needs no dummy schedule date
groniz posts create -c "Content" -m "$MEDIA" -t now \
  --settings '{"who_can_reply_post":"everyone"}' -i "<id>"

# 6. Analyze
groniz analytics platform <id> -d 30
groniz analytics post <post-id> -d 7

# 7. Research (optional) — read ANY public X account, connected or not
groniz reads x search --query "ai agents" --since 7d    # what is being said
groniz reads x metrics --username competitor            # what performs for them
```

**`integrations settings <id>` is the source of truth**, always correct for the live
instance. Before posting to any channel, read it:
- `.output.settings.required` — the mandatory settings (a missing one is a 400).
  E.g. **X requires `who_can_reply_post`**; Reddit and YouTube require a title.
- `.output.maxLength` — the character limit (enforced on scheduled/published posts,
  not drafts — keep within it yourself).
- `.output.tools` — callable integration tools. May be `[]` (e.g. X on some
  instances); calling `integrations trigger` then returns `404 Tool not found`.

Don't hardcode platform assumptions — discover them here at runtime.

## Reading platform data (`reads`) vs your analytics (`analytics`)

Two different questions with two different costs. Pick with this rule:

| Question | Command | Reads with | Cost |
|---|---|---|---|
| How did MY connected channel do? | `analytics platform` / `analytics post` | your channel's OAuth token | free |
| Anything about ANY public account | `reads x *` | the server's provider key | metered per upstream call |

**Use `analytics` for your own account.** `reads x metrics --username <you>` returns
much the same thing and bills for it. `reads` earns its cost when the account is **not
yours** (a competitor, a prospect) or has no connected channel — `analytics` cannot see
those at all.

```bash
groniz reads x profile  --username elonmusk              # who they are, follower counts
groniz reads x posts    --username elonmusk --since 7d   # what they posted lately
groniz reads x search   --query "from:elonmusk grok"     # X search syntax works
groniz reads x metrics  --username elonmusk --since 30d  # totals + averages over a window
groniz reads x post     <post-id>                        # one post by id
```

Notes that save a round-trip:
- **A target is mandatory** (`--username` or `--user-id`). There is no "me" here — reads use
  the server's key, not your token, so nothing identifies you.
- **`--user-id` is cheaper than `--username`**: it skips the handle lookup. `profile`, `posts`
  and `metrics` return the id (`.id` / `user_id`) — reuse it across follow-up reads. (`search`
  and `post` don't: they carry `posts[].author_id` instead.)
- `--since` takes a duration (`30d`, `12h`, `45m`), a date (`2026-07-01`), or an ISO timestamp.
  Anything else is a 400 — the window is never silently ignored. `metrics` defaults to 30 days;
  `posts` and `search` have no window unless you set one. `profile` takes no `--since`/`--limit`.
- `metrics.averages` is `null` on an empty sample — that means "no posts in the window", NOT
  "zero engagement". Widen `--since` before concluding anything.
- Only `x` exists today; more platforms are coming behind the same `reads <platform> *` shape.
- `503` means the server has no read key configured: reads are unavailable there, and there
  is nothing wrong with your credentials.

## Non-obvious traps

**jq paths differ.** `integrations list` returns a bare array (`.[]`); `posts list`
returns an object (`.posts[]`). stdout is pure JSON; status lines go to stderr, so
piping to `jq` is safe. Avoid a `$PATH` variable name — it shadows the shell PATH.

**Comments = threads.** Repeat `-c` for a thread (X) or reply chain; each `-c` can
have its own `-m`. `-d` is delay-in-minutes between them. Never put a `#` comment on
a `\`-continued line — it eats the newline and drops the next arg.

```bash
groniz posts create -c "1/3" -c "2/3" -c "3/3" -d 5 -s "2030-12-31T12:00:00Z" -i "<x-id>"
```

**JSON mode shape** (`--json file.json`) — for multi-platform or complex posts. The
shape is strict; the wrong shape is a 400. Each post is
`{ integration: { id }, value: [ { content, image } ], settings }`. There is NO
top-level `integrations` array and NO `provider`/`post`/`__type` field (the backend
derives type from the id). `image` is a REQUIRED array — `[]` for none, else objects
`{ "id": "<any>", "path": "<groniz upload .path>" }`, never bare strings.

```json
{
  "type": "schedule",
  "date": "2030-01-01T12:00:00Z",
  "posts": [
    { "integration": { "id": "<x-id>" },
      "value": [{ "content": "Short tweet #tech", "image": [] }],
      "settings": { "who_can_reply_post": "everyone" } },
    { "integration": { "id": "<linkedin-id>" },
      "value": [{ "content": "Longer version…", "image": [{ "id": "a", "path": "<.path>" }] }] }
  ]
}
```

**Missing release id.** If `analytics post` returns `{"missing": true}` (the post
published but the platform gave no usable id — e.g. TikTok), analytics stay
unavailable until you link it:

```bash
groniz posts missing <post-id>                    # list provider content: [{id, url}, ...]
groniz posts connect <post-id> --release-id "<id>" # link the right one; analytics work after
```

**Draft skips validation.** `-t draft` skips content-length and provider-settings
checks at create time — a too-long or mis-configured draft passes create and fails
only when promoted with `groniz posts status <id> --status schedule`.

**`-t now` starts publication asynchronously.** It removes the scheduled wait and
queues the existing provider workflow immediately; the command returns the created
post JSON, not a synchronous confirmation from the remote platform. Omit `-s` in
this mode. `schedule` and `draft` still require an ISO-8601 date.

## Command map

Commands are grouped into domains: `groniz <domain> <command>`. `groniz --help` names every
command in one screen; `groniz <domain> <command> --help` adds flags, examples, and a
`Risk: … · Cost: free|metered` footer.

```
posts create        -c content (repeat=thread) · -m media (uploaded .path) · -i ids (comma=multi-channel)
                    -s ISO-8601 date (unless `-t now`) · -t schedule|draft|now · -d delay-min · --settings '<json>' · --json <file>
posts list          --startDate / --endDate (default: -30d..+30d) → { posts: [...] }
posts delete <id>   ·  posts status <id> --status draft|schedule
posts missing <id>  ·  posts connect <id> --release-id "<id>"      (missing-release flow)
integrations list [--group <id>]  ·  integrations groups  ·  integrations settings <id>
integrations trigger <id> <method> -d '<json>'
analytics platform <id> [-d days]  ·  analytics post <id> [-d days]   (default 7, free)
reads x profile     --username U | --user-id ID            (no window flags)
reads x posts|metrics  --username U | --user-id ID  [--limit N] [--since 30d|2026-07-01|ISO]
reads x search      --query "..." [--limit N] [--since 7d]  ·  reads x post <post-id>
                    (any public account; metered — for YOUR channel use analytics instead)
upload <file>       → { id, name, path, ... }  — use .path
auth apikey set|show  ·  auth login  ·  auth logout  ·  whoami  ·  update [--check]
```
