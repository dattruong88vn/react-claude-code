# CLAUDE.md

Vite + React 19 + TypeScript app, built in phases as a learning project.
Each phase is documented in [docs/SETUP-GUIDE.md](docs/SETUP-GUIDE.md) (Vietnamese).
When finishing a phase, update its section in SETUP-GUIDE.md in the same commit as the code.

## Commands

Use **pnpm** only. Importing a package missing from `package.json` fails by design: add the dependency, don't work around it.

- `pnpm dev` | `pnpm typecheck` | `pnpm lint` | `pnpm format` | `pnpm build`
- Done = `pnpm typecheck && pnpm lint && pnpm build` all clean. No test framework yet.

## Architecture

- `src/app`: providers, router.
- `src/features/<name>/{components,hooks,api,model}`: code grouped by feature.
- `src/shared/ui`: design system. `src/shared/{lib,hooks}`: shared utilities.
- `src/tokens`: design tokens.
- NEVER import from `features` into `shared`.

## Conventions

- Function components, named exports. No `any`, no `@ts-ignore`.
- `noUncheckedIndexedAccess` is on: `arr[0]` is `T | undefined`, narrow it.
- Do not add libraries without asking.

## Guardrails

- Pre-commit runs `pnpm lint-staged` then `pnpm typecheck`. If a commit is blocked, fix the errors. Never use `--no-verify`.
- In `eslint.config.js`, `eslint-config-prettier` must stay the LAST entry.

## Git

- Branch: `<type>/<short-description>`, type is one of phase | feat | fix | chore | docs | explore. Lowercase, hyphens, ASCII only.
- Commits: Conventional Commits (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`).
- One purpose per branch, small PRs. No commits directly to `main`. No `git push --force`.

## Don'ts

- Do not edit `.env*`. Do not change dependencies without asking.
