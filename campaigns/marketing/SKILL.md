---
name: blitz-marketing-campaign
description: >-
  Social-media marketing operations the user conducts with their agent. The
  agent drafts X and Reddit replies; the user reviews and sends. A skill with
  memory that improves over time: every send, reject, and revise teaches the
  agent the user's voice and judgment. Eventually runs autonomously. Two daily
  commands: **draft** (research + post N new candidates) and **send** (redraft
  any pending-revision items, then post all pending-send items). Triggers:
  "set up marketing campaign", "draft N messages", "draft a reply to <url>",
  "send".
metadata:
  config_path: ~/.claude/skills/blitz-marketing-campaign/config.json
  depends_on: agent-socket-connect
---

# marketing campaign

The user's marketing inbox. The forked Blitz app at `<slug>.app.blitz.dev` shows pending drafts. The user clicks Revise / Reject / Send in the UI; the app flips pending flags. This skill, running in any Claude Code session, drains the queue: revises drafts, posts to platforms via agent-socket, records outcomes, and folds the user's feedback into the next research cycle.

- **All AI work runs in the user's Claude session.** The Blitz worker is UI + DB only. No Anthropic API key.
- **All browser work delegates to `agent-socket-connect`.** It mints a session URL for the active site (x.com, reddit.com); this skill calls its tool endpoints over HTTPS.

---

## Setup (one time)

Triggered by "set up marketing campaign" or any first-time invocation when `config.json` is missing.

### Steps

1. **Verify the user wants to proceed.** Confirm in chat: "I'll fork the marketing campaign template to your Blitz account, then ask you 5 questions to calibrate your voice. About 5 minutes. Proceed?"

2. **Fork the template.** Call the Blitz API to fork the canonical marketing-campaign template (current source slug: `autolead`) to a unique slug. Pick something like `<username>-marketing-campaign` or ask the user.

   ```bash
   curl -X POST https://blitz.dev/api/v1/projects/fork \
     -H "Content-Type: application/json" \
     -d '{"source_slug":"autolead","slug":"<new-slug>","visibility":"private"}'
   ```

   Response includes `project_id`, `slug`, `agent_link`, `preview_url`, `claim_url`. The fork is anonymous (12h TTL) until claimed; tell the user the `claim_url` and ask them to claim it (Google login) before they close the terminal.

3. **Get the agent token from `agent_link`.** It's encoded in the URL: `/agent/<token>/agents.md` → token is `tp_…`.

4. **Set BOTH worker secrets — required, do not skip.** The deployed UI loads with a "UI_PASSWORD not configured on the worker" error if `UI_PASSWORD` is missing, and every `/api/*` call 503s if `MARKETING_API_TOKEN` is missing. Generate and PUT both before continuing.

   ```bash
   # api_token: gates /api/* (calls from this skill).
   API_TOKEN=$(openssl rand -hex 24)
   # ui_password: gates the human-facing UI. Short so it's easy to type / share.
   UI_PASSWORD=$(openssl rand -hex 4)

   curl -X PUT "https://blitz.dev/api/v1/projects/<slug>/secrets/MARKETING_API_TOKEN" \
     -H "Authorization: Bearer <agent_token>" \
     -H "Content-Type: application/json" \
     -d "{\"value\":\"$API_TOKEN\"}"

   curl -X PUT "https://blitz.dev/api/v1/projects/<slug>/secrets/UI_PASSWORD" \
     -H "Authorization: Bearer <agent_token>" \
     -H "Content-Type: application/json" \
     -d "{\"value\":\"$UI_PASSWORD\"}"
   ```

   `MARKETING_API_TOKEN` is the API surface this skill uses; `UI_PASSWORD` is what the user types in the browser to log into their fork. Both go into `config.json` (next step) so the agent can hand them back to the user later without making them remember.

