# Working in this repo

Read this first, whether you are a person or an agent. It is the index: what is here, where the
rules live, and which of them a machine will catch for you.

Occasio is a multi-tenant event platform. Every event is a tenant whose **config data drives both
the look and the behaviour** — there is no per-event code, and a decision that hardcodes either
breaks the product rather than the style.

## The one rule that governs the others

> Every convention is machine-enforced, or it does not exist.

From [CONTRIBUTING.md](CONTRIBUTING.md). If you find yourself explaining a convention in review,
that convention is missing a rule. **See [Conventions a machine does not catch](#conventions-a-machine-does-not-catch)
for the ones that currently fail this test** — they are the ones you are most likely to get wrong.

## Code map

| Path             | Lines | What lives there                                                                                 | Depends on  |
| ---------------- | ----- | ------------------------------------------------------------------------------------------------ | ----------- |
| `packages/core`  | 72    | Branded ids (`TenantId`, `SessionId`) and nothing else                                           | —           |
| `packages/theme` | 1.7k  | `resolveTheme`: preset + seed colour → ~150 contrast-checked tokens. Pure, synchronous, no React | core        |
| `packages/data`  | 11k   | Row types, domain types, repositories, typed errors, the mock adapter, four fixture events       | core        |
| `packages/ui`    | 5.6k  | Themed primitives and components. No router, no data                                             | core, theme |
| `apps/mobile`    | 6.4k  | One Expo app → iOS, Android, web                                                                 | all         |
| `tools`          | 2.6k  | The merge gate, enforcement probes, visual capture                                               | —           |
| `test`           | 193   | Jest setup and stubs                                                                             | —           |

Imports flow one way and `eslint-plugin-boundaries` enforces it:
`core → theme → data → ui → features → app`.

### `packages/data` — the seam the whole architecture turns on

| File                       | What it is                                                                                                                |
| -------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| `rows.ts`                  | snake_case, mirrors the eventual Postgres tables **exactly**. Changing it is a schema decision                            |
| `domain.ts`                | camelCase types the app uses                                                                                              |
| `mappers.ts`               | The only meeting point, and one of two files where `as` is allowed                                                        |
| `repositories.ts`          | Every repository interface. `TenantDirectory` is the one cross-tenant surface                                             |
| `errors.ts`                | `NotFoundError`, `ForbiddenError`, `ValidationError` + type guards. `catch` gives `unknown`; the guards are the only door |
| `mock/adapter.ts`          | The mock, with latency, persistence and realtime                                                                          |
| `fixtures/`                | Four events that look nothing alike, on purpose                                                                           |
| `adapter.contract.test.ts` | One suite every adapter must pass (D29). The Supabase swap is proven here, not asserted                                   |

### `apps/mobile`

| Path            | What it is                                                                                  |
| --------------- | ------------------------------------------------------------------------------------------- |
| `app/`          | expo-router routes. **Thin adapters only**: read params, compose providers, render a screen |
| `src/features/` | Screens and their controllers. Pure — props in, JSX out, no router hooks                    |
| `src/tenant/`   | Which event is this? Platform-split resolution (`.web.ts` / `.ts`)                          |
| `src/auth/`     | Who is signed in. Carries identity and never authority                                      |
| `src/access/`   | `useRole`, `RoleGate`, `PlatformGate`. UX gating, not enforcement                           |
| `src/data/`     | Query keys, hooks, the adapter provider                                                     |

## Where the rules live

| Source                                 | Holds                                                                         |
| -------------------------------------- | ----------------------------------------------------------------------------- |
| [CONTRIBUTING.md](CONTRIBUTING.md)     | The meta-rule, the six rules you will hit, component guidance                 |
| [docs/decisions.md](docs/decisions.md) | D1–D43, frozen. Change one by arguing in an issue, never by working around it |
| [docs/adr/](docs/adr/)                 | Seven records of _why_, and what the alternative would have cost              |
| [docs/workflow.md](docs/workflow.md)   | Branching, review, the four gates, releases                                   |
| [.coderabbit.yaml](.coderabbit.yaml)   | Ten per-directory review instructions                                         |
| `eslint.config.mjs`                    | Layer boundaries, banned imports, banned casts, banned literals               |
| `tools/enforcement/`                   | Probes proving each architectural rule actually fires                         |

## What a machine catches

Run `npm run verify` before pushing. It is the same five gates CI runs.

- **`tsc`** at maximum strictness — `strictTypeChecked`, `noUncheckedIndexedAccess`,
  `exactOptionalPropertyTypes`, `verbatimModuleSyntax`.
- **ESLint** — `strictTypeChecked` + `stylisticTypeChecked`, layer boundaries, no `@supabase/*`
  outside its adapter, no `expo-router` in `ui/`, no `as` outside `mappers.ts` and `ids.ts`, no
  literal colours or spacing in styles.
- **`verify:enforcement`** — ten probes proving those rules fire. Two rules in this repo were
  configured, looked correct, and did nothing until a probe caught them.
- **Jest** — `unit`, `components` (jsdom + react-native-web), `contracts`.

## Conventions a machine does not catch

These are real rules, currently enforced only by review. They are listed because they are what
reviewers actually catch, and by the meta-rule above each one is a missing lint rule.

- **Route files are thin adapters.** Data fetching, error wording and input validation belong in
  `src/features/`. This is the single most frequently missed rule in this repo.
- **A test must be able to fail.** See the `mutation-check` skill. Assertions that hold
  regardless of the code under test are treated as defects, not as coverage.
- **A comment must match the code.** A comment asserting behaviour the code does not have is
  worse than no comment — it stops the next reader looking.
- **File naming**: `PascalCase.tsx` for a file exporting a component, `camelCase.ts` otherwise.
  `tools/` currently breaks this (`check-reviewed.mjs` beside `reviewGate.mjs`) and nothing
  enforces it.
- **Model state as discriminated unions**, never several booleans that can contradict.
- **Every repository method takes `tenantId` first**, even where it could be inferred. That
  mirrors row-level security and is what stops cross-tenant leaks.

## Known gaps

Worth knowing before you trust a green run:

- **No `eslint-plugin-react-hooks`.** Neither `rules-of-hooks` nor `exhaustive-deps` runs. Expo
  ships these in `eslint-config-expo`; this repo hand-rolls its ESLint config and omits them.
- **No enforcement behind the access gates.** D4 puts the backend in a later phase, so there is
  no RLS yet. `RoleGate` decides what is _offered_; nothing yet decides what is _allowed_.
- **The Claude fallback reviewer does not work** (#142). Parked.

## Skills

`.claude/skills/` holds task-shaped guides for the three things most often done wrong here:

| Skill               | Use it when                                                  |
| ------------------- | ------------------------------------------------------------ |
| `mutation-check`    | You have written a test and want to know whether it can fail |
| `repository-method` | Adding or changing anything in `packages/data`               |
| `themed-component`  | Adding a component to `packages/ui`                          |
