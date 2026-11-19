# Restep skills

Agent skills for [Restep](https://restep.wubytes.com/), a profile for HTTP JSON APIs at REST level 2½.

| Skill | Use it to |
|---|---|
| [`restep-design`](skills/restep-design/SKILL.md) | Build an API, or change one, so it follows Restep, then verify it with `restep-check` |
| [`restep-review`](skills/restep-review/SKILL.md) | Review an API change, an endpoint or an OpenAPI document against Restep |

Each skill folder is self-contained: it carries the specification (`spec.md`), the checker
(`restep-check.mjs`) and the JSON schemas, so it works offline.

## Install

Any agent (Claude Code, Codex, Cursor and others), with the [skills](https://skills.sh/) CLI:

```bash
npx skills add restep-spec/skills
npx skills add restep-spec/skills --skill restep-design   # just one
```

Claude Code, as a plugin:

```
/plugin marketplace add restep-spec/skills
/plugin install restep@restep
```

Or by hand: download [restep-design.zip](https://restep.wubytes.com/skills/restep-design.zip) or
[restep-review.zip](https://restep.wubytes.com/skills/restep-review.zip) and unzip it in your
agent's skills folder (for Claude Code, `~/.claude/skills/`).

## Source

This repository is generated from the Restep site; do not edit it by hand. The specification is
at <https://restep.wubytes.com/1/spec.md>.

## License

MIT. See [LICENSE](LICENSE).