5. **Detect platform handles via agent-socket-connect — do not ask cold.** The user's X and Reddit handles populate `config.user.{x,reddit}` and gate the duplicate-reply check in Draft. Most users are already logged into these platforms in their dedicated agent-socket browser window, so detect first, ask only on miss.

   For each supported platform (X, Reddit), mint or reuse an agent-socket session via the `agent-socket-connect` skill, open the platform's home page, then `eval` a selector to read the logged-in handle:

   - **X** — open `https://x.com/home` (use `/x_open_home` if the session exposes it, else `/navigate`), then:

     ```js
     // sidebar profile link: href is "/<handle>"
     (() => {
       const a = document.querySelector('[data-testid="AppTabBar_Profile_Link"]');
       return a ? a.getAttribute('href').replace(/^\//, '') : null;
     })()
     ```

   - **Reddit** — `/navigate { url: "https://www.reddit.com" }`, then:

     ```js
     // new reddit (shreddit) exposes the logged-in user on the root element
     (() => {
       const app = document.querySelector('shreddit-app');
       return app?.getAttribute('user-id') ? (app.getAttribute('username') || null) : null;
     })()
     ```

     Fallback (if the shreddit selector returns null but the page rendered): `/navigate { url: "https://old.reddit.com" }`, then `(() => window?.r?.config?.logged || null)()`.

   Handling the results:

   - **Detected on both platforms**: confirm in one message — "I detected `@<x_handle>` on X and `u/<reddit_handle>` on Reddit. Correct?" — and proceed on yes. Don't re-prompt per platform.
   - **Detected on one, null on the other**: save the detected one, tell the user "I couldn't detect a logged-in user on `<platform>` in the agent-socket browser. Either log in there now and reply 'retry', or tell me your `<platform>` handle, or say 'skip' and I'll only operate on `<other-platform>`."
   - **Both null**: same prompt, both platforms. Don't write config.json until at least one handle is set.
   - **Detected but user says wrong**: trust the user, overwrite with what they say.

   Why this is a hard rule: Reddit's per-thread profile links match the OP of the thread you're reading, not the logged-in user, so any later scrape-from-DOM is unreliable. The `config.user` block written in step 6 is the only trusted handle source for the rest of the skill (duplicate-reply check, /with_replies reads, etc.) — and it has to be right.

6. **Save config.json.** Write to `~/.claude/skills/blitz-marketing-campaign/config.json`:

   ```json
   {
     "slug": "<new-slug>",
     "endpoint": "https://<new-slug>.app.blitz.dev",
     "agent_token": "tp_...",
     "api_token": "<API_TOKEN>",
     "ui_password": "<UI_PASSWORD>",
     "user": {
       "x": "<x-handle without @>",
       "reddit": "<reddit-handle without u/>"
     }
   }
   ```

   The `user` block is the trusted source for the user's identity on each platform — the duplicate-reply check in Research reads from here instead of guessing from page DOM (Reddit's profile-link selector matches the OP, not the logged-in user). Per-platform handles can differ for the same person, so collect both. Set file permissions to 0600.

7. **Run the interview** (see "Interview" below). Skip Q3 ("which platforms") for any platform where step 5 detected a handle — that's the implicit affirmative. Only ask Q3 for platforms that were null AND the user didn't say "skip" on.

8. **Synthesize the voice doc** from the interview answers + any files/URLs the user pointed at.

9. **Present back to the user** for confirmation.

10. **POST the voice doc to the fork**:

    ```bash
    curl -X POST "<endpoint>/api/skill" \
      -H "Authorization: Bearer <api_token>" \
      -H "Content-Type: application/json" \
      -d '{"content": "...", "skill_name": "marketing-campaign-voice", "notes": "..."}'
    ```

    Response includes a `hash`. Save the hash, all future drafts reference it as `skill_snapshot_hash`.

11. **Done.** Tell the user:
    - the URL of their fork (`<endpoint>`),
    - their `ui_password` (so they can log in to the UI on first visit),
    - the daily commands they can run ("draft N" / "send").

### Interview (5 questions)

Ask one at a time, wait for the answer, then move on.

1. **What's the purpose of this marketing?** Free text. e.g., "growing awareness of my SaaS product among AI developers." Use this as the overall north star for every draft.

