# New CoA Rules — Unified Referential Table

> **TL;DR** — We collapse the 3 separate models (AccountType / AccountHead / SubAccount) into **one self-referential table**. We **store only the raw seed** the user types; the padded display code is **computed on read**. Company picks a global `digits` length (e.g. 10) and all codes re-render instantly.

---

## 1. Why the change

Today we have 3 models with duplicated logic and rigid digit lengths. Problems:
- Padding rules live in multiple places.
- Changing total digit length means migrating data.
- Hierarchy is only 3 levels deep — real CoAs need N levels.

New design fixes all three:
- **One table**, parent–child via `parent_uuid`.
- **N levels deep** (Assets → Bank → Bank1 → Bank1-sub → … as deep as needed).
- **Only seeds stored** → changing `digits` is a single config change, no data migration.

---

## 2. The Table

| column        | type          | notes                                                                       |
| ------------- | ------------- | --------------------------------------------------------------------------- |
| `uuid`        | PK            | row id                                                                      |
| `parent_uuid` | FK / nullable | `null` → this row is a root Account Type (Assets/Liabilities/Income/…)      |
| `seed`        | string        | the raw characters the user typed (`1`, `1E`, `D`, `a`, `00001`, …)         |
| `title`       | string        | display name (Bank, Bank1, Bank1-acc1)                                      |
| `definition`  | string        | optional description                                                        |
| `is_postable` | boolean       | `false` = head (intermediate), `true` = sub-account (leaf, journal-postable) |

**Global config (per company):** `digits` (e.g. `10`).

That's it. No `Account_Head` column, no `Sub_Account` column stored — those are **computed**.

---

## 3. The Two Display Formulas

Pick the formula based on `is_postable`.

Let `ancestorSeeds` = concatenation of seeds from root → … → parent (NOT including self).  
Let `selfSeed` = this row's seed.  
Let `D` = company's `digits`.

### (a) Head row — `is_postable = false`
```
display = ancestorSeeds + selfSeed,  then right-pad with '0' until length = D
```

### (b) Sub-account row — `is_postable = true`
```
prefix  = ancestorSeeds              // ancestors at the LEFT
suffix  = selfSeed                   // own seed at the RIGHTMOST
middle  = '0' repeated (D - len(prefix) - len(suffix)) times
display = prefix + middle + suffix
```

If `middle < 0` → **reject**. User must raise global `digits`.

---

## 4. Worked Example (digits = 10)

The same tree, two views:

### View 1 — what the user sees (padded, computed)

| Account_Type | Account_Head | Sub_Account  | Title       |
| ------------ | ------------ | ------------ | ----------- |
| Assets       | `1000000000` | `-`          | Assets      |
| Liabilities  | `2000000000` | `-`          | Liabilities |
| Income       | `3000000000` | `-`          | Income      |
| Assets       | `1100000000` | `-`          | Bank        |
| Assets       | `1110000000` | `-`          | Bank1       |
| Assets       | `1110000000` | `1110000001` | Bank1-acc1  |
| Assets       | `1110000000` | `1110000002` | Bank1-acc2  |
| Assets       | `11D0000000` | `-`          | Bank2       |
| Assets       | `11D0000000` | `11D000000a` | Bank2-acc1  |
| Assets       | `1100000000` | `1100000001` | Bank (sub)  |

### View 2 — what the DB actually stores (seeds only)

| uuid  | seed | parent_uuid | title       | is_postable |
| ----- | ---- | ----------- | ----------- | ----------- |
| abc1  | `1`  | `null`      | Assets      | false       |
| abc2  | `2`  | `null`      | Liabilities | false       |
| abc3  | `3`  | `null`      | Income      | false       |
| abc4  | `1`  | abc1        | Bank        | false       |
| abc5  | `1`  | abc4        | Bank1       | false       |
| abc6  | `1`  | abc5        | Bank1-acc1  | true        |
| abc7  | `2`  | abc5        | Bank1-acc2  | true        |
| abc8  | `D`  | abc4        | Bank2       | false       |
| abc9  | `a`  | abc8        | Bank2-acc1  | true        |
| abc10 | `1`  | abc4        | Bank (sub)  | true        |

**Notice:** the DB stores ~25 chars of seeds total. The user sees 10-digit padded codes. Bump `digits` to 12 and every code re-renders — no migration.

---

## 5. Walk-through of a few rows

