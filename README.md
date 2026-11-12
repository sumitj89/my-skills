# my-skills

Collection of agent skills. Install with [`skills`](https://github.com/vercel-labs/skills) CLI — no publish step needed, installs directly from GitHub.

## Skills

| Skill | Description |
|-------|-------------|
| [teach-fumadocs](./teach-fumadocs) | Teach a topic via a structured Fumadocs site with real-world examples and human prose. |
| [designer-ui-style](./designer-ui-style) | Enforce Designer UI layout, design-token, and styling constraints for React components. |
| [designer-fabric-canvas](./designer-fabric-canvas) | Fabric.js canvas, layers, hooks, and editor undo/redo architecture for Designer. |

## Install

Requires Node 18+.

### teach-fumadocs

```bash
# install teach-fumadocs to current project (auto-detects agent)
npx skills add sumitj89/my-skills --skill teach-fumadocs

# shorthand with @ syntax
npx skills add sumitj89/my-skills@teach-fumadocs

# full GitHub URL
npx skills add https://github.com/sumitj89/my-skills --skill teach-fumadocs

# global install (all projects)
npx skills add sumitj89/my-skills --skill teach-fumadocs -g -y
```

Verify locally without installing:

```bash
npx skills add sumitj89/my-skills --skill teach-fumadocs --list
```

Other package managers (same CLI):

```bash
pnpm dlx skills add sumitj89/my-skills --skill teach-fumadocs
bunx skills add sumitj89/my-skills --skill teach-fumadocs
```

### designer-ui-style

```bash
# install designer-ui-style to current project (auto-detects agent)
npx skills add sumitj89/my-skills --skill designer-ui-style

# shorthand with @ syntax
npx skills add sumitj89/my-skills@designer-ui-style

# full GitHub URL
npx skills add https://github.com/sumitj89/my-skills --skill designer-ui-style

# global install (all projects)
npx skills add sumitj89/my-skills --skill designer-ui-style -g -y
```

Verify locally without installing:

```bash
npx skills add sumitj89/my-skills --skill designer-ui-style --list
```

Other package managers (same CLI):

```bash
pnpm dlx skills add sumitj89/my-skills --skill designer-ui-style
bunx skills add sumitj89/my-skills --skill designer-ui-style
```

### designer-fabric-canvas

```bash
# install designer-fabric-canvas to current project (auto-detects agent)
npx skills add sumitj89/my-skills --skill designer-fabric-canvas

# shorthand with @ syntax
npx skills add sumitj89/my-skills@designer-fabric-canvas

# global install (all projects)
npx skills add sumitj89/my-skills --skill designer-fabric-canvas -g -y
```

### What it does

Copies/symlinks `teach-fumadocs/SKILL.md` into your agent's skills directory (e.g. `.claude/skills/`, `.cursor/skills/`, `.opencode/skills/`). Updates tracked in `skills-lock.json` for `npx skills experimental_install` restore.

To update later:

```bash
npx skills update teach-fumadocs
```