2. **Where can I read your voice?** Multi: paths to local repos, individual files, or URLs. e.g., "~/Documents/notes/, ~/projects/blitz-marketing/plans/, https://github.com/<user>." Use the Read tool for local paths, fetch tool for URLs. Skim, don't deep-read; ~10 files max. Extract: tone, sentence rhythm, characteristic phrases, banned words, what they care about.

3. **Which platforms should I work on?** Multi-select from: X (twitter.com), Reddit, LinkedIn, Hacker News, Discord. Default to X if unclear. **Skip this question for X / Reddit if step 5 already detected a logged-in handle** (treat detection as the implicit affirmative). Only ask it for the platforms step 5 couldn't auto-confirm, plus the ones it doesn't cover (LinkedIn, HN, Discord).

4. **What outcomes matter?** Multi-select with custom: signups for product, awareness/brand, qualified feedback users, community engagement, customer leads, recruiting. The drafts should optimize for these.

5. **Anything banned?** Free text. e.g., "no em dashes, no marketing slogans, no engagement bait, never post about politics." Treat these as hard rules in every draft.

### Synthesizing the voice doc

After answers + file reads, write a markdown doc with these sections in order:

- `## Purpose` — Q1 answer + any framing you inferred
- `## Identity` — who they are from the files (role, what they build, who they engage with)
- `## Voice` — tone, sentence rhythm, ~5 characteristic patterns from their writing
- `## Platforms` — checkbox list, e.g. `- [x] x.com`, `- [ ] linkedin`
- `## Goals (priority order)` — numbered list, top goal first
- `## Hard bans` — bullets, banned phrases/patterns/topics
- `## Patterns learned (max 10, CRUD every cycle)` — empty on first synthesis; populated by Review
- `## Examples` — empty on first synthesis; populated by Review from `/api/examples?format=md`
- `## Rejections` — empty on first synthesis; populated by Review from `/api/rejections`

Present as a single block. Ask: "Does this look right? Tell me what to add or fix." Iterate, then POST to `/api/skill`.

---

## Daily commands

Two user-facing commands: **draft** and **send**. Both start with `GET <endpoint>/api/skill` to load the current voice snapshot (treat its `content` as a high-priority context block, like a system prompt). If `/api/skill` returns null, setup isn't done — tell the user to run "set up marketing campaign".

### Draft

Triggers: "draft N messages", "draft a reply to <url>", "find me leads", "look at this tweet/post and draft something".

Two modes:
- **Single-target**: user supplies a URL. Run Review (step 1), then run steps 2–9 for that one URL.
- **Bulk auto-discovery**: user asks for N candidates (default 5). Run Review (step 1), then scan the platforms in `## Platforms` for fresh threads matching the voice doc's purpose + goals, score by EV, take the top N, and run steps 2–9 for each.

> **HARD RULE — hitting N is non-negotiable in bulk mode.**
>
> If the user asks for N candidates, you POST exactly N to `/api/research`. Not N-2 with an apology. Not "the pool is dry, here are 3." Quality-filter patterns from the voice doc apply *inside each candidate*; they are not a budget cap on the cycle total. "Pool feels small" is not an exit condition.
>
> Exhaust every surface the voice doc's `## Platforms` section names before stopping. For each platform: multiple communities (subs/topics/hashtags) per voice-doc goal, both /new and /hot/top, keyword search on terms from `## Purpose` and `## Goals`, and scrolling past the first page. The voice doc is the only source for *which* communities are in scope — do not bake any community list into this skill.
>
> The only acceptable short-of-N outcome is: every community listed in the voice doc has been queried on every relevant tab and keyword set, and you can point to the scrolled pages. Anything else is a search bug, not a justified outcome. Before stopping short, check: have I actually exhausted what the voice doc points at? If not, keep going.

#### Step 1 — Review the last cycle's UI feedback (always run first)

Goal: read **every UI interaction the user did since the snapshot's `captured_at`** and fold the signal into a fresh snapshot before drafting. Skipping a category (especially pre-send manual edits, which look passive but are strong intent) means Draft works from a partial view of what the user actually wants and repeats the same mistakes.

