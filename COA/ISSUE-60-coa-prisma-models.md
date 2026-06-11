# Issue 60: Unified CoA — Prisma Model

Date: 2026-06-11 | ID: 60 | Parent story: #12 | Status: In progress | Branch: `fino-60-coa-data-models`

> **Note:** This issue was redesigned. The original 3-table approach (`AccountType` / `AccountHead` / `SubAccount`) is being replaced by a single self-referential `Account` table. Earlier notes in this file have been superseded.

## Summary

Replace the 3-table CoA design (`AccountType` / `AccountHead` / `SubAccount`) with a single self-referential `Account` table that supports arbitrary depth. Each row stores the raw `seed` the user types; the padded `displayCode` is denormalized on the same row and computed by `CoaService` from the seed chain, sibling-level widths, and a company-wide `coaDigits` config. Scope of this PR is **database only** — schema edit + `prisma migrate dev`, no service logic.

## Root cause

The current 3-table design hardcodes hierarchy depth at exactly three levels and stores already-padded codes as the source of truth on every row. Two consequences fall out of that:

1. Adding a 4th level requires a new table and migration.
2. Changing the company-wide digit width (e.g. 10 → 12) requires rewriting every stored code and would break any FK that referenced a code value.

The new design separates **user input** (`seed`) from **display** (`displayCode`). Depth becomes a property of data, not schema. Digit width becomes a config value, not a structural decision.

## Decision

```
## Decision: Unified CoA — self-referential Account table with denormalized displayCode

Options:
  A — Self-referential Account + denormalized displayCode (the spec). Stores raw
      seed; service computes and writes displayCode; DB enforces (parentId, seed)
      and (displayCode) uniqueness.
      Chosen.
  B — Self-referential Account, compute displayCode on every read (no storage).
      Rejected: every voucher-screen list walks the tree per row; loses DB-level
      uniqueness guarantee on the user-visible code; weakens the safety net.
  C — Materialized path column storing full ancestor-id chain per row.
      Rejected: every re-parent or structural edit cascades a write to every
      descendant; trades a rare cost (read) for a frequent one (write) in the
      wrong direction. Does not actually solve the displayCode problem.

Chosen: A.
Reasoning: Only option that keeps both wins — configurable digit width AND
DB-enforced global uniqueness on the visible code. Performance cost (recompute
on widening) is bounded and localized to one service. Read cost is O(1).
Trade-off accepted: schema cannot enforce "only CoaService writes displayCode" —
that rule lives in app code. We accept this because the (displayCode) unique
constraint catches the failure mode that actually matters (duplicates from
races, bugs, or shortcuts).
Risks:
  - Cascading recompute on widening could touch many rows under load.
    Mitigation: handled in service ticket; transactional, scoped to subtree.
  - Concurrent inserts under the same parent could race on width computation.
    Mitigation: (parentId, seed) unique constraint catches duplicates at DB.
  - Self-referential FK with onDelete: Restrict means parents with children
    cannot be deleted. By design — soft delete via isActive.
```

## Target schema (sketch)

```prisma
enum AccountType {
  ASSETS
  LIABILITIES
  EQUITY_NET_ASSETS
  INCOME
  EXPENSES
}

model Account {
  id            String      @id @default(cuid())
  seed          String
  displayCode   String      @unique
  title         String
  definition    String?
  accountType   AccountType
  parentId      String?
  parent        Account?    @relation("AccountTree", fields: [parentId], references: [id], onDelete: Restrict)
  children      Account[]   @relation("AccountTree")
  isPostable    Boolean
  isActive      Boolean     @default(true)
  createdBy     String
  createdByUser User        @relation(fields: [createdBy], references: [id], onDelete: Restrict)
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt

  @@index([parentId])
  @@index([accountType, isPostable])
  @@unique([parentId, seed])
  @@map("accounts")
}

model CompanyConfig {
  id        String   @id @default(cuid())
  coaDigits Int      @default(10)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("company_config")
}
```

And on `User`, remove the three old back-relations and add:

```prisma
accountsCreated Account[]
```

## Key learnings

