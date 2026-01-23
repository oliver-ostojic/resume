# Resume Portfolio Monorepo

This repository contains my portfolio projects as git submodules.

## Structure

- `logbook-writer/` - Crew scheduling system (Next.js + Fastify + Python MILP solver)
- `portfolio-site/` - Portfolio website (Next.js 16 + React 19 + Tailwind 4)

## Submodule Commands

```bash
# Initialize submodules after cloning
git submodule update --init --recursive

# Update submodules to latest
git submodule update --remote

# Pull latest for a specific submodule
cd logbook-writer && git pull origin main
```

## Per-Project Documentation

Each submodule has its own `CLAUDE.md` with detailed context:

- `logbook-writer/CLAUDE.md` - Full API, solver, and domain documentation
- `portfolio-site/demo/CLAUDE.md` - Next.js site structure and styling

## Quick Start

### Logbook Writer
```bash
cd logbook-writer
pnpm install --frozen-lockfile
pnpm dev  # Starts API on :4000 and web on :3000
```

### Portfolio Site
```bash
cd portfolio-site/demo
pnpm install
pnpm dev  # Starts on :3000
```

## Development Notes

- Both projects use pnpm as package manager
- Logbook-writer is a monorepo (Turborepo)
- Portfolio-site uses Next.js 16 with Tailwind CSS 4

## Current Focus

Building a unified portfolio experience by overlaying logbook-writer's glass UI components onto the portfolio site background.
