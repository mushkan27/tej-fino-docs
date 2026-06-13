# Issue 62: CoA Backend API — Create + List with DisplayCodeService

Date: 2026-06-11 | ID: 62 | Parent story: #12 | Status: In progress | Branch: `fino-62-coa-api-backend`

## Summary

Build the NestJS backend for the Chart of Accounts: `GET /api/accounts` (flat list, reads denormalized `displayCode` straight from the column) and `POST /api/accounts` (single endpoint that creates root, head, or sub). The server owns every rule — seed validation, parent-compatibility checks, the head/sub formulas, and the cascade-on-widening recompute — and writes `displayCode` itself. The client only ever sends a raw `seed`.

## Root cause

This is missing-feature work, not a bug. The `Account` Prisma model and `(parentId, seed)` + `displayCode @unique` constraints landed in commit `f1c3d7c`, plus a partial unique index `one_root_per_account_type ON accounts(accountType) WHERE parentId IS NULL` in the same migration. What's missing is the HTTP surface, the transactional `CoaService`, and the `DisplayCodeService` seam.

Computing `displayCode` on the client is structurally wrong:

1. **Global state the client can't see.** The padded code embeds the max seed length of every sibling group along the ancestor chain. A client would have to fetch every sibling at every level just to pad correctly — and a concurrent insert that widens any level would silently invalidate the answer.
2. **Cascade isn't a client operation.** When a new sibling widens its level, every existing descendant of the parent has its code shift. That recompute has to land inside the same transaction as the insert. The client isn't in the transaction.
3. **The server has to recompute anyway** to validate a client-sent code — so accepting one from the client is pure overhead.

## Decision

```
## Decision: Subtree recompute via iterative BFS in Prisma (not raw CTE)
Date: 2026-06-11 | Context: CoaService cascade-on-widening needs a subtree load+update strategy
Options:
  A — Recursive CTE + bulk UPDATE via $queryRaw. Rejected: scope and raw-SQL silo
      not justified for the COA domain (shallow trees, infrequent cascades).
  B — Iterative BFS with Prisma findMany per level + per-row update inside the
      same transaction. CHOSEN.
  C — Materialized path column. Rejected: explicitly out of scope; adds a new
      column, new migration, and a new invariant to maintain for no current win.
Chosen: B.
Reasoning: COA trees are shallow by domain (root → head → sub, ~3–4 levels).
  Cascade is a rare event (widening only). Team-Prisma idioms keep the cascade
  code readable to anyone on the team. The DisplayCodeService seam means the
  implementation can be swapped to A later without touching CoaService or the
  controller if profiling ever shows it.
Trade-off accepted: O(depth) round-trips and O(subtree-size) per-row UPDATEs per
  cascade. Acceptable because cascade is rare and trees are shallow.
Risks: pathological wide/deep trees → slow cascade. Mitigation: profile if it
  ever surfaces; swap implementation behind the seam.

## Sub-decision: Transaction propagation
Chosen: pass `tx` explicitly from CoaService.create into
  DisplayCodeService.recomputeSubtree(rootId, tx).
Rejected: nestjs-cls / AsyncLocalStorage — overkill for two services in one
  module.

## Sub-decision: coaDigits source for this sprint
Chosen: module-level constant `coaDigits = 10` inside DisplayCodeService.
Rejected (deferred): reading from CompanyConfig. The model exists in the schema
  but is not read this sprint. When runtime config lands, only DisplayCodeService
  changes; callers don't notice.

## Sub-decision: Uniqueness — pre-check vs catch-and-translate
Chosen: catch P2002 inside the transaction and translate by meta.target:
  - one_root_per_account_type → 409 "root already exists for this type"
  - displayCode               → 409 "display code collision"
  - parentId,seed             → 409 "sibling seed already taken"
Rejected: pre-check via findUnique before insert. Pre-checks are racy; the DB
  is the only race-free source of truth. App-layer pre-checks for friendly
  errors are OK only as belt-and-suspenders, never as the sole guard.
```

## Key learnings

- **The seam earns its keep.** `DisplayCodeService.for(account)`, `forMany(accounts)`, and `recomputeSubtree(rootId, tx)` is the only surface callers may touch. This sprint, `for`/`forMany` are trivial column reads and `recomputeSubtree` does the cascade. Next sprint can swap any of them — column read → on-demand compute, BFS → CTE — without changing a single caller. The pure padding helper stays private to the seam.
- **Head vs sub formulas share inputs, differ on filler placement.**
  - Head: `(paddedAncestorChain + paddedSelf).padEnd(coaDigits, '0')` — chain (root → parent) + self, then right-fill with zeros.
  - Sub:  `paddedAncestorChain + '0'.repeat(middle) + paddedSelf` where `middle = coaDigits − paddedAncestorChain.length − paddedSelf.length`. Self appears **once** at the trailing leaf position; its chain slot is absorbed into the filler. `middle < 0` → `422`.
  Same building blocks, different placement of the filler — at the end for heads, in the middle for subs.
- **Widening shifts the whole subtree, not just direct siblings.** Every descendant's code embeds the padded chain, so widening level *N* shifts every code at level ≥ *N* in the parent's subtree. Recursion is required, not optional.
- **Rejection paths must come from the DoD checklist, not memory.** Keep the ticket open while implementing — auth (401), DTO (400), service input rules (400), parent lookup (404), parent compatibility (400), root-exists pre-check (409), overflow (422), DB unique races (409). Cheap-first ordering.
- **`P2002.meta.target` distinguishes the three uniqueness failures.** Match by constraint name for the partial index, match by column array for `(parentId, seed)`, match by single column for `displayCode`.

