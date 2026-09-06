# DOXA Marketing Strategy System v1.1

Portable knowledge and reasoning package for DOXA Intelligence.

## Minimum required
- `DOXA_MARKETING_PLAYBOOK.md`
- `DOXA_MARKETING_STRATEGIST.md`

## Recommended supporting knowledge
- `references/methodology.md`
- `references/proof-points.md`
- `references/terminology.md`
- `references/scientific-foundation.md`
- `references/research-evidence.md`

## Platform adapters
- Codex: `adapters/codex/SKILL.md`
- Hermes: `adapters/hermes/system-prompt.md`
- Paperclip: `adapters/paperclip/agent.md`
- ChatGPT-style skill: `adapters/chatgpt/SKILL.md`
- Other systems: `adapters/generic/bootstrap.md`

## Principle
**Knowledge once, adapters many.**

The Playbook is canonical strategy.
The Strategist file defines how an agent applies it.
References contain supporting or evolving knowledge.
Adapters remain thin.

## Governance
### Canonical / human-approved
`DOXA_MARKETING_PLAYBOOK.md`

### Behavioral policy
`DOXA_MARKETING_STRATEGIST.md`

### Controlled references
`references/methodology.md`
`references/proof-points.md`
`references/terminology.md`

### Research / evolving evidence
`references/scientific-foundation.md`
`references/research-evidence.md`

## Update workflow
1. Agent discovers a new learning.
2. Agent proposes a change.
3. Human reviews it.
4. Approved change is merged.
5. Version/changelog are updated.
6. Agents load the new version.

## Installation
Keep this folder intact whenever the target platform can mount a directory or repository. Point the agent at the appropriate adapter and make the canonical files available.

If a platform supports only a system prompt, use the adapter as the system prompt and attach/mount the canonical knowledge files where possible.
