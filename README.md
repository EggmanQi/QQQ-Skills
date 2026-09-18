# QQQ-Skills

A collection of agent skills for vertical debugging and engineering scenarios. Each skill is a self-contained directory with a `SKILL.md` that encodes a systematic, battle-tested workflow for one class of hard problems.

## Skills

| Skill | Description |
|---|---|
| [intermittent-bug-flow](./intermittent-bug-flow/SKILL.md) | Systematic flow for debugging intermittent/probabilistic feature failures in voice/social iOS apps (ObjC & Swift) — flag leaks, same-key early-exits, async races, guard chains, duplicate triggers. Invoke when a feature works sometimes but not always, fails only on re-entry, or fires twice. |

## Usage

Each skill follows the standard agent skill format: a directory named after the skill, containing a `SKILL.md` with YAML frontmatter (`name`, `description`) and the protocol body. Point your agent's skill directory here, or copy individual skill folders into your project's skills path.

## License

[MIT](./LICENSE)
