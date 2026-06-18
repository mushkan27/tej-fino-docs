# Voucher Entry — End-to-End Flow

|#|User flow|Frontend action|Backend endpoint|Description|
|---|---|---|---|---|
|1|User clicks **Finance → Voucher Entry** in sidebar|Navigate to `/finance/voucher-entry`|—|Page loads. No backend call yet.|
|2|Page mounts, "Generate Voucher" card is active by default|Render empty form (voucher type fixed to `PAYMENT`)|—|Frontend shows the hardcoded Payment Voucher form scaffold.|
|3|"Voucher #" field needs a draft number|Render placeholder (`—` or "Will be generated on save")|—|No backend call. Number is allocated only when the draft is saved.|
|4|User clicks **Bank A/C** dropdown|Open cascading picker, request roots|`GET /api/accounts/children`|Returns `{ nonPostable, postable }` for top-level accounts.|
|5|User picks a non-postable root, picker shows next level|Request children of that node|`GET /api/accounts/children?parentId=<id>`|One call per drill level. Lazy.|
|6|User reaches the postable leaf and selects it|Store `bankAccountId` in form state, close picker|—|Selection only — no backend write yet.|
|7|User fills **Payee Name**, **Payment Reference**, **Voucher Date**|Local form state|—|Free-text and date input.|
|8|User clicks first **Account Head** dropdown on line 1|Same cascading picker opens|`GET /api/accounts/children`|Same endpoint reused. Returns roots.|
|9|User drills until they pick a non-postable parent|Show **Transaction Type** dropdown with that node's postable children|(already returned in step 8's response)|The same single response feeds both dropdowns at each level.|
|10|User picks the postable leaf in Transaction Type|Store `accountHeadId` + `postableAccountId` on the line|—|Frontend keeps both ids — head for grouping, postable for posting.|
|11|User types **Description**, **Debit** or **Credit** for the line|Local form state; recompute running totals|—|Live totals at bottom: Dr / Cr / Difference.|
|12|User clicks **Add line item**|Append empty row to lines array|—|Min 2 non-empty lines for save.|
|13|User drops PDF/JPG/etc. into **Supporting Documents** zone|Hold files in memory until first save|—|Cannot upload yet — voucher has no id.|
|14|User clicks **Save Draft** (first time)|POST form + lines|`POST /api/vouchers`|Server generates `voucherNumber`, creates `Voucher` (status=`draft`) + `VoucherLine` rows in one transaction. Returns the saved voucher.|
|15|After save, frontend has the voucher id|Update UI with returned `voucherNumber`, switch to "Edit Draft" mode|—|Voucher # field now shows `PV-2026-00042`.|
|16|Frontend uploads queued attachments|Multipart POST per file batch|`POST /api/vouchers/:id/attachments`|Server validates mime + size, stores BLOBs, returns metadata array.|
|17|User clicks an attachment row to preview|GET the blob|`GET /api/vouchers/:id/attachments/:attId`|Server streams the file with correct `Content-Type`.|
|18|User clicks **x** on an attachment|DELETE|`DELETE /api/vouchers/:id/attachments/:attId`|Allowed only while voucher is `draft`.|
|19|User edits a line / adds another (still in draft)|Re-save|`POST /api/vouchers` _or_ a future PUT|**Note:** for current scope, simplest is to disallow edit after first save and treat "Save Draft" as one-shot. If edit-draft is in scope, add `PATCH /api/vouchers/:id` (deferred).|
|20|Difference banner shows ₹0 — voucher is balanced|**Send for Payment** button becomes enabled|—|Client-side gate; server re-validates.|
|21|User clicks **Send for Payment**|POST transition|`POST /api/vouchers/:id/send-for-payment`|Server re-runs all validations + balance check + required-field check for PAYMENT. Flips status `draft → review`. Returns updated voucher.|
|22|Success — toast "Sent for payment"|Mark voucher read-only, show `REVIEW` status pill|—|No further edits allowed by UI.|
|23|User clicks **Discard** before saving|Clear form state, navigate away|—|No backend call — draft was never created.|

## Status pill mapping (display only — pure frontend)

|Backend value|UI label|
|---|---|
|`draft`|DRAFT|
|`review`|REVIEW|
|`approved`|APPROVED|
|`paid`|