**`abc5` Bank1** (head, parent Bank)
- ancestorSeeds = `1`(Assets) + `1`(Bank) = `11`
- selfSeed = `1`
- head formula → `11` + `1` = `111`, right-pad to 10 → `1110000000` ✓

**`abc6` Bank1-acc1** (sub-account, parent Bank1)
- ancestorSeeds = `1` + `1` + `1` = `111`
- selfSeed = `1`
- sub formula → prefix `111`, suffix `1`, middle = `0` × (10 − 3 − 1) = `000000`
- display = `111` + `000000` + `1` = `1110000001` ✓

**`abc9` Bank2-acc1** (sub-account, parent Bank2 whose seed is `D`)
- ancestorSeeds = `1` + `1` + `D` = `11D`
- selfSeed = `a`
- display = `11D` + `000000` + `a` = `11D000000a` ✓

**`abc10` Bank (sub)** — postable sub-account directly under Bank, alongside sub-heads Bank1 & Bank2
- ancestorSeeds = `1` + `1` = `11`
- selfSeed = `1`
- display = `11` + `0000000` + `1` = `1100000001` ✓

---

## 6. Bank can have BOTH sub-heads AND sub-accounts

This is the case that confuses people. Bank (`abc4`) has:
- Sub-heads: Bank1 (`abc5`), Bank2 (`abc8`)
- Sub-accounts directly under it: `abc10` (display `1100000001`), and more if added

Bank-the-head occupies `1100000000`. A postable sub-account directly under Bank uses any other suffix: `1100000001`, `1100000002`, `110000000X`, etc.

If you want Bank itself to be postable, **create a separate sub-account row** under Bank — don't try to flip the head's flag. (Head row reserves the `…0000` slot; the sub-account row gets its own slot.)

---

## 7. Rules & Constraints

1. **Sibling-unique seeds.** Two rows under the same parent cannot share a seed.
2. **Globally unique displayed code.** After computing the display string, it must be unique across the whole table (both head and sub-account displays). This catches the rare case where a sub-head's right-pad collides with a sub-account's left-prefix+right-suffix.
3. **Seed `0` for sub-accounts is banned.** A sub-account with seed `0` under parent X would render identical to X's head row (all zeros). Reject at input.
4. **Overflow → reject.** If `len(ancestorSeeds) + len(selfSeed) > digits`, reject. The fix is to raise global `digits`, which re-renders all codes.
5. **Posting (journal entries) only on `is_postable = true` rows.** Heads are organizational only.
6. **Roots = Account Types.** `parent_uuid = null` rows are the top-level types (Assets, Liabilities, Income, Expense, Equity, …). They are heads (`is_postable = false`).

---

## 8. Computing display code (pseudocode)

```ts
function displayCode(row, digits) {
  const ancestors = []
  let cur = row.parent_uuid
  while (cur) {
    const p = findById(cur)
    ancestors.unshift(p.seed)   // root ends up at index 0
    cur = p.parent_uuid
  }
  const ancestorSeeds = ancestors.join('')
  const self = row.seed

  if (!row.is_postable) {
    // head: right-pad
    const code = ancestorSeeds + self
    if (code.length > digits) throw new Error('Increase digits')
    return code.padEnd(digits, '0')
  } else {
    // sub-account: ancestors left, self rightmost, zeros in middle
    const middleLen = digits - ancestorSeeds.length - self.length
    if (middleLen < 0) throw new Error('Increase digits')
    return ancestorSeeds + '0'.repeat(middleLen) + self
  }
}
```

---

## 9. Open questions (track with Maam)

- [ ] Confirm `is_postable` as an explicit flag (vs. derived "has no children"). Recommended: **explicit flag** — matches the Bank case where both a head row and a sub-account row coexist for the same conceptual node.
- [ ] Confirm seed `0` is banned for sub-accounts.
- [ ] Confirm multi-character seeds (e.g. `1E`, `00001`) are allowed for both heads and sub-accounts.
- [ ] Maximum depth — unbounded, or capped (say 8)?

---

## 10. Migration note (from current 3 models)

- Account Types → root rows (`parent_uuid = null`, `is_postable = false`).
- Account Heads → child rows of types (`is_postable = false`).
- Sub Accounts → leaf rows (`is_postable = true`).
- Old padded `code` columns → throw away; recompute from `seed` + `digits`.
- The previously-stored `Account_Head` / `Sub_Account` numbers stop being source-of-truth — they're now derived.
