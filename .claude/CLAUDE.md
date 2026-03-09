# Claude React Native Knowledge Base

## Project Identity

This is a **knowledge base repository** for Claude agents — not a running React Native application. It provides structured guidance, skill definitions, and implementation patterns for React Native/Expo development teams using AI coding assistants.

**Consumers:** Claude Code sessions, Claude Agent SDK, Cursor, and other AI coding assistants
**Usage:** Referenced by target projects as a submodule, skill source, or agent configuration

## Repository Structure

```
agents/           5 AI agent definitions (system prompts for specialized roles)
commands/         7 workflow command templates (build, test, review, a11y)
docs/             Core reference documents (authentication, best practices)
examples/         4 complete code example collections
guides/           18 detailed implementation guides
skills/           15 executable skill definitions with trigger rules
  skill-rules.json  Skill routing configuration (keywords → skill files)
  development/    Frontend dev guidelines, glassmorphism
  testing/        Unit test generator, mocking patterns
  e2e-testing/    E2E test generator (Maestro/Detox)
  code-quality/   Code review checklist, refactoring, type organization
  debugging/      Bug fixing, crash handler, performance debugger
  ui-design/      Design system setup, accessible components
  converters/     Figma-to-RN, HTML-to-RN
```

## Tech Stack (Opinionated)

| Layer | Choice |
|-------|--------|
| Framework | React Native 0.76+ with Expo SDK 52+ |
| Language | TypeScript 5.x (strict mode) |
| Routing | Expo Router 4.x (file-based) or React Navigation 7.x |
| Client State | Redux Toolkit 2.x |
| Server State | TanStack Query 5.x |
| HTTP Client | Axios via centralized HttpService |
| Styling | NativeWind 4.x (Tailwind for RN) |
| UI Components | React Native Paper 5.x |
| Forms | React Hook Form 7.x + Zod 3.x |
| Storage | MMKV |
| E2E Testing | Detox + Maestro |
| Unit Testing | Jest + React Native Testing Library |

## Key Architectural Decisions

- **Skills vs Guides:** Skills are executable workflows with trigger conditions in `skill-rules.json`. Guides are passive reference documents. Never mix the two.
- **HttpService pattern:** All API calls go through a centralized Axios instance with interceptors — never use raw `fetch()` or direct `axios` calls in examples.
- **Dual navigation:** Both Expo Router and React Navigation are documented because real projects use both. Prefer Expo Router for new projects.
- **State separation:** Redux for auth/UI state, TanStack Query for server data. Never use Redux for API data caching.

## Content Conventions

- All skills MUST have YAML frontmatter with `name` and `description`
- Code examples use TypeScript with explicit types — never use `any`
- Styling examples use NativeWind `className` — never use `StyleSheet.create` as the primary approach
- Use `Pressable` — never `TouchableOpacity`
- Show preferred patterns labeled **GOOD** and anti-patterns labeled **BAD**
- Each pattern has ONE canonical location — other files link to it, never duplicate

## Common Workflows

### Adding a new guide
1. Create file in `guides/`
2. Add entry to `skill-rules.json` → `guides` section
3. Add cross-references from related files

### Adding a new skill
1. Create file in `skills/<category>/`
2. Add YAML frontmatter (`name`, `description`)
3. Add entry to `skill-rules.json` → `skills` section with `promptTriggers`
4. Verify all relative path links resolve

### Updating tech stack versions
1. Update `skills/development/frontend-dev-guidelines.md` tech stack table
2. Grep all files for old version references and update

## Important Files

- `skills/skill-rules.json` — Central routing config for all 15 skills and 18 guides
- `skills/development/frontend-dev-guidelines.md` — Master skill referencing all guides
- `agents/rn-frontend-developer.md` — Primary development agent
- `agents/rn-test-engineer.md` — Testing specialist agent (unit + E2E)
- `agents/rn-code-quality-reviewer.md` — Code review agent
- `agents/rn-ui-designer.md` — UI implementation agent
- `docs/AUTHENTICATION.md` — Auth architecture patterns
- `guides/data-fetching.md` — Core data layer architecture (HttpService)
- `guides/unit-testing.md` — Jest + RNTL testing patterns
- `guides/accessibility.md` — WCAG 2.1 AA compliance
- `examples/complete-examples.md` — Reference implementations
- `examples/testing-examples.md` — Test code examples

## Do Not

- Use `any` type in examples
- Use `StyleSheet.create` as primary styling (use NativeWind)
- Use `TouchableOpacity` (use `Pressable`)
- Duplicate code patterns across files (link to canonical source)
- Add skills without updating `skill-rules.json`
- Reference web tooling (Vite, Webpack, browser DevTools) — this is React Native
