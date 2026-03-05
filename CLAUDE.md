# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Turborepo monorepo with two Next.js applications and shared packages. Package manager is pnpm.

## Monorepo Structure

- `apps/web` - Next.js app running on port 3000
- `apps/docs` - Next.js app running on port 3001
- `packages/ui` - Shared React component library
- `packages/biome-config` - Shared Biome linting configurations
- `packages/typescript-config` - Shared TypeScript configurations

## Common Commands

### Development
```bash
# Run all apps in dev mode
pnpm dev

# Run specific app
pnpm dev --filter=web
pnpm dev --filter=docs
```

### Building
```bash
# Build all apps and packages
pnpm build

# Build specific app
pnpm build --filter=web
```

### Linting & Type Checking
```bash
# Lint all packages
pnpm lint

# Type check all packages
pnpm check-types

# Format code with Prettier
pnpm format
```

## Code Style

- Linting: Biome (configured in root biome.json)
- Formatting: Prettier for markdown/TypeScript, Biome for code
- Biome uses tabs (width 2) and line width 120
- TypeScript strict mode enabled

## Workspace Dependencies

Packages use workspace protocol (`workspace:*` or `workspace:^`) to reference internal packages. When adding dependencies between workspace packages, use this format in package.json.

## Next.js Apps

Both apps use:
- Next.js 16.1.5
- React 19.2.0
- App Router (default for Next.js 16)
- TypeScript 5.9.2

The `@repo/ui` package is imported directly via workspace dependencies.

## Learning Roadmap

The `.roadmap/` folder contains a 6-month AI agents developer learning plan:
- `.roadmap/README.md` - High-level blueprint and timeline
- `.roadmap/roadmap-level-{0-8}-*.md` - Detailed per-level workflow guides with daily steps, code examples, and checkpoints

Goal: Become job-ready as an AI Engineer specializing in workflow automation (80%) and chatbots (20%). Each level builds a small project to learn concepts hands-on.
