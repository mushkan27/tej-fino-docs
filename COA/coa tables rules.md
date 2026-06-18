# Chart of Accounts — Design

**Module:** Finance → Chart of Accounts

---

## Hierarchy

```
Account Type  →  group only, never postableAccount Head  →  group only, never postableSub-Account   →  leaf, always postable
```

Voucher module will only ever query `sub_accounts`.

---

## Numbering Rules (live in code, not DB)

```
subDigits      =  totalDigits > 4 ? 3 : 2headDigits     =  totalDigits - 1 - subDigitstypeIncrement  =  10 ^ (totalDigits - 1)headIncrement  =  10 ^ subDigitssubIncrement   =  1
```

**Auto-suggest:**

```
New type  →  MAX(type codes)            + typeIncrementNew head  →  MAX(heads under this type) + headIncrementNew sub   →  MAX(subs under this head)  + 1             or headCode + 1 if no subs yet
```

`totalDigits` is read from any existing code in DB at runtime.

---

## Schema

### `account_types`

Seeded at import. Never changes after.

| column     | type                | notes                        |
| ---------- | ------------------- | ---------------------------- |
| id         | UUID PK             |                              |
| name       | VARCHAR(100) UNIQUE | Enum[Assets, Liabilities...] |
| code       | VARCHAR(20) UNIQUE  | 10000, 20000... from Excel   |
| created_at | TIMESTAMPTZ         |                              |

---

### `account_heads`

|column|type|notes|
|---|---|---|
|id|UUID PK||
|account_type_id|UUID FK → account_types||
|code|VARCHAR(20) UNIQUE||
|title|VARCHAR(255)||
|description|TEXT|nullable|
|is_active|BOOLEAN|default true, never delete|
|created_by|UUID FK → users||
|created_at|TIMESTAMPTZ||

---

### `sub_accounts`

|column|type|notes|
|---|---|---|
|id|UUID PK||
|account_head_id|UUID FK → account_heads||
|code|VARCHAR(20) UNIQUE||
|title|VARCHAR(255)||
|description|TEXT|nullable|
|is_active|BOOLEAN|default true, never delete|
|created_by|UUID FK → users||
|created_at|TIMESTAMPTZ||

---

## Key Decisions

|Decision|Reason|
|---|---|
|`code` is VARCHAR not INT|label only, structure lives in FK|
|No `is_postable` field|table itself answers this|
|No `coa_config` table|rules are fixed in code|
|`is_active` not DELETE|transactions will reference codes later|
|3 tables not enum|type codes come from Excel, must be stored|

---

## Permissions

Enforced at API level, not frontend.

|Action|Roles|
|---|---|
|View|Finance Admin, Finance Manager, Accountant|
|Import, Create, Deactivate|Finance Admin, Finance Manager only|

---

## Import Rule

Validate every row → if one fails, save nothing → show row-by-row errors.