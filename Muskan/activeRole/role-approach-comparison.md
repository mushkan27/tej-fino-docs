# Role approach comparison: `activeRole` column vs array of objects

---

## Schema shape

### `activeRole` column ✅ Recommended

```
roles:      Role[]          // simple enum array
            [FINANCE_ADMIN, EMPLOYEE]

activeRole: Role?           // separate nullable field
            FINANCE_ADMIN
```

The active role lives **beside** the array as its own column.

---

### Array of objects

```
roles: UserRole[]           // richer objects, no extra field
       { role: FINANCE_ADMIN, isDefault: true  }
       { role: EMPLOYEE,      isDefault: false }
```

The active role lives **inside** the array via `isDefault: true`.

---

## Login flow

| Step | `activeRole` column | Array of objects |
|------|---------------------|------------------|
| 1 | User logs in via Google OAuth | User logs in via Google OAuth |
| 2 | `findUnique` on user → get `roles[]` and `activeRole` | `findUnique` on user → get `roles[]` (array of objects) |
| 3 | If `activeRole` is null → set to `roles[0]`, save to DB | Filter roles for `isDefault: true` → that is the active role |
| 4 | Sign JWT with `{ sub, email, roles, activeRole }` | Sign JWT with `{ sub, email, roles, activeRole }` |
| 5 | CASL builds permissions for `activeRole` | CASL builds permissions for `activeRole` |

---

## Role switch flow

| Step | `activeRole` column | Array of objects |
|------|---------------------|------------------|
| 1 | `PATCH /auth/switch-role` with new role | `PATCH /auth/switch-role` with new role |
| 2 | Validate role is in user's `roles[]` | Validate role exists in user's `roles[]` |
| 3 | **1 DB call** — update `activeRole` column directly | **DB call 1** — set current default's `isDefault → false` |
| 4 | Re-sign JWT, return new token | **DB call 2** — set new role's `isDefault → true` |
| 5 | — | Re-sign JWT, return new token |

```ts
// activeRole column — one DB call
await prisma.user.update({
  where: { id: userId },
  data: { activeRole: newRole },
});

// Array of objects — two DB calls
await prisma.user.update({
  where: { id: userId },
  data: {
    roles: {
      updateMany: [
        { where: { isDefault: true },  data: { isDefault: false } },
        { where: { role: newRole },     data: { isDefault: true  } },
      ]
    }
  }
});
```

---

## At a glance

| Aspect | `activeRole` column | Array of objects |
|--------|---------------------|------------------|
| Migration | ✅ Minimal — 1 new column | ⚠️ Medium — restructure roles field |
| Schema change | Add nullable field to `users` | `roles` becomes array of objects |
| DB calls to switch | ✅ 1 | ⚠️ 2 |
| Audit trail | ❌ None | ❌ None (no `assignedBy`/`assignedAt`) |
| Complexity | ✅ Low | ⚠️ Medium |
| JWT & CASL output | ✅ Identical | ✅ Identical |

> Both approaches produce the exact same JWT and CASL output.
> The difference is purely in schema and operational cost.

---

## Verdict

**Go with the `activeRole` column.**

For a finance app at this stage, you don't need audit history — there's no admin UI for managing other users' roles yet. The array-of-objects approach is a cleaner concept but costs more to migrate, takes 2 DB calls on every role switch, and delivers zero extra value until you actually build role assignment management.

Add it later if compliance or admin features demand it.
