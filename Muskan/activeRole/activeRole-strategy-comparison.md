# Strategy: JWT-only vs DB+JWT Hybrid

## 1. What the Approaches Mean

|                        | JWT Only                    | DB + JWT Hybrid                                             |
| ---------------------- | --------------------------- | ----------------------------------------------------------- |
| Where activeRole lives | Inside the JWT payload only | Persisted in the DB (`users.activeRole`) AND carried in JWT |
| Schema change needed   | No                          | Yes — Prisma migration required                             |
| Source of truth        | The token the client holds  | The database                                                |

---

## 2. General Pros & Cons

| Dimension                       | JWT Only                                       | DB + JWT Hybrid                                      |
| ------------------------------- | ---------------------------------------------- | ---------------------------------------------------- |
| Persistence across logout/login | ✗ Resets to `roles[0]` on next login           | ✓ Remembered from DB                                 |
| Cross-device sync (eventual)    | ✗ No                                           | ✓ Yes, on token refresh                              |
| Cross-device sync (immediate)   | ✗ No                                           | ✗ No (JWT limitation — needs blocklist or WebSocket) |
| DB write on role switch         | ✗ None                                         | ✓ One write per switch                               |
| Query active role server-side   | ✗ Not possible                                 | ✓ `SELECT * FROM users WHERE activeRole = 'admin'`   |
| Audit trail support             | ✗ Must manually extract from token at log time | ✓ Always queryable from DB                           |
| Implementation complexity       | Low                                            | Medium                                               |
| Rollback ease                   | Easy                                           | Requires migration rollback                          |

---

## 3. Role Switch Flow

### JWT Only

```
User clicks "Switch to Admin"
  → POST /auth/switch-role { role: "admin" }
  → Server validates "admin" is in user's roles[]
  → Server issues NEW JWT { activeRole: "admin" }
  → Frontend replaces old token
  → User operates as admin (current session only)
```

### DB + JWT Hybrid

```
User clicks "Switch to Admin"
  → POST /auth/switch-role { role: "admin" }
  → Server validates "admin" is in user's roles[]
  → Server writes activeRole: "admin" to DB       ← extra step
  → Server issues NEW JWT { activeRole: "admin" }
  → Frontend replaces old token
  → User operates as admin (persisted)
```

---

## 4. Scenario Comparison

### After Logout & Login

|                 | JWT Only                                    | DB + JWT Hybrid                                    |
| --------------- | ------------------------------------------- | -------------------------------------------------- |
| What happens    | `activeRole` resets to `roles[0]` (default) | `activeRole` read from DB, user resumes as `admin` |
| User experience | Must re-switch role manually                | Seamless, remembered                               |

### On a Second Device (mid-session)

|                      | JWT Only                                 | DB + JWT Hybrid                          |
| -------------------- | ---------------------------------------- | ---------------------------------------- |
| Immediate reflection | ✗ Old token still active on other device | ✗ Old token still active on other device |
| After token refresh  | ✗ Still derives from default             | ✓ Picks up `activeRole` from DB          |
| Fix for instant sync | Token blocklist + re-issue               | Token blocklist or WebSocket push        |

> **Note:** Mid-session cross-device sync is a fundamental JWT limitation in **both** approaches.
> DB+JWT only closes the gap at token refresh time, not instantly.

---

## 5. Audit Trail & Admin Visibility

| Need                                           | JWT Only                                           | DB + JWT Hybrid                                    |
| ---------------------------------------------- | -------------------------------------------------- | -------------------------------------------------- |
| Log which role performed an action             | Must manually extract from token at each log point | Always available via DB query                      |
| Admin dashboard: "who is acting as admin now?" | ✗ Not queryable server-side                        | ✓ `SELECT * FROM users WHERE activeRole = 'admin'` |
| Reconstruct role state during an incident      | ✗ Difficult                                        | ✓ Straightforward                                  |
| Compliance / regulated industries              | ✗ Insufficient alone                               | ✓ Supports audit requirements                      |

---

## 6. When to Choose Which

| Use Case                                                    | Recommended Approach |
| ----------------------------------------------------------- | -------------------- |
| Role is purely session/ephemeral (resets on logout is fine) | JWT Only             |
| Simple internal tool, no compliance needs                   | JWT Only             |
| Short-lived tokens, re-login is acceptable                  | JWT Only             |
| Role choice must survive logout/login                       | DB + JWT Hybrid      |
| Multi-device consistency matters                            | DB + JWT Hybrid      |
| Need server-side queryability of active roles               | DB + JWT Hybrid      |
| Regulated industry (finance, healthcare, legal)             | DB + JWT Hybrid      |
| Super-admin needs to monitor/override user role states      | DB + JWT Hybrid      |

---
