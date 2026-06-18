# Issue: Chart of Accounts — frontend component (self-contained, beginner v1)

Date: 2026-06-13 | ID: FINO-66 | Status: In progress

## Summary

Build a self-contained `ChartOfAccounts` React component (Next.js App Router) that another teammate will mount inside the existing HomeLayout/nav. Backend is already done. The component fetches `GET /api/accounts` on mount, shows an Import CSV button when no accounts exist, and switches to a table + `+ Create CoA` button once at least one account exists.

## Root cause

Missing feature — pure frontend build. No bug, no migration, no API work. Backend endpoints (`GET /api/accounts`, `POST /api/accounts`, `POST /coa/import`) and rules already live on this branch.

## Decision

## Decision: ChartOfAccounts component — scope, structure, and dependencies for v1
Date: 2026-06-13 | Context: building the right-side CoA component as a standalone, self-contained piece her teammate will mount inside the existing HomeLayout/nav. Backend is done; this is pure frontend.

Options:
  A — One big component file, local form state inside AddRow, drag-and-drop import, live code preview, no search.
      Rejected: large file is unmaintainable; AddRow-local state breaks "Cancel keeps values"; drag-drop and live preview add risk without being in scope for v1.
  B — Per-role split (ChartOfAccounts + CoATable + AddRow + ConfirmModal + ImportButton), form state lifted to parent, click-only import, no live preview, search+filter included.
      CHOSEN.
  C — Atomic split + global state via context.
      Rejected: premature for a one-off feature.

Chosen: B.

Reasoning: Per-role split keeps files small. Lifting form state to parent is the only way Cancel-keeps-values works. Click-only import ships fast. Live preview is the highest-risk piece — Confirm modal already shows server-truth code. Search/filter are trivial client-side. Tailwind v4 (already installed) + @radix-ui/react-dialog (accessibility for free).

Trade-off accepted: deviation from existing CSS-Modules convention (will flag to team lead); no live preview in v1; +1 dep (radix-dialog, ~5KB gz).

Risks & mitigations:
- Stale codes after backend cascade → always refetch list, never append.
- Loading flicker → isLoading flag + skeleton before reading accounts.length.
- Double-submit on slow net → disable Confirm during in-flight POST.
- Refetch race → `let ignore = false` cleanup pattern.
- Form state lost on server error → clear only on success.
- Two inline rows → isAdding boolean guard.

## Key learnings

- Lifting state up isn't a style preference — it's load-bearing for "Cancel must keep values" because unmounting a child component wipes its local useState.
- "Refetch, don't append" is forced by the backend's widening cascade: adding a sibling can renumber other rows' displayCodes.
- displayCode is a string (potentially alphanumeric), not a number — never do math on it, never strip leading zeros.
- For one accessible modal, Radix UI Dialog ships ~25 lines vs ~80–120 from scratch (Esc, focus trap, scroll lock, ARIA all free).
- Plain HTML `<table>` + Tailwind beats TanStack Table for small custom-rendered tables — libraries pay off only when you need sort/virtualize/resize.

## Resources

- [React: sharing state between components](https://react.dev/learn/sharing-state-between-components) — why form state lives in ChartOfAccounts, not AddRow.
- [React: useEffect data fetching](https://react.dev/reference/react/useEffect#fetching-data-with-effects) — race-condition guard pattern.
- [MDN FormData](https://developer.mozilla.org/en-US/docs/Web/API/FormData) — multipart upload for CSV (don't set Content-Type manually).
- [Radix UI Dialog](https://www.radix-ui.com/primitives/docs/components/dialog) — modal primitive used for ConfirmModal.
- [Tailwind v4 styling](https://tailwindcss.com/docs/styling-with-utility-classes) — utility-first styling on the new v4 install.
- Internal: backend/src/accounts/display-code.service.ts — formula source of truth.
- Internal: backend/src/accounts/accounts.service.ts — POST flow including `recomputeSubtree`.

## Acceptance criteria

- [ ] Component lives at `frontend/src/components/ChartOfAccounts/` and is rendered by an external route (teammate's nav) without modification.
- [ ] On mount, `GET /api/accounts` is called once with the `let ignore = false` race guard.
- [ ] Loading state shows a skeleton table; no Import-button flash before fetch resolves.
- [ ] Empty state (`accounts.length === 0`) shows only the Import CSV button.
- [ ] Click Import → file picker → CSV uploads via `POST /coa/import` (multipart, no manual Content-Type).
- [ ] On import success, list refetches and component flips to populated state automatically.
- [ ] Populated state shows table + `+ Create CoA` button; Import button hidden.
- [ ] Table columns: Account Type, Account Head, Sub Account, Title, Definition; head bold, sub indented, type badge only on first row of each group.
- [ ] Search input filters loaded rows by title/code/type/definition; type-filter dropdown narrows by AccountType.
- [ ] `+ Create CoA` opens at most one inline yellow row; clicking again while open is a no-op.
- [ ] Form state lives in `ChartOfAccounts`, not `AddRow` — Cancel from the Confirm modal closes the modal but keeps the inline row populated.
- [ ] AddRow correctly switches between ROOT / HEAD / SUB modes (no parents exist for type → ROOT; parent picked + radio chooses HEAD/SUB).
- [ ] Save validates client-side, then opens Confirm modal showing all fields including server-shape preview.
- [ ] Confirm Cancel → modal closes, row stays open with values intact.
- [ ] Confirm Confirm → `POST /api/accounts`, Confirm button disabled while in flight, then refetch full list on success.
- [ ] Non-2xx response shows server's error message in dismissible red banner under top bar; inline row + form state preserved.
- [ ] No business rules duplicated client-side (no v1 preview helper).
- [ ] Tailwind v4 used for styling; deviation from CSS-Modules convention flagged to team lead in PR description.

## Implementation notes

- Files: `ChartOfAccounts.tsx` (parent, owns state + fetch), `CoATable.tsx`, `AddRow.tsx` (controlled), `ConfirmModal.tsx` (Radix Dialog), `ImportButton.tsx`, `types.ts` (mirrors of backend `Account` and `CreateAccountDto`).
- API helpers: extend `frontend/src/api/` with `accounts.ts` exposing `listAccounts()`, `createAccount(dto)`, `importCoaCsv(file)`.
- Import: use `FormData`, append the file under the field name `file` (matches backend's `FileInterceptor('file')`). Do NOT set `Content-Type` — axios + browser set the boundary.
- Auth: existing `@/api/axios` instance auto-attaches the Bearer token from localStorage. Don't re-implement.
- Modal: `@radix-ui/react-dialog`. Install with `npm install @radix-ui/react-dialog` inside `frontend/`.
- AccountType enum values to match backend exactly: `ASSETS | LIABILITIES | EQUITY_NET_ASSETS | INCOME | EXPENSES`.
- Display labels for AccountType (UI only): Assets, Liabilities, Equity / Net Assets, Income, Expenses.
- displayCode is a string — never parse as number.
- After POST success: `setForm(emptyForm); setIsAdding(false); setConfirmOpen(false); await refetch();` in that order.
- On POST error: `setConfirmOpen(false); setError(serverMessage);` — keep `form` and `isAdding` untouched.
- Group-badge rendering: in `CoATable`, track previous row's type while mapping; render the badge only when current type !== previous type.
