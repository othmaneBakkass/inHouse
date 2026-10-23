# AGENTS.md

## Rules

- Never skip validation, error handling, or security.
- Open pull requests; never merge.
- Before coding, read `Architecture.md` and the nearest similar use case. Read other docs only as needed.
- Write the least code possible. Reuse existing code, standard library, platform features, and dependencies.
- Follow existing patterns. Keep one use case per folder and one responsibility per file. Move code to `shared/` only when a second use case needs it. Avoid unnecessary layers.
- Keep core (`execute.ts`, `errors.ts`, `shared/`) framework-agnostic: plain functions and data, no classes, hidden state, or DI containers. Import only plain TypeScript and `common/`. Pass external dependencies explicitly through `execute.ts` and wire them in `route.ts`. Routes must not access the database client or platform directly.
- Document intentional shortcuts:
  ```ts
  // debt: <shortcut>. Limit: <when it breaks>. Fix: <alternative>.
  ```

## Conventions

- Files and folders: `kebab-case`.
- Functions, variables, constants, and Zod schemas: `camelCase`.
- Types: `PascalCase`; prefer `type` over `interface`.
- Prefer one object argument. Name its type `<FunctionName>Args`.
- Give every function a named `<FunctionName>Output` type, an explicit return type, and use `satisfies` on returned values.

## Agent skills

- **Issues:** GitHub Issues via `gh`. See `docs/agents/issue-tracker.md`.
- **Triage:** Use the five canonical roles and default labels. See `docs/agents/triage-labels.md`.
- **Domain docs:** One root `GLOSSARY.md` and `docs/adr/`. See `docs/agents/domain.md`.
- **Architecture:** `Architecture.md` at the repository root.