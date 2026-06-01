# blitzdotdev/campaigns

Skill bundles for [Blitz](https://blitz.dev) campaigns. PR a new directory under `campaigns/`, merge, and `blitz.dev` picks it up within 5 minutes.

A campaign is a Blitz app plus the skill that runs it. Triggered by a user sentence like "set up a marketing campaign".

## Add one

1. `campaigns/<name>/SKILL.md` with frontmatter (`name`, `description`, `metadata.config_path`, `metadata.depends_on`).
2. Add an entry to `manifest.json` (`name`, `skill_name`, `title`, `description`, `trigger_phrases`, `skill_path`, `fork_template_slug`).
3. Open a PR.

See [`campaigns/marketing/SKILL.md`](campaigns/marketing/SKILL.md) for the reference.

## What blitz.dev serves

- `/campaigns` list (markdown, or JSON via `?format=json`)
- `/campaigns/<name>/skill.md` full SKILL.md
- `/campaigns/<name>/install.sh` installer that drops SKILL.md into `~/.claude/skills/blitz-<name>-campaign/`
