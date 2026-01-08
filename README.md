# Claude React Native

Claude Code configuration for React Native mobile development. This is a framework-specific submodule designed to be used alongside `claude-base` (shared/generic config).

## Contents

### Agents
- **frontend-developer.md** - Mobile screen development, API integration, Detox/Jest E2E testing
- **frontend-error-fixer.md** - Debug React Native errors (build and runtime)

### Skills
- **frontend-dev-guidelines/** - React Native development patterns with 21 resource files:
  - Figma to React Native conversion
  - NativeWind (Tailwind CSS for RN) styling
  - React Native Paper components
  - React Navigation 6.x routing
  - Redux Toolkit state management
  - Axios data fetching
  - Detox E2E testing

## Tech Stack

- React Native 0.73+
- React Navigation 6.x
- Redux Toolkit
- NativeWind (Tailwind CSS for RN)
- React Native Paper (Material Design)
- React Hook Form + Zod
- Axios
- Jest + React Native Testing Library
- Detox (E2E)

## Usage

Add as a git submodule to your project:

```bash
git submodule add https://github.com/potentialInc/claude-react-native.git .claude/react-native
```

### Project Structure

```
.claude/
├── base/           # Generic/shared config (git submodule)
├── nestjs/         # NestJS-specific config (git submodule) - if using NestJS backend
├── react-native/   # This repo (git submodule)
└── settings.json
```

## Related Repos

- [claude-base](https://github.com/potentialInc/claude-base) - Shared/generic Claude Code config
- [claude-nestjs](https://github.com/potentialInc/claude-nestjs) - NestJS-specific Claude Code config
- [claude-react](https://github.com/potentialInc/claude-react) - React web-specific Claude Code config
- [claude-django](https://github.com/potentialInc/claude-django) - Django-specific Claude Code config
