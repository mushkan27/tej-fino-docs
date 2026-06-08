# Issue: Department Configuration and API Endpoints

Date: 2026-06-08 | ID: ISSUE-5 (sub-task: dept-config) | Status: In Progress

## Summary

This sub-task adds a NestJS department module that serves Finance menu
data from a db.json file. It exposes two API endpoints that return
departments and role-filtered sub-headings, forming the backend
foundation for the Finance navigation UI.

## Root Cause

No backend exists to serve department/menu data dynamically. The Finance
navbar is currently hardcoded in the frontend. This sub-task replaces
that with a proper API so sub-headings are role-aware and data-driven.

## Decision

### Decision: Use db.json as temporary data source for Department API

Date: 2026-06-08
Context: Need to deliver department configuration and role-filtered API
endpoints within a small sprint. Prisma/Supabase setup is being
handled by another team member and is not yet available.

Options:
- A — Hardcode in service: Rejected — mixes data with logic, requires redeployment for any data change, creates technical debt
- B — db.json file: Chosen
- C — Prisma + Supabase: Rejected — blocked by team dependency, overkill for sprint size S

Chosen: B — db.json

Reasoning: Fastest to implement within sprint S. Keeps data separate
from logic. Only the service layer changes when Prisma lands.

Trade-off accepted: No validation or migrations — acceptable because
this is temporary config data, not production data.

Risks: If db.json structure changes before Prisma migration, dependent
sub-issues break. Mitigation: freeze the response shape as a contract.

## Key Learnings

- db.json separates data from logic — change a label without touching code
- Role filtering belongs in the API, not just the frontend (security, not just UX)
- The service is the only file that changes when Prisma replaces db.json
- API contract (response shape) must be frozen early — other sub-issues depend on it
- Mirror src/auth/ folder structure for src/department/

## Resources

- [NestJS Controllers](https://docs.nestjs.com/controllers) — route and query param setup
- [NestJS Modules](https://docs.nestjs.com/modules) — wiring module, controller, service
- [NestJS Query Params guide](https://geshan.com.np/blog/2024/07/nestjs-query-params/) — reading ?role= param
- [NestJS Project Structure 2026](https://encore.dev/articles/nestjs-project-structure-best-practices) — feature-folder pattern

## Acceptance Criteria

- [ ] db.json contains Finance department with all 7 sub-headings
- [ ] Each sub-heading has a roles array
- [ ] GET /departments returns all departments
- [ ] GET /departments/:id/menu?role=admin returns role-filtered sub-headings
- [ ] NestJS module wired: module, controller, service, spec files
- [ ] Response shape matches agreed contract
- [ ] Tests written for service and controller

## API Contract (frozen)

```
GET /departments
Response: { departments: [{ id, name }] }

GET /departments/:id/menu?role=
Response: {
  department: "Finance",
  subHeadings: [
    { id, label, path }
  ]
}
```

Note: roles are filtered server-side and NOT returned in the response.

## Implementation Notes

- Mirror src/auth/ folder structure — create src/department/
- Files to create:
  - department.module.ts
  - department.controller.ts
  - department.controller.spec.ts
  - department.service.ts
  - department.service.spec.ts
- db.json lives at project root or src/ level
- role param is temporary — when Sub-issue 1 (AuthContext) merges,
  role will come from JWT/session instead of query param
- Register DepartmentModule in app.module.ts
- Add all 7 sub-headings to db.json even if only CoA page exists now
  (others point to their path — frontend handles what is active)

## db.json Structure

```json
{
  "departments": [
    {
      "id": "finance",
      "name": "Finance",
      "subHeadings": [
        { "id": "coa", "label": "Chart of Account (CoA)", "path": "/finance/coa", "roles": ["admin", "accountant", "finance_manager"] },
        { "id": "invoice", "label": "Invoice", "path": "/finance/invoice", "roles": ["admin", "accountant", "finance_manager", "payment_approver"] },
        { "id": "voucher-entry", "label": "Data Entry (Voucher Entry)", "path": "/finance/voucher-entry", "roles": ["admin", "accountant"] },
        { "id": "payment", "label": "Payment", "path": "/finance/payment", "roles": ["admin", "payment_approver"] },
        { "id": "report", "label": "Report", "path": "/finance/report", "roles": ["admin", "accountant", "finance_manager"] },
        { "id": "budget", "label": "Budget", "path": "/finance/budget", "roles": ["admin", "finance_manager"] },
        { "id": "advance", "label": "Advance", "path": "/finance/advance", "roles": ["admin", "accountant", "finance_manager", "payment_approver"] }
      ]
    }
  ]
}
```
