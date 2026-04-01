# Installation

## Claude Code

```bash
/plugin marketplace add OpenCageData/opencage-skills
/plugin install opencage@opencage-skills
```

Then restart Claude Code or run `/reload-plugins`.

To make the plugin available to your whole team, add to your project's `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "opencage-skills": {
      "source": {"source": "github", "repo": "OpenCageData/opencage-skills"}
    }
  }
}
```

## Gemini CLI

```bash
gemini extensions install https://github.com/OpenCageData/opencage-skills
```

## Codex

Add to your project's `codex.md` or agent instructions:

```
Fetch and follow instructions from
https://raw.githubusercontent.com/OpenCageData/opencage-skills/refs/heads/master/skills/opencage-geocoding-api/SKILL.md
https://raw.githubusercontent.com/OpenCageData/opencage-skills/refs/heads/master/skills/opencage-geosearch/SKILL.md
```

## Other agents

The skills are standard markdown files. Point your agent at the relevant file:

- **Geocoding API:** `skills/opencage-geocoding-api/SKILL.md`
- **Geosearch widget:** `skills/opencage-geosearch/SKILL.md`

Each `SKILL.md` has a YAML frontmatter `description` field explaining when to use it. The `references/` directory under the geocoding skill contains language-specific SDK guides (Python, Node.js, Ruby, PHP, Java, Perl, and command-line).

Clone the repo and point your agent at the files, or fetch them directly:

```
https://raw.githubusercontent.com/OpenCageData/opencage-skills/refs/heads/master/skills/opencage-geocoding-api/SKILL.md
https://raw.githubusercontent.com/OpenCageData/opencage-skills/refs/heads/master/skills/opencage-geosearch/SKILL.md
```