## Resources

### Official docs
- [Prisma — Interactive transactions](https://www.prisma.io/docs/orm/prisma-client/queries/transactions#interactive-transactions) — `$transaction(async (tx) => ...)`. Pass `tx` into `DisplayCodeService.recomputeSubtree` so the cascade is atomic with the insert.
- [Prisma — `PrismaClientKnownRequestError` (P2002)](https://www.prisma.io/docs/orm/reference/error-reference#p2002) — `code === 'P2002'` + `meta.target` is how we distinguish the three uniqueness failures.
- [NestJS — Prisma recipe](https://docs.nestjs.com/recipes/prisma) — canonical `PrismaService` injection pattern; matches `backend/src/prisma/`.
- [NestJS — Validation pipes / class-validator](https://docs.nestjs.com/techniques/validation) — `400` on bad DTO shape for free. Reserve service-layer errors for business rules.
- [NestJS — Built-in HTTP exceptions](https://docs.nestjs.com/exception-filters#built-in-http-exceptions) — `BadRequestException` (400), `NotFoundException` (404), `ConflictException` (409), `UnprocessableEntityException` (422).

### Real-world examples
- [wanago.io — Recursive relationships with Prisma + PostgreSQL](https://wanago.io/2023/12/11/api-nestjs-sql-recursive-relationships-prisma-postgresql/) — walks through self-referential trees in Nest/Prisma. Useful reference even though we chose BFS over CTE for the cascade.
- [Masoud — Managing Prisma transactions in NestJS](https://masoudx.medium.com/stop-passing-the-hot-potato-managing-prisma-transactions-in-nestjs-8f30eeb5bb54) — the "pass `tx` explicitly vs CLS" trade-off. Confirms explicit passing is the right call at this scale.

### Internal
- Commit `f1c3d7c` — `feat(db): add COA data models and finance role enums` (the `Account` model + enums this issue builds on).
- Migration `20260611055654_unified_coa_account_model` — schema + partial unique `one_root_per_account_type`.
- [COA/ISSUE-60-coa-prisma-models.md](./ISSUE-60-coa-prisma-models.md) — prior task; explains the model design and why we landed on a single self-referential table.
- [COA/coa tables rules.md](./coa%20tables%20rules.md), [COA/newcoarules.md](./newcoarules.md) — domain rules reference.

## Acceptance criteria

- [ ] One root per `accountType` enum value, ever (enforced by `one_root_per_account_type` partial unique + app-layer pre-check).
- [ ] Rejects postable parent / type mismatch with parent / sibling seed clash / global `displayCode` collision / padded-chain overflow / all-zero sub seed — each at the correct layer with the correct HTTP code.
- [ ] `GET /api/accounts` returns `displayCode` on every row by reading the column. No tree walk per request.
- [ ] On insert, if the new seed widens its sibling level, `CoaService` recomputes `displayCode` for the parent's entire subtree (parent + all descendants) inside one transaction. Silent — no extra fields in the response.
- [ ] On a non-widening insert, only the new row's `displayCode` is computed and written.
- [ ] All display-code reads/writes go through `DisplayCodeService`. The pure padding helper is private to that file and never imported by other modules.
- [ ] `displayCode` is never written by hand or via the controller. Only `CoaService` create paths invoke `DisplayCodeService.recomputeSubtree` or compute-and-write for the single new row.
- [ ] Unit tests cover: head formula, sub formula, sibling-padding behavior, cascade on widening (parent + multi-level descendants), every rejection path (400 / 404 / 409 / 422), and the three `P2002.meta.target` branches.
- [ ] Code written, reviewed, merged to dev.

## Implementation notes

- **Layer ordering (cheap-first):** AuthGuard → role check → DTO validation (`class-validator`) → `CoaService` input rules → parent lookup (404) → parent compatibility → app-layer uniqueness pre-checks (friendly 409) → `DisplayCodeService` compute (422 on overflow) → DB commit (race-proof 409 via `P2002`).
- **Catch-don't-check for uniqueness.** Pre-checks for friendly errors are OK; the DB constraint is the only race-free guard.
- **All-zero seed rule is sub-only.** `seed: ""` is caught at the DTO with `@IsNotEmpty`. `seed: "00"` is allowed on a head but rejected on a sub — that check lives in `CoaService` because it depends on `isPostable`.
- **Type inheritance on nested create.** A child's `accountType` must equal its parent's. If the client sends a different `accountType` with a non-null `parentId`, that's a `400`.
- **Cascade scope.** "Parent's entire subtree" = the parent itself plus every descendant, computed level-by-level via BFS. Use the same `tx` for every read and update.
- **`createdBy`.** Pulled from the authenticated user on the request — not from the DTO. Confirm the guard pattern under `backend/src/auth/` before wiring.
- **`CompanyConfig`.** Schema model exists but is **not read** this sprint. `coaDigits = 10` is a module-level constant inside `DisplayCodeService`. Still seed one `CompanyConfig` row so the table isn't empty when later work wires it up.
- **The pure helper boundary.** Keep `computeDisplayCode(paddedChain, paddedSelf, isPostable, coaDigits)` as a non-exported function inside the `DisplayCodeService` file (or co-located and re-exported only through the service). No other module imports it directly — that's the seam.