**Early-exit**: if no feed_item has `updated >= captured_at` AND no items are in `revisions`/`sends` queues, skip Review and go straight to step 2.

**Signal sources** (query all of them, every time):

| User did this in the UI | Where the signal lives | How to read it |
|---|---|---|
| Sent without editing | `examples` row, `outcome='approved-clean'` | weak positive (target OK, voice as-drafted OK) |
| Sent with manual edit | `examples` row, `edit_chain` contains `user_manual_edit` | **strong positive** — the user's final text is the desired form. Diff against `initial_draft` |
| Reject with reason | `feed_items.rejection_reason`, surfaced by `/api/rejections` | **strong negative** — this candidate shape shouldn't recur |
| Revise (text feedback) | `feed_items.revision_feedback`, surfaced by `/api/pending` revisions | **mixed; must triage**. May be real revision direction ("make it shorter") OR reject-with-reason content typed before the Reject button was visible (browser cache). If it reads as "this candidate is bad because X", reroute via `POST /items/:id/reject` with the same text as `reason`, then treat as negative signal |
| Queued for send, not yet sent (`send_pending=1`) | `feed_items.edit_chain` for `user_manual_edit` entries | **strong positive** — same as a sent-with-edit. The send is provisional but the user's edited text is intent. Easy to overlook because it isn't terminal — query `/api/pending` explicitly |
| Quick discard (trash icon, no reason) | `status='rejected'`, `rejection_reason=NULL` | no signal — ignored |

**Procedure**:

1. Pull every signal source since `captured_at` — three calls, no shortcuts:
   - `GET <endpoint>/api/examples?since=<captured_at>` → sent items (positive)
   - `GET <endpoint>/api/rejections?since=<captured_at>` → rejected with reason (negative)
   - `GET <endpoint>/api/pending` → in-progress queue. Filter to rows where `updated >= captured_at`. For each:
     - **Revision-pending**: read `revision_feedback`. Triage real-revision vs misclassified-reject (per table). Reroute misclassified ones via `POST /items/:id/reject` before continuing.
     - **Send-pending**: parse `edit_chain` for `user_manual_edit` entries, diff against the first `ai`-role entry.
2. **Group items by audience / channel / topic before reading individuals.** Per-item reading misses signal that only emerges when 3+ related drafts get the same kind of edit (e.g. multiple drafts in the same sub all rewritten to point at one canonical link, or all reframed around the same trusted brand). Cross-item patterns are usually the strongest signal in a cycle and only surface under grouping.
3. Decide two things:
   - **Voice base** (purpose / identity / voice / platforms / goals / hard bans): edit only if a signal contradicts or extends what's there; base sections decay slowly.
   - **Patterns learned (≤10 bullets)**: CRUD across all signal types. Add when the cycle gives clear signal not covered by an existing bullet (rejection reasons, heavy manual edits, cross-item patterns all qualify). Update when new evidence sharpens or generalizes. Delete the weakest old single-data-point if adding pushes count over 10. Stay honest — leave count below 10 if no real signal.
4. Rebuild the doc: same base sections (possibly edited) + updated `Patterns learned` + a fresh `Examples` section literally from `GET /api/examples?since=...&format=md` + a hand-formatted `Rejections` section listing each `rejection_reason` with parent context.
5. `POST <endpoint>/api/skill` with the rebuilt content. The new snapshot becomes the base for the rest of Draft (steps 2–9 use the new patterns, new rejections, etc).
6. Briefly tell the user what changed (which patterns added/updated/deleted, new hash, any revisions rerouted to rejections) before moving on.

#### Steps 2–9 — Research and post each target

