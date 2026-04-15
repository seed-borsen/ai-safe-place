# ai-safe-place
A safe place for Claude AI

## Skills (Slash Commands)

Skills are on-demand playbooks you invoke with /skill-name. They only consume context when used.

Create a personal skill:
`~/.claude/skills/\<skill-name\>/SKILL.md`

  Or a project-scoped skill:
  `.claude/skills/<skill-name>/SKILL.md`

  Basic SKILL.md format:
  ```
  name: my-skill
  description: What this skill does and when Claude should use it
  ```

  Your instructions/playbook here...

  Skills take effect immediately - no restart needed.

  ---
  CLAUDE.md (Guidance Files)

  CLAUDE.md is always-loaded context, not a skill. Use it for persistent
  project knowledge:
  - Architecture decisions
  - Coding conventions
  - Team guidelines

  Paths:
  - .claude/CLAUDE.md — project-specific
  - ~/.claude/CLAUDE.md — applies to all projects

  You can generate one automatically with /init.

  ---
  Key Difference
|             |          CLAUDE.md          |              Skills               |
| ------------| ----------------------------|-----------------------------------|
| When loaded | Every session               | Only when invoked                 |
| Best for    | Facts, conventions, context | Procedures, workflows, checklists |

  ---
  Other Customization

  - Hooks (settings.json) — automate actions before/after commands
  - MCP servers (.claude/mcp.json) — connect external tools
  - Subagents (.claude/agents/) — specialized delegated agents
  - Keybindings (~/.claude/keybindings.json) — custom shortcuts
