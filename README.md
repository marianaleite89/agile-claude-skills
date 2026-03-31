# claude-skills

My personal Claude Code skills.

## Skills

| Skill | Description |
|-------|-------------|
| [sonos-quarter-review](skills/sonos-quarter-review/) | Generates slide-ready quarterly delivery summaries for Sonos IT projects |

## Installation

Add to your `~/.claude/settings.json`:

```json
"extraKnownMarketplaces": {
  "marianaleite89-skills": {
    "source": {
      "source": "github",
      "repo": "marianaleite89/agile-claude-skills"
    }
  }
},
"enabledPlugins": {
  "sonos-quarter-review@marianaleite89-skills": true
}
```

Then restart Claude Code.
