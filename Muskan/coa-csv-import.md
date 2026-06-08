# Issue: CSV Import Feature for Chart of Accounts (COA)

Date: 2026-06-08 | Branch: csv-import-api | Status: In Progress

## Summary

A new `POST /coa/import` endpoint is needed on the backend to accept a CSV file
upload representing Chart of Accounts data. The endpoint validates file type,
checks for five required column headers (case-insensitively), strips extra
columns, sanitises values, and logs the result to the console to confirm receipt.
No database writes in this phase — console.log is the acceptance signal.

## Root Cause

Missing feature. CASL permissions for the COA subject already exist in
`casl-ability.factory.ts` (FINANCE_ADMIN and FINANCE_MANAGER can `create` COA),
but there is no COA module, no file upload infrastructure, and no CSV parsing
library in the project. The backend returns 404 for any COA route today.

## Decision

### Decision: CSV Import endpoint implementation approach
Date: 2026-06-08
Context: New POST /coa/import endpoint needed with file type validation, header
validation, and data sanitisation before console.log confirmation.

Options:
- A — multer fileFilter + csv-parser + mapHeaders (chosen)
- B — ParseFilePipe + FileTypeValidator + manual string split,
      rejected: brittle against real-world CSVs (BOM, quoted headers, CRLF)
- C — multer fileFilter + fast-csv,
      rejected: heavier library, overkill for PoC scope

Chosen: A

Reasoning: csv-parser is minimal, handles encoding edge cases, and the pattern
(fileFilter → buffer → mapHeaders → validate → sanitise) maps cleanly to the
spec requirements without any fragile string manipulation.

Trade-off accepted: adds one npm dependency (csv-parser). Acceptable because it
is a single-purpose, stable, widely-adopted library with no transitive deps.

Risks: Browser MIME type for .csv can be 'application/vnd.ms-excel' on Windows —
mitigated by checking both MIME type AND file extension in fileFilter.

## Pipeline (implementation order)

```
Upload → multer fileFilter (type check: MIME + extension)
       → Buffer in memory (no disk I/O)
       → csv-parser with mapHeaders (toLowerCase + trim)
       → validate all 5 required headers exist → 400 if missing
       → parse rows, drop extra columns
       → sanitise values (trim whitespace)
       → console.log parsed data
```

## Key Learnings

- fileFilter, header normalisation, header validation, and value sanitisation
  are four distinct steps — conflating them causes bugs
- csv-parser's `mapHeaders` option handles normalisation at parse time; no
  manual looping over keys needed
- Browser MIME type for .csv files can be `application/vnd.ms-excel` on Windows,
  so checking MIME type alone is not sufficient — always check file extension too
- `Readable.from(buffer)` converts the multer Buffer into a Node.js stream that
  csv-parser can consume without writing anything to disk

## Resources

- [NestJS File Upload Docs](https://docs.nestjs.com/techniques/file-upload) —
  FileInterceptor, ParseFilePipe, FileTypeValidator reference
- [csv-parser npm](https://www.npmjs.com/package/csv-parser) —
  mapHeaders option, stream API
- [In-memory CSV parsing without disk storage](https://dev.to/damir_maham/streamline-file-uploads-in-nestjs-efficient-in-memory-parsing-for-csv-xlsx-without-disk-storage-145g) —
  pattern for Buffer → Readable → csv-parser

## Acceptance Criteria

- [ ] POST /coa/import with valid CSV → 201, data logged to console
- [ ] Non-CSV file → 400 `{ message: "Only CSV files are allowed" }`
- [ ] CSV missing any required header → 400 `{ message: "Missing required columns: ..." }`
- [ ] CSV with extra columns → extra columns silently dropped
- [ ] Values trimmed before logging
- [ ] Endpoint requires valid JWT → 401 without it

## Implementation Notes

- Required headers (case-insensitive): `account type`, `account head`,
  `sub account`, `title`, `definition`
- Install: `npm install csv-parser` and `npm install --save-dev @types/multer`
- New files: `src/coa/coa.module.ts`, `coa.controller.ts`, `coa.service.ts`
- Register COAModule in `src/app.module.ts`
- Use `JwtGuard` from existing auth module to protect the route
- Multer memory storage (no `dest` option) keeps files in Buffer — no cleanup needed
