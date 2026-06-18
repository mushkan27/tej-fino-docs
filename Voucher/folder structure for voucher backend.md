# Voucher Backend — Shared File Structure

> Schema is already merged. Both PRs work in parallel from this layout.

---

## Folder Layout

```
backend/src/vouchers/
├── vouchers.module.ts
├── vouchers.controller.ts
├── vouchers.service.ts
├── vouchers.helper.ts
├── dto/
│   └── create-voucher.dto.ts
├── vouchers.service.spec.ts
└── vouchers.helper.spec.ts
```

### Files touched outside the module

```
backend/src/accounts/accounts.controller.ts   → PR 1 adds /children route
backend/src/accounts/accounts.service.ts      → PR 1 adds getChildren()
backend/src/app.module.ts                     → register VouchersModule
```

---

## Ownership Matrix

| File | PR 1 owns | PR 2 owns |
| ---- | --------- | --------- |
| `vouchers.module.ts` | creates the module | adds `MulterModule` import |
| `vouchers.controller.ts` | `createDraft`, `getOne` | `sendForPayment`, attachment routes |
| `vouchers.service.ts` | `createDraft`, `getOne` | `sendForPayment`, attachment methods |
| `vouchers.helper.ts` | `generateVoucherNumber`, `prefixFor`, validators | reuses the validators |
| `dto/create-voucher.dto.ts` | full | — |
| `vouchers.service.spec.ts` | create + get tests | send-for-payment + attachment tests |
| `vouchers.helper.spec.ts` | generator + validator tests | — |
| `accounts.controller.ts` | adds `/children` | — |
| `accounts.service.ts` | adds `getChildren()` | — |
| `app.module.ts` | registers `VouchersModule` | — |

---

## Endpoint Ownership

### PR 1
- `GET  /api/accounts/children`
- `POST /api/vouchers`
- `GET  /api/vouchers/:id`

### PR 2
- `POST   /api/vouchers/:id/send-for-payment`
- `POST   /api/vouchers/:id/attachments`
- `GET    /api/vouchers/:id/attachments/:attId`
- `DELETE /api/vouchers/:id/attachments/:attId`

---

## Notes

- **One DTO file is enough.** Attachment uploads use `multipart/form-data` — multer provides `Express.Multer.File[]` directly, no DTO needed.
- **One spec file per source file is enough** at this size. Split later if it grows.
- **Stub all methods on day one** using the agreed controller / service skeleton. Each person fills in only their own methods. Merge becomes a mechanical join.
- **Both PRs depend only on the schema migration**, which is already merged. No code dependency between PR 1 and PR 2.
- **Shared validators** (`assertPostable`, `assertNonPostable`, `assertDescendant`) live in `vouchers.helper.ts`. Whichever PR implements them first, the other imports.

---

## Merge Strategy

1. Both branches grow in parallel from `main`.
2. Whichever PR finishes first → merge to `main`.
3. The second PR rebases on `main`, resolves mechanical conflicts (~30 min) in:
   - `vouchers.module.ts`
   - `vouchers.controller.ts`
   - `vouchers.service.ts`
4. Run tests, ship.