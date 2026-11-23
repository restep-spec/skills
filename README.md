# Restep skills

Agent skills for [Restep](https://restep.wubytes.com/), a profile for HTTP JSON APIs at REST level 2½.

| Skill | Use it to |
|---|---|
| [`restep-design`](skills/restep-design/SKILL.md) | Build an API, or change one, so it follows Restep, then verify it with `restep-check` |
| [`restep-review`](skills/restep-review/SKILL.md) | Review an API change, an endpoint or an OpenAPI document against Restep |

Each skill folder carries the specification (`spec.md`) and the JSON schemas, so the agent can read
them offline. The checker runs from npm, with `npx restep-check` (Node 18 or later).

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

## Source

This repository is generated from the Restep site; do not edit it by hand. The specification is
at <https://restep.wubytes.com/1/spec.md>.

## License

MIT. See [LICENSE](LICENSE).
