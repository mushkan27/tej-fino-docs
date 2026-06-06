# Issue 60 — CoA Prisma Models Implementation Notes

**Date:** 2026-06-06 | **Status:** In progress | **Branch:** `fino-60-coa-data-models`

---

## Summary

Add three Prisma models (`AccountType`, `AccountHead`, `SubAccount`) to `backend/prisma/schema.prisma` and run a migration. These models are the database foundation for the Chart of Accounts module. No API endpoints in this issue — data layer only.

---

## File to edit

`backend/prisma/schema.prisma`

Command to run after saving:

```bash
npx prisma migrate dev --name add_coa_models
```

---

## What to add to schema.prisma

```prisma
enum AccountTypeName {
  ASSETS
  LIABILITIES
  EQUITY_NET_ASSETS
  INCOME
  EXPENSES
}

model AccountType {
  id           String          @id @default(cuid())
  name         AccountTypeName @unique
  code         String          @unique
  createdBy    String
  createdByUser User           @relation("AccountTypeCreatedBy", fields: [createdBy], references: [id], onDelete: Restrict)
  createdAt    DateTime        @default(now())
  updatedAt    DateTime        @updatedAt
  accountHeads AccountHead[]

  @@map("account_types")
}

model AccountHead {
  id              String      @id @default(cuid())
  accountTypeId   String
  accountType     AccountType @relation(fields: [accountTypeId], references: [id], onDelete: Restrict)
  code            String      @unique
  title           String
  description     String?
  isActive        Boolean     @default(true)
  createdBy       String
  createdByUser   User        @relation("AccountHeadCreatedBy", fields: [createdBy], references: [id], onDelete: Restrict)
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt
  subAccounts     SubAccount[]

  @@map("account_heads")
}

model SubAccount {
  id            String      @id @default(cuid())
  accountHeadId String
  accountHead   AccountHead @relation(fields: [accountHeadId], references: [id], onDelete: Restrict)
  code          String      @unique
  title         String
  description   String?
  isActive      Boolean     @default(true)
  createdBy     String
  createdByUser User        @relation("SubAccountCreatedBy", fields: [createdBy], references: [id], onDelete: Restrict)
  createdAt     DateTime    @default(now())
  updatedAt     DateTime    @updatedAt

  @@map("sub_accounts")
}
```

Also add back-relations to the `User` model:

```prisma
model User {
  // ... existing fields ...
  accountTypesCreated  AccountType[]  @relation("AccountTypeCreatedBy")
  accountHeadsCreated  AccountHead[]  @relation("AccountHeadCreatedBy")
  subAccountsCreated   SubAccount[]   @relation("SubAccountCreatedBy")
}
```

---

## Key decisions made

| Decision | What | Why |
|---|---|---|
| Prisma enum | `AccountTypeName` in schema | DB-level enforcement — 5 types are a global accounting standard, never change |
| Hard FK for `createdBy` | `@relation` to `User` | Financial records must always trace to a real user |
| `onDelete: Restrict` | On all FK relations | Parent rows cannot be deleted if children exist — correct for financial data |
| `onUpdate: Cascade` | Default, not written explicitly | CUIDs never change after creation, so this never fires |
| `isActive` not DELETE | Soft delete on `AccountHead` and `SubAccount` | Past transactions reference codes — hard delete would break them |
| `@@map` snake_case | All three models | Consistent with existing `users` table convention |
| `updatedAt` included | All three models | Consistent with existing `User` model |

---

## Hierarchy reminder

```
AccountType   →  grouping only, never postable
  AccountHead →  grouping only, never postable
    SubAccount →  leaf, always postable (vouchers reference this only)
```

---

## Acceptance criteria

- [ ] Three models added to `schema.prisma`
- [ ] `AccountTypeName` enum defined in schema
- [ ] Migration runs clean with no errors (`prisma migrate dev`)
- [ ] Prisma client regenerates successfully
- [ ] `User` model has back-relations for all three `createdBy` fields

---

## Gotchas

- `name` on `AccountType` is `@unique` — only one row per type (Assets, Liabilities, etc.)
- Each `@relation` on `createdBy` needs a unique name string (e.g. `"AccountTypeCreatedBy"`) because `User` will have multiple relations to these models — Prisma requires named relations in this case
- Run `npx prisma generate` after migrate to make sure the client is up to date before writing service code
