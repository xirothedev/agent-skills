# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Cursor, Copilot, etc.) when working with code in this repository.

## Repository Overview

A collection of packages and skills for NestJS best practices and AI agent assistance. Contains tooling for migrating NestJS projects to follow best practices, plus skills for Claude.ai and Claude Code.

## Packages

### nestjs-best-practices-build

Located in `packages/nestjs-best-practices-build/`, this package helps migrate NestJS projects to follow best practices from [vercel/nestjs-best-practices](https://github.com/vercel/nestjs-best-practices).

**Key features:**
- Migrates package manager from pnpm to bun 1.3.6
- Configures TypeScript for bun runtime
- Validates and extracts test files
- Supports configurable migration rules

**Usage:**
```bash
cd packages/nestjs-best-practices-build
bun run migrate              # Run full migration
bun run validate             # Validate project structure
bun run extract-tests        # Extract test files
bun run build                # Build the package
```

**Configuration:**
- `tsconfig.json` - Configured for bun (ESNext, bundler module resolution)
- `package.json` - Uses bun as package manager with `@types/bun` for TypeScript support
- `src/config.ts` - Migration rules and configuration

## Creating a New Skill

### Directory Structure

```
skills/
  {skill-name}/           # kebab-case directory name
    SKILL.md              # Required: skill definition
    scripts/              # Required: executable scripts
      {script-name}.sh    # Bash scripts (preferred)
  {skill-name}.zip        # Required: packaged for distribution
```

### Naming Conventions

- **Skill directory**: `kebab-case` (e.g., `vercel-deploy`, `log-monitor`)
- **SKILL.md**: Always uppercase, always this exact filename
- **Scripts**: `kebab-case.sh` (e.g., `deploy.sh`, `fetch-logs.sh`)
- **Zip file**: Must match directory name exactly: `{skill-name}.zip`

### SKILL.md Format

```markdown
---
name: {skill-name}
description: {One sentence describing when to use this skill. Include trigger phrases like "Deploy my app", "Check logs", etc.}
---

# {Skill Title}

{Brief description of what the skill does.}

## How It Works

{Numbered list explaining the skill's workflow}

## Usage

```bash
bash /mnt/skills/user/{skill-name}/scripts/{script}.sh [args]
```

**Arguments:**
- `arg1` - Description (defaults to X)

**Examples:**
{Show 2-3 common usage patterns}

## Output

{Show example output users will see}

## Present Results to User

{Template for how Claude should format results when presenting to users}

## Troubleshooting

{Common issues and solutions, especially network/permissions errors}
```

### Best Practices for Context Efficiency

Skills are loaded on-demand — only the skill name and description are loaded at startup. The full `SKILL.md` loads into context only when the agent decides the skill is relevant. To minimize context usage:

- **Keep SKILL.md under 500 lines** — put detailed reference material in separate files
- **Write specific descriptions** — helps the agent know exactly when to activate the skill
- **Use progressive disclosure** — reference supporting files that get read only when needed
- **Prefer scripts over inline code** — script execution doesn't consume context (only output does)
- **File references work one level deep** — link directly from SKILL.md to supporting files

### Script Requirements

- Use `#!/bin/bash` shebang
- Use `set -e` for fail-fast behavior
- Write status messages to stderr: `echo "Message" >&2`
- Write machine-readable output (JSON) to stdout
- Include a cleanup trap for temp files
- Reference the script path as `/mnt/skills/user/{skill-name}/scripts/{script}.sh`

## Working with NestJS Projects

When using this repository's tools with NestJS projects:

1. **Package Manager**: Use bun 1.3.6 for optimal performance
   ```bash
   bun install              # Install dependencies
   bun run build           # Build the project
   bun run start           # Start the application
   ```

2. **TypeScript Configuration**:
   - Target: `ES2017` or `ESNext`
   - Module: `ESNext`
   - Module resolution: `bundler`
   - Types: `@types/bun` instead of `@types/node`

3. **Best Practices**:
   - Use dependency injection properly
   - Implement proper error handling with exception filters
   - Use guards for authentication/authorization
   - Implement DTOs with validation pipes
   - Follow the official NestJS documentation patterns

4. **Testing**:
   - Use `@nestjs/testing` for unit tests
   - Write e2e tests for critical flows
   - Mock external dependencies properly

## Creating the Zip Package

After creating or updating a skill:

```bash
cd skills
zip -r {skill-name}.zip {skill-name}/
```

### End-User Installation

Document these two installation methods for users:

**Claude Code:**
```bash
cp -r skills/{skill-name} ~/.claude/skills/
```

**claude.ai:**
Add the skill to project knowledge or paste SKILL.md contents into the conversation.

If the skill requires network access, instruct users to add required domains at `claude.ai/settings/capabilities`.

## Troubleshooting

### NestJS Best Practices Migration

**Issue**: Module resolution errors after migration
- **Solution**: Ensure `tsconfig.json` has `"moduleResolution": "bundler"` and no `.js` extensions in imports

**Issue**: Type errors with bun runtime
- **Solution**: Install `@types/bun` matching your bun version, set types to `["bun-types"]`

**Issue**: pnpm commands still being used
- **Solution**: Update all CI/CD workflows and scripts to use `bun` instead of `pnpm`

### Skills Development

**Issue**: Skill not loading in Claude Code
- **Solution**: Verify SKILL.md has valid frontmatter with `name` and `description` fields

**Issue**: Scripts failing with permission errors
- **Solution**: Ensure scripts have execute permissions: `chmod +x scripts/*.sh`

### Common Issues

- **Network timeouts**: Check firewall rules and domain allowlists
- **Path issues**: Use absolute paths when referencing skill scripts
- **Context limits**: Keep SKILL.md under 500 lines, use supporting files for detailed docs