2. Determine the platform from the URL (x.com vs reddit.com vs etc).
3. Ensure an agent-socket session for that platform exists. If not, invoke the `agent-socket-connect` skill to mint one.
4. **Fetch the parent context AND use it as a validity gate.** Parent-of-parent is used for two things, not one:
   1. **Validity gate (decide whether to draft at all).** If the parent-of-parent (or quoted tweet) is off-topic for our product context, the focal is NOT a valid target no matter how interesting the focal text reads in isolation. Concrete example: a reply that says "Oui'd?" with the focal text alone looks intriguing, but if the parent-of-parent is one of the user's own personal jokes, the focal has zero Blitz hook and the candidate must be skipped, not drafted-around. **Skip rule**: parent-of-parent is unrelated to product domain AND the focal doesn't introduce its own product-relevant angle → don't POST to `/api/research`, just report "skipped — focal is a reply to off-topic parent: <parent excerpt>".
   2. **Draft context (when the candidate IS valid).** Flatten parent + focal so the draft answers what the OP is actually responding to, not just the focal text.

   How to fetch:
   - **X**: `POST <socket>/x_open_tweet { url }`, then `/x_get_tweet_thread { limit: 3 }`. **Pitfall**: navigating to `https://x.com/i/status/<id>` for a reply shows the PARENT as `article[0]` and the focal as `article[1+]`. Search-result scrapes that grab the first `/status/` link inside an article often capture the QUOTED tweet's id, not the focal's. Match by author handle or by the `time` element's parent anchor to get the focal status reliably. Flatten as `parent_text`: `"Thread root (@root): <text>\n\nReplying to (@parent): <text>\n\nFocal: <text>"`. For quote-tweets specifically, capture the quoted tweet inline: `"Quoting (@quoted_author): <quoted text>\n\nFocal (@author): <focal text>"`.
   - **Reddit**: `POST <socket>/navigate { url }`. If the URL is `/comments/<id>/.../<comment_id>/` (target is a comment), also read the OP title+body and parent-comment chain via `eval` on `shreddit-comment[thingid="<id>"]`. Same flatten format.
   - Bare top-level posts/tweets need no flattening.
5. **Duplicate-reply check.** Verify the user hasn't already replied — via this system OR directly on the platform. Read the per-platform handle from `config.user.{x|reddit}` (the only trusted source; do NOT scrape from page DOM — Reddit's profile links match the OP, not the logged-in user).
   - **DB**: `GET /api/examples?q=<status_id_or_post_id>&limit=3`. Hit on this parent's permalink → skip.
   - **Live thread**:
     - X: eval `Array.from(document.querySelectorAll('article')).some(a => Array.from(a.querySelectorAll('a[data-testid="User-Name"]')).some(l => l.href.endsWith('/' + config.user.x)))`
     - Reddit: eval `Array.from(document.querySelectorAll('shreddit-comment')).some(c => (c.getAttribute('author')||'').toLowerCase() === config.user.reddit.toLowerCase())`
   - If a duplicate is found, do NOT POST to `/api/research`. Report: "skipped — you already replied at <link>". Most common false-positive in cold marketing research; never skip this check.
