---
name: repository-method
description: Add or change anything in packages/data — a repository method, a row type, a domain type, a fixture, or the mock adapter. Use when touching rows.ts, domain.ts, repositories.ts, mappers.ts or mock/adapter.ts, and when a change needs a new cross-tenant lookup.
---

# Changing the data layer

This layer exists so the backend can be swapped without moving a component (D5, D29). Every rule
below protects that, or protects tenant isolation.

## The shape of a method

```ts
readonly list: (
  tenantId: TenantId,          // always first, even where it could be inferred
  query: SessionQuery,
  page?: PageRequest,          // last, optional, wherever a Page is returned
) => Promise<Page<Session>>;   // async and paged, even where the mock answers from an array
```

- **`tenantId` first, always.** It mirrors row-level security, and it is what stops a wedding's
  schedule reaching a conference. A method that could infer it still takes it.
- **Async, cursor-paginated, error-typed** even where the mock ignores all three. A method
  returning `Promise<Session[]>` compiles, works against forty fixture rows, and is a rewrite of
  every caller the day the list is longer than a screen.
- **Throw typed errors** — `NotFoundError`, `ForbiddenError`, `ValidationError`. Never a
  `{ data, error }` envelope: `catch` gives `unknown` and the exported type guards are the only
  narrowing allowed.

`repositories.test.ts` pins the inventory by name and count. It will fail when you add a method —
that is the gate working. Update it deliberately.

## Rows, domain and mappers

- `rows.ts` is **snake_case and mirrors the eventual Postgres tables exactly.** Editing it is a
  schema decision; it should arrive with the migration and the policy it implies.
- `domain.ts` is camelCase.
- `mappers.ts` is the only meeting point and one of two files where `as` is permitted.
- **Never spread an externally-supplied object into a domain type.** `{ ...input }` copies own
  enumerable properties, so an object carrying `role: 'owner'` satisfies a structural type and
  the extra field travels. Copy named fields. This defect has been found twice in this repo.

## Cross-tenant surfaces are gated on purpose

`TenantDirectory` is the only type whose methods do not start with a `TenantId`, and
`repositories.test.ts` pins its methods **by name** so a new one has to be a decision somebody
writes down.

Before adding one, answer in the PR: what makes this impossible to scope to a tenant? `bySlug`
and `byJoinCode` qualify because they are how a caller *obtains* a tenant. A convenience listing
does not.

## The mock is not a stub

`mock/adapter.ts` is the reference implementation and the thing the contract suite proves. It:

- resolves permissions **per delivery**, not at subscribe time — a demoted moderator must stop
  receiving unapproved bodies without resubscribing;
- serialises writes onto a chain, so two read-modify-writes in flight cannot lose one;
- persists **before** it believes — assigning state and then writing leaves the adapter reporting
  something nothing has stored.

## The contract suite is where it is proven

Add cases to `adapter.contract.test.ts`, not only to a mock-specific test. It runs against every
adapter, so a rule asserted there is a property of the swap rather than of one implementation.

Use the exported type guards, never `instanceof` — a second adapter constructs its own error
instances. And assert the call *rejected*: a `try/catch` whose subject quietly resolves leaves
the `catch` unreached and the test green.

## Before opening the PR

- `npm run verify`
- Run `mutation-check` on every new assertion.
- If you touched a fixture, check whether a test now passes because the fixture cannot reach the
  branch it claims to cover.
