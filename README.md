# hl-skills

A collection of AI agent skills for Claude Code, OpenCode and other compatible agents.

## Install

```bash
npx skills add imzhuhl/hl-skills
```

Install a specific skill:

```bash
npx skills add imzhuhl/hl-skills -s ncu-kernel-analyzer
```

List available skills without installing:

```bash
npx skills add imzhuhl/hl-skills --list
```

## Skills

| Skill | Description |
|-------|-------------|
| [ncu-kernel-analyzer](skills/ncu-kernel-analyzer/SKILL.md) | Analyze CUDA kernel performance from NVIDIA Nsight Compute (.ncu-rep) profiling reports. Identifies bottlenecks and provides actionable optimization recommendations. |

## Adding a New Skill

```bash
npx skills init <skill-name>
```

This creates a `<skill-name>/SKILL.md` template. Move it into the `skills/` directory, edit the frontmatter (`name`, `description`) and content, then commit.

```
skills/
└── your-skill-name/
    └── SKILL.md
```