6. **Check fresh rejections.** `GET /api/rejections?since=<voice_doc.captured_at>`. If the current candidate resembles a rejected one (same sub, similar parent shape, same flagged angle), skip and report: "skipped — resembles rejection of <id>: <reason>". Most common bulk-research failure mode.
7. **Generate the draft.** Hold the voice snapshot in working context. Read the parent context. If the voice doc doesn't give enough signal for this shape — unfamiliar audience, ambiguous angle, EV near the skip line — query past examples for priors before drafting:
   - `GET /api/examples?channel=X&audience=Y&limit=10` — same-shape items
   - `GET /api/examples?q=<keyword>&limit=10` — substring on parent + drafts
   - `GET /api/examples?goal=Z&channel=X&limit=10` — items kept for this goal type
   Read the returned drafts + diffs as concrete priors, not templates. If still ambiguous after 1–2 queries, ask the user for one-line direction.

   Produce: `summary_subject`, `summary_body` (2–3 lines on why it's worth their time), `ev_reasoning`, `ev_score` (0.0–1.0), `current_draft`.
8. **POST the feed_item** to `/api/research` with all those fields plus `channel`, `message_type`, `audience`, `goal`, `parent_text`, `parent_author_handle`, `parent_author_meta` (JSON: `{name, follower_count, verified}`), `parent_engagement` (JSON: `{likes, retweets, replies}`), `parent_posted_at` (ISO), `parent_url`, `target_handle`, `skill_snapshot_hash`. Standard `Authorization: Bearer <api_token>` header.
9. Tell the user the URL `<endpoint>/items/<new_id>` and a one-line summary.

### Send

Triggers: "send", "send pending", "drain the queue", "post the approved drafts".

Does two things in order: (1) **redrafts** any items the user clicked Revise on, then (2) **posts** any items the user clicked Send on. Folded into one command because the user usually wants both done before any newly-drafted items show up in the inbox.

Steps:
1. `GET <endpoint>/api/pending` → returns `{ revisions: [...], sends: [...] }`.
2. **Redraft each revision.** For every item in `revisions`:
   - Read `revision_feedback` (note the user typed) + the latest `edit_chain` entry for prior drafts.
   - **Triage first**: is this feedback real revision direction ("shorter", "more skeptical") or reject-with-reason content ("wrong audience", "OP is committed to X")? If the text reads as a kill reason, route via `POST /items/:id/reject` with the same text as `reason` and skip the redraft. Otherwise:
   - Generate a new draft that addresses the feedback. Be honest about what changed. POST to `/api/items/:id/apply-revision` with `{draft, subject, note, skill_snapshot_hash}`. The item flips back to a fresh draft for the user to review.
3. **Post each send.** For every item in `sends` after redrafts complete:
   - Ensure a fresh agent-socket session for the channel exists; mint via `agent-socket-connect` if not.
   - Drive the platform:
     - **X reply** (when `parent_url` is a status permalink): navigate to `https://x.com/intent/post?text=<encoded>&in_reply_to=<status_id>`, wait for the composer, click `[data-testid=tweetButton]`. Confirm via `/page_info` showing "Your post was sent." or by reading the user's `/with_replies` profile.
     - **X new post**: `/x_open_home`, `/x_compose_draft { text }`, `/x_post_now`.
     - **Reddit comment**: `/navigate { url: parent_url }`, `/reddit_compose_reply_to_post { text }`, `/reddit_submit_reply`. Capture the returned permalink.
   - On success: `POST /api/items/:id/mark-sent` with `{platform_url}`. On failure: `POST /api/items/:id/mark-send-failed` with `{error}`.
4. Report: "Redrafted N. Sent M. Failed K."

## What this skill does NOT do

- Send anything on the user's behalf without an explicit Send action in the UI. Even "send pending" only sends items the user already clicked Send on; that flipped `send_pending=1` and recorded their intent.
- Make Anthropic API calls. All AI is the user's running Claude session.
- Persist any state locally beyond `config.json`. Voice, drafts, examples all live in the fork DB.
- Mint or refresh agent-socket sessions on its own. Delegates to `agent-socket-connect`.
- Modify the fork's worker code or schema. Read/write through `/api/*` only.

## Failure modes

| Symptom | Cause | Fix |
|---|---|---|
| `401 unauthorized` on /api | `api_token` in config doesn't match `MARKETING_API_TOKEN` secret on worker | re-set the secret, update config.json |
| `503 MARKETING_API_TOKEN not configured` | secret not set on worker | use agent_token to PUT the secret (step 4) |
| `UI_PASSWORD not configured on the worker` shown in browser | `UI_PASSWORD` secret not set on worker | generate one (`openssl rand -hex 4`), PUT to `/api/v1/projects/<slug>/secrets/UI_PASSWORD`, save into `config.json` as `ui_password`, tell the user. This is the setup step 4 the agent skipped. |
| `app_offline` on socket call | session died | mint a new one via agent-socket-connect, optionally cache the new URL |
| `GET /api/skill` returns null | no voice yet | run "set up marketing campaign" or re-run interview |
| revision draft same as previous | feedback didn't actually drive change | look harder at the feedback, regenerate |
| send fails on platform | DOM changed, captcha, rate limit | report to user, mark-send-failed with explicit error |

## Multi-fork

v1 assumes one fork per machine. `config.json` is single-tenant. If the user has multiple forks, manual config switching for now.
