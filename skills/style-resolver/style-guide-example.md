# Code Style Guide

## Source
- Language: TypeScript (strict)
- Framework: React 18 + Express 4
- Based on: 5 source files + .eslintrc.json + package.json

## Formatting
- Indentation: 2 spaces, no tabs
- Line width: 100 characters
- Line endings: LF
- Trailing whitespace: not allowed
- Final newline: required
- Bracket style: same-line opening brace

## Quotes & Semicolons
- Quotes: single (`'use strict'`)
- JSX attributes: double (`className="btn"`)
- Semicolons: required
- Trailing commas: required in multiline arrays/objects

## Naming Conventions
| Element | Convention | Example |
|---------|-----------|---------|
| Files | kebab-case | user-service.ts |
| React components | PascalCase + .tsx | UserProfile.tsx |
| Test files | *.test.ts or *.spec.ts | user-service.test.ts |
| Directories | kebab-case | user-profile/ |
| Variables / methods | camelCase | userName, getAge() |
| Functions | camelCase | fetchUserById() |
| Classes / React components | PascalCase | UserController |
| Interfaces / Types | PascalCase, no I-prefix | UserData, ProductProps |
| Constants (file-level) | UPPER_SNAKE | API_BASE_URL |
| Private class members | camelCase, no underscore | private cache: Map |
| Boolean variables | is/has/should prefix | isLoading, hasError |

## Imports
- External packages first, then internal modules, blank line between groups
- Preferred: named imports over default imports
- Avoid barrel imports from `index.ts` — import from specific file
- Order: React / libs / utils / components / styles

## Functions
- Prefer arrow functions for callbacks, `function` keyword for top-level exports
- Max 30 lines per function; extract helpers otherwise
- Destructure object parameters instead of positional args (> 2 params)
- Explicit return type on exported functions

## React Components
- Functional components with hooks, no class components
- Props interface: named `ComponentNameProps`, in same file or adjacent types file
- Hooks: declare at top of component body, before any logic
- Styling: CSS Modules (`component-name.module.css`) or Tailwind utility classes
- Default export for the component, named exports for helpers

## Express
- Route handler: `async (req, res, next) => { ... }` with `try/catch(next)`
- Request validation: at route entry, before any business logic
- Middleware: exported as named function, imported explicitly per route
- Error responses: `{ error: string, details?: string }`
- File naming: `thing.route.ts` for route definitions, `thing.controller.ts` for handlers

## Error Handling
- Use `try/catch` at async boundaries; never let a Promise reject silently
- Custom error classes extending `Error` for domain errors (e.g. `ValidationError`)
- Catch block: narrow type via `instanceof`, not string matching
- Log errors with structured context (userId, requestId, timestamp)

## Comments
- JSDoc: required on all exported functions and classes
- Inline comments (`//`): explain "why", not "what" — and only when non-obvious
- Markers: `FIXME:` for known bugs, `TODO:` for planned work
- No commented-out code; use git history for old code

## Testing
- Framework: Vitest (preferred) or Jest
- Test files: co-located with source (`foo.ts` → `foo.test.ts`)
- Naming: `describe('ClassName', () => { it('should do X', () => {...}) })`
- Mocks: prefer `vi.fn()` / `jest.fn()` over manual mock objects
- Coverage target: 80% statements, 100% for critical paths

## Git Commits
- Format: `type(scope): short description` (conventional commits)
- Types: feat, fix, refactor, test, docs, chore
- Scope: component or module name in kebab-case
- Description: imperative mood, under 72 chars, no trailing period


<!--
  AGENT NOTES — how to use this example
  =====================================
  1. This file shows what a COMPLETE style_guide.md looks like.
     Your output for the real project must match this structure section
     by section. Do NOT skip sections even if you have no data for them
     yet — write "N/A — <reason>" instead.

  2. Every concrete value in this example (indentation, quote style, etc.)
     MUST be replaced with the actual project's conventions. The example
     values are just to show what filled-in entries look like.

  3. Annotate where each rule came from:
     - ["from code": file.ts:42] — detected by reading source files
     - ["from config": .eslintrc.json] — found in a config file
     - ["from convention"] — community best practice for that language/framework
     - ["from user"] — confirmed by user during interaction

  4. When the real project has multiple frameworks (e.g. React + Express),
     list each under its own subsection. When there's only one, collapse
     into the general sections.

  5. If you cannot determine a convention and can't find a community standard,
     DO NOT INVENT ONE. Leave the entry empty with a `?` and ask the user.
-->