- **Self-relations in Prisma require an explicit `@relation("Name", ...)` label on both ends** — even when there's only one such relation. Prisma can't disambiguate two `Account`-typed fields without it. The label is metadata for the Prisma parser only; it never appears in the database.
- **Navigation fields (`parent`, `children`) are not database columns.** Only `parentId` is a real column. `parent` and `children` are TypeScript accessors that Prisma generates so you can write `account.parent` / `account.children` in code; both compile down to queries against the same `parentId` column.
- **DB constraints are the safety net for app-owned invariants.** Even though `CoaService` is the only thing that "should" write `displayCode`, the `@unique` constraint catches the failure modes that matter: races between concurrent inserts, bugs in `computeDisplayCode()`, and future one-off scripts that bypass the service.
- **Recursive CTEs** are the right tool for ancestor-chain walks in a self-referential table. The walk happens inside Postgres in a single query instead of N round trips from app code. This is what makes the self-referential design competitive on reads against materialized-path alternatives.
- **Storing the raw `seed` rather than the padded code** is what makes `coaDigits` a config value instead of a schema decision. If we stored only the padded code, changing the digit width would lose the source of truth. The denormalized `displayCode` is a read-optimization on top of `seed`, not a replacement for it.

## Resources

- [Prisma — Self-relations (one-to-many)](https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/self-relations#one-to-many-self-relations) — exact pattern for `parentId? → Account`. Documents the required `@relation("Name", ...)` label on both ends.
- [Prisma — Composite unique constraints](https://www.prisma.io/docs/orm/prisma-schema/data-model/indexes#unique) — syntax for `@@unique([parentId, seed])`.
- [Prisma — Referential actions](https://www.prisma.io/docs/orm/prisma-schema/data-model/relations/referential-actions) — `onDelete: Restrict` semantics on self-FK.
- [Prisma — Models & enums](https://www.prisma.io/docs/orm/prisma-schema/data-model/models) — general model + enum syntax reference.
- [PostgreSQL — Recursive CTEs](https://www.postgresql.org/docs/current/queries-with.html#QUERIES-WITH-RECURSIVE) — used later in `CoaService` to walk ancestor chains in a single query.
- Internal: existing `AccountHead` / `SubAccount` models in `backend/prisma/schema.prisma` (lines 65–96) — mirror their `createdBy` + `@relation(..., onDelete: Restrict)` pattern on the new `Account` model.
- Internal: commit `8b344b7` — the commit that introduced the 3-table design this PR reverses. Useful as historical context.

## Acceptance criteria

- [ ] Old `AccountType` / `AccountHead` / `SubAccount` models removed from `schema.prisma`.
- [ ] Old `AccountTypeName` enum removed.
- [ ] Old back-relations on `User` (`accountTypesCreated`, `accountHeadsCreated`, `subAccountsCreated`) removed.
- [ ] New `AccountType` **enum** added with values `ASSETS`, `LIABILITIES`, `EQUITY_NET_ASSETS`, `INCOME`, `EXPENSES`.
- [ ] New `Account` model added with all fields per spec (`id`, `seed`, `displayCode`, `title`, `definition?`, `accountType`, `parentId?`, `isPostable`, `isActive`, `createdBy`, `createdAt`, `updatedAt`).
- [ ] Self-relation on `Account` uses `@relation("AccountTree", ...)` label on both ends with `onDelete: Restrict`.
- [ ] `accountsCreated Account[]` back-relation added on `User`.
- [ ] Indexes: `@@index([parentId])`, `@@index([accountType, isPostable])`, `@@unique([parentId, seed])`, `displayCode @unique`.
- [ ] New `CompanyConfig` model added with `coaDigits Int @default(10)`.
- [ ] `prisma migrate dev` runs cleanly against a fresh database.
- [ ] Generated migration SQL reviewed — DROP order is sane, FK constraints intact, no unintended data loss surprises beyond the expected destructive drop.

## Implementation notes

- **Destructive migration accepted.** Existing seeded CoA rows will be dropped; real data will be imported in a separate later issue.
- **`createdBy` is a full Prisma relation**, not just a string column. Mirror the existing pattern: `createdBy String` + `createdByUser User @relation(fields: [createdBy], references: [id], onDelete: Restrict)`.
- **Eyeball the enum churn.** Dropping the old `AccountType` model AND introducing a new `AccountType` enum (same name) in one migration — Prisma handles it, but verify the generated SQL is DROP-then-CREATE in the right order.
- **`@@unique([parentId, seed])` covers prefix lookups on `parentId`** — a separate `@@index([parentId])` is technically redundant for that prefix. Spec asks for both; keep both for clarity and future use. Cost is negligible.
- **`CompanyConfig` is a normal model with no "singleton" enforcement at the DB level.** The "only one row" rule lives in the service layer for now (acceptable — multi-tenant is deferred).
- **Out of scope for this PR:** `computeDisplayCode()` helper, ancestor-walk helper, unit tests, lazy root creation, any service logic. Those land in follow-up tickets.
