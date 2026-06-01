# Contributing a campaign

This repo hosts pluggable "campaigns" for [Blitz](https://blitz.dev). Each
campaign is a self-contained skill bundle that an AI agent can install +
follow to stand up a Blitz-backed app + the workflow that drives it.

The bar for inclusion: **end-to-end usable from one sentence.** A user says
"set up a `<thing>` campaign" and the agent (running on top of Blitz) takes
it from intent to a live URL without the user touching code.

## Required files

```
campaigns/<your-campaign>/
  SKILL.md           ← canonical skill file
  POINTER.md         ← (optional) 3-line snippet for always-on rules harnesses
  config-template.json ← (optional) starter config; setup flow fills in real values
```

`SKILL.md` is the hard requirement. Everything else is optional.

## SKILL.md shape

Frontmatter is mandatory. Use the same shape as the marketing campaign:

```yaml
---
name: blitz-<campaign>-campaign
description: >-
  One-paragraph pitch. Must include the canonical trigger phrases so an agent
  reading SKILL listings can route on intent.
metadata:
  config_path: ~/.claude/skills/blitz-<campaign>-campaign/config.json
  depends_on: <comma-separated sibling skills — e.g. agent-socket-connect>
---
```

Body should follow the pattern:

- **Prerequisites** — what other skills or accounts the user needs.
- **Setup (one time)** — fork the canonical Blitz template, generate tokens,
  save to `config.json`, run the interview, synthesize voice/strategy doc, POST
  to the fork. Be explicit about every API call.
- **Daily commands** — the N user-facing commands (draft / send / review etc).
- **What this skill does NOT do** — explicit non-goals so other agents don't
  drift past the scope.
- **Failure modes** — table of common errors and fixes.

The marketing campaign is the live reference: see [campaigns/marketing/SKILL.md](campaigns/marketing/SKILL.md).

## manifest.json entry

Once your SKILL.md is in place, add an entry to `manifest.json`:

```json
{
  "name": "<short-slug>",
  "skill_name": "blitz-<short-slug>-campaign",
  "title": "Human-readable one-line title",
  "description": "Slightly-longer one-paragraph pitch",
  "trigger_phrases": [
    "set up <thing> campaign",
    "start a new <thing> campaign",
    "..."
  ],
  "platforms": ["x", "reddit"],
  "depends_on": ["agent-socket-connect"],
  "skill_path": "campaigns/<your-campaign>/SKILL.md",
  "fork_template_slug": "<blitz-project-slug-to-fork>"
}
```

The `fork_template_slug` must point at a real Blitz project that lives on the
`blitz.dev` org's account and is publicly forkable. Use an existing campaign's
fork-template as starting clay if you don't have one yet.

## Review checklist (for PR reviewers)

- [ ] SKILL.md frontmatter validates (`name`, `description`, `metadata.depends_on`, `metadata.config_path` present)
- [ ] manifest.json entry uses kebab-case `name`, references a real `skill_path`
- [ ] `fork_template_slug` exists on blitz.dev and is publicly forkable
- [ ] Trigger phrases don't conflict with another campaign
- [ ] Body explains the full lifecycle (setup → daily ops → non-goals → failure modes)
- [ ] No secrets, API keys, or personal tokens checked in
- [ ] Same authoring conventions as marketing campaign (lowercase casual where appropriate, no em dashes if you want it on-brand)

## Cache + rollout

Once merged to `main`, `blitz.dev` picks up the new campaign within the
cache TTL (~5 min). The agent-readable listing at `blitz.dev/campaigns` and
the runtime-injected `## Campaigns` section in `blitz.dev/skill.md` both
refresh from `manifest.json` automatically. No deploy needed on the blitz
side.
