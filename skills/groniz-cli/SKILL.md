---
name: groniz-cli
description: Schedules and manages social media and chat posts with the Groniz CLI — authenticate, discover channels and their required settings, upload media, create scheduled posts, drafts, or threads, and read post and platform analytics. Use when a user wants to schedule, draft, or analyze a post, or mentions Groniz or any of its 28+ channels — X, LinkedIn, LinkedIn Page, Reddit, Instagram, Facebook Page, Threads, YouTube, Google My Business, TikTok, Pinterest, Dribbble, Discord, Slack, Kick, Twitch, Mastodon, Bluesky, Lemmy, Farcaster, Telegram, Nostr, VK, Medium, Dev.to, Hashnode, WordPress, ListMonk.
license: Apache-2.0
compatibility: Requires network access and jq. Installs the groniz native binary from https://groniz.com/install.sh
metadata:
  openclaw: '{"emoji":"🌎","requires":{"bins":[],"env":[]}}'
---

Groniz CLI schedules posts across 28+ social channels. `groniz --help` lists every
command; `groniz <command> --help` gives its flags and examples. This guide covers
only what `--help` can't: the two hard rules, the workflow, and the non-obvious
traps that cause 400s. The HTTP API behind the CLI is documented at
<https://docs.groniz.com/public-api/introduction>.

Install if missing (self-contained native binary, no node required):
`curl -fsSL https://groniz.com/install.sh | sh`. Self-update: `groniz update --check`
(exit 10 = behind) then `groniz update`.

## Two hard rules

**1. Authenticate first.** Every command fails without credentials. Check with
`groniz whoami`. To set up, prefer the STORED key — it survives across shells
and agents (`export` only affects the current shell):

```bash
groniz auth:apikey set --api-key <key>
groniz auth:apikey show                          # what's stored + which source wins
```

Alternatives: `groniz auth:login` (OAuth device flow) or `export GRONIZ_API_KEY=<key>`
(this shell only). Precedence: **credentials file > env > default**.

**2. Upload media before posting.** Never pass a raw path or external URL to `-m`.
Run `groniz upload <file>` first and pass back its `.path`:

```bash
MEDIA=$(groniz upload image.png | jq -r '.path')
groniz posts:create -c "Content" -m "$MEDIA" -s "2030-12-31T12:00:00Z" -i "<id>"
```

Enforcement is server-side (`RESTRICT_UPLOAD_DOMAINS`) and, where enabled, rejects
non-Groniz URLs at create time — drafts included. Upload first regardless. Wherever
this guide shows `-m "something.jpg"`, read it as "the `.path` from `groniz upload`".

## Workflow

```bash
# 1. Auth — see rule 1
groniz whoami

# 2. Discover — get the channel id AND its required settings/limits (mandatory).
groniz integrations:list                       # ids + identifiers (X = "x"); bare JSON array
groniz integrations:settings <id>              # read .output.settings.required, .output.maxLength, .output.tools

# 3. Fetch dynamic data (only if .output.tools is non-empty)
groniz integrations:trigger <id> <method> -d '{"key":"value"}'

# 4. Upload media (rule 2)
MEDIA=$(groniz upload image.png | jq -r '.path')

# 5. Post — include EVERY required setting for the channel
groniz posts:create -c "Content" -m "$MEDIA" -s "2030-12-31T12:00:00Z" \
  --settings '{"who_can_reply_post":"everyone"}' -i "<id>"

# 6. Analyze
groniz analytics:platform <id> -d 30
groniz analytics:post <post-id> -d 7
```

**`integrations:settings <id>` is the source of truth**, always correct for the live
instance. Before posting to any channel, read it:
- `.output.settings.required` — the mandatory settings (a missing one is a 400).
  E.g. **X requires `who_can_reply_post`**; Reddit and YouTube require a title.
- `.output.maxLength` — the character limit (enforced on scheduled/published posts,
  not drafts — keep within it yourself).
- `.output.tools` — callable integration tools. May be `[]` (e.g. X on some
  instances); calling `integrations:trigger` then returns `404 Tool not found`.

Don't hardcode platform assumptions — discover them here at runtime.

## Non-obvious traps

**jq paths differ.** `integrations:list` returns a bare array (`.[]`); `posts:list`
returns an object (`.posts[]`). stdout is pure JSON; status lines go to stderr, so
piping to `jq` is safe. Avoid a `$PATH` variable name — it shadows the shell PATH.

**Comments = threads.** Repeat `-c` for a thread (X) or reply chain; each `-c` can
have its own `-m`. `-d` is delay-in-minutes between them. Never put a `#` comment on
a `\`-continued line — it eats the newline and drops the next arg.

```bash
groniz posts:create -c "1/3" -c "2/3" -c "3/3" -d 5 -s "2030-12-31T12:00:00Z" -i "<x-id>"
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

**Missing release id.** If `analytics:post` returns `{"missing": true}` (the post
published but the platform gave no usable id — e.g. TikTok), analytics stay
unavailable until you link it:

```bash
groniz posts:missing <post-id>                    # list provider content: [{id, url}, ...]
groniz posts:connect <post-id> --release-id "<id>" # link the right one; analytics work after
```

**Draft skips validation.** `-t draft` skips content-length and provider-settings
checks at create time — a too-long or mis-configured draft passes create and fails
only when promoted with `groniz posts:status <id> --status schedule`.

## Command map

```
posts:create        -c content (repeat=thread) · -m media (uploaded .path) · -i ids (comma=multi-channel)
                    -s ISO-8601 date (REQUIRED) · -t schedule|draft · -d delay-min · --settings '<json>' · --json <file>
posts:list          --startDate / --endDate (default: -30d..+30d) → { posts: [...] }
posts:delete <id>   ·  posts:status <id> --status draft|schedule
posts:missing <id>  ·  posts:connect <id> --release-id "<id>"      (missing-release flow)
integrations:list [--group <id>]  ·  integrations:groups  ·  integrations:settings <id>
integrations:trigger <id> <method> -d '<json>'
analytics:platform <id> [-d days]  ·  analytics:post <id> [-d days]   (default 7)
upload <file>       → { id, name, path, ... }  — use .path
auth:apikey set|show  ·  auth:login  ·  auth:logout  ·  whoami  ·  update [--check]
```
