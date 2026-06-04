# CASL Integration Plan

## What Are We Doing and Why?

Right now the backend only checks **who you are** (authentication — via Google OAuth + JWT).
It does NOT check **what you are allowed to do** (authorization).

CASL adds the authorization layer. Example:

- An HR user can view employees but cannot touch finance data.
- A Finance Admin can create and update invoices but cannot manage users.
- An Employee can only view their own profile.

---

## New Role Being Added

We are adding a **`FINANCE_ADMIN`** role (also called Finance Manager).

| Role            | What They Do                           |
| --------------- | -------------------------------------- |
| `ADMIN`         | Full access to everything              |
| `HR`            | Manage employees and HR records        |
| `FINANCE`       | View finance records                   |
| `FINANCE_ADMIN` | View + create + update finance records |
| `EMPLOYEE`      | View their own profile only            |

---

## Step-by-Step Plan

---

### Step 1 — Add `FINANCE_ADMIN` to the Database

**File:** `prisma/schema.prisma`

**What to do:**
Add `FINANCE_ADMIN` to the `Role` enum, then run a Prisma migration.

```prisma
enum Role {
  ADMIN
  HR
  FINANCE
  FINANCE_ADMIN   // ← add this
  EMPLOYEE
}
```

**Why first?**
Every other step depends on roles existing in the database. The user table already supports multiple roles (`roles Role[]`), so no structural changes are needed — just adding the new value.

**Commands to run after editing schema:**

```bash
npx prisma migrate dev --name add-finance-admin-role
npx prisma generate
```

**Why new Prisma Migration is required?**
Role enum is not just TypeScript type - it actually lives inside PostgreSQL database as real database-level type. So when we define role enum, Prisma translated that into an actual SQL command that told Postgres: "Create a type called Role that only allows these 4 values." The database enforces this — if you try to assign a value that isn't in the list, Postgres will flat out reject it.

---

### Step 2 — Install `@casl/ability`

**What to do:**

```bash
npm install @casl/ability
```

**Why?**
`@casl/ability` is the library that lets you write human-readable rules like:

- "ADMIN can manage everything"
- "HR can read and update Users"
- "FINANCE_ADMIN can create and update Invoices"

Without this package, you'd have to write messy if/else role checks scattered all over the code.

---

### Step 3 — Create the `CaslModule` and `AbilityFactory`

**Files to create:**

```
src/casl/
  casl.module.ts
  casl-ability.factory.ts
```

**What is the AbilityFactory?**
It is a single service that takes a user (with their roles) and returns a list of things they are allowed to do. Think of it as the rulebook.

**Example rules (simplified):**

```ts
// ADMIN → can do anything
if (user.roles.includes("ADMIN")) {
  can("manage", "all");
}

// HR → can read and update Users
if (user.roles.includes("HR")) {
  can("read", "User");
  can("update", "User");
}

// FINANCE → can only read Finance records
if (user.roles.includes("FINANCE")) {
  can("read", "FinanceRecord");
}

// FINANCE_ADMIN → can read, create, and update Finance records
if (user.roles.includes("FINANCE_ADMIN")) {
  can("read", "FinanceRecord");
  can("create", "FinanceRecord");
  can("update", "FinanceRecord");
}

// EMPLOYEE → can only read their own profile
if (user.roles.includes("EMPLOYEE")) {
  can("read", "User", { id: user.id });
}
```

**Why one central place?**
If permissions change in the future, you only update this one file. No hunting through controllers.

---

### Step 4 — Create a JWT Guard and JWT Strategy

**Files to create:**

```
src/auth/
  jwt.guard.ts
  jwt.strategy.ts
```

**What does the JWT Guard do?**
It sits in front of every protected route. When a request comes in, it:

1. Reads the `Authorization: Bearer <token>` header
2. Validates the token using the JWT secret
3. Extracts the user info (id, email, roles) from the token
4. Attaches the user to the request object so the next guard can use it

**What is the JWT Strategy?**
It is the logic that actually decodes and validates the token. The guard uses the strategy under the hood.

**Why do we need this?**
Currently, NO route is protected. Anyone can call any endpoint without a token. This guard closes that gap.

**Work-Flow**
Request comes in
↓
JWT Guard triggers
↓
calls Strategy → "is this token valid? who is this?"
↓
Strategy decodes token → returns { sub, email, roles }
↓
Guard attaches that user to req.user → lets request through
↓
Policies Guard triggers (Step 5)
↓
asks CaslAbilityFactory → "can this user do THIS action?"
↓
YES → hits the actual controller/route
NO → 403 Forbidden

---

### Step 5 — Create the `PoliciesGuard` and `@CheckPolicies` Decorator

**Files to create:**

```
src/casl/
  policies.guard.ts
  check-policies.decorator.ts
```

**What is the PoliciesGuard?**
A second guard that runs AFTER the JWT guard. It asks the AbilityFactory: "Given this user's roles, are they allowed to do what they're trying to do?"

**What is the `@CheckPolicies` decorator?**
A simple decorator you put on a controller method to declare what permission is required.

**Example usage on a controller:**

```ts
@UseGuards(JwtGuard, PoliciesGuard)
@CheckPolicies((ability) => ability.can('read', 'FinanceRecord'))
getFinanceRecords() {
  // only FINANCE and FINANCE_ADMIN users reach here
}
```

**Why two guards instead of one?**

- JWT guard → answers "who are you?"
- Policies guard → answers "are you allowed to do this?"

Keeping them separate means you can use the JWT guard alone on routes that just need login (no specific role needed), and add the Policies guard only where role-based access matters.

| JWT Guard                   | Policies Guard                               |
| --------------------------- | -------------------------------------------- |
| Confirms you're logged in   | Confirms you're allowed                      |
| Checks: Is the token valid? | Checks: Does this role/user have permission? |
| On fail: `401 Unauthorized` | On fail: `403 Forbidden`                     |
| Lives in `src/auth/`        | Lives in `src/casl/`                         |

---

### Step 6 — Apply Guards to Controllers

**What to do:**
As new controllers are added (users, finance records, etc.), decorate each route with the appropriate guards and policy checks.

**Pattern:**

```ts
@Controller('finance')
@UseGuards(JwtGuard)           // all finance routes require login
export class FinanceController {

  @Get()
  @UseGuards(PoliciesGuard)
  @CheckPolicies((ab) => ab.can('read', 'FinanceRecord'))
  getAll() { ... }             // FINANCE and FINANCE_ADMIN

  @Post()
  @UseGuards(PoliciesGuard)
  @CheckPolicies((ab) => ab.can('create', 'FinanceRecord'))
  create() { ... }             // FINANCE_ADMIN only

}
```

**Why apply at the controller level?**
NestJS guards are declarative — you describe what is needed, not how to check it. This keeps controllers clean and the permission logic stays in the AbilityFactory.

---

## File Structure After Full Integration

```
backend/src/
├── auth/
│   ├── auth.controller.ts
│   ├── auth.service.ts
│   ├── auth.module.ts
│   ├── jwt.guard.ts           ← NEW
│   └── jwt.strategy.ts        ← NEW
├── casl/
│   ├── casl.module.ts         ← NEW
│   ├── casl-ability.factory.ts ← NEW
│   ├── policies.guard.ts      ← NEW
│   └── check-policies.decorator.ts ← NEW
└── prisma/
    ├── prisma.module.ts
    └── prisma.service.ts
```

---

## Order of Implementation (Do Not Skip Steps)

```
1. Add FINANCE_ADMIN to Role enum → migrate DB
2. Install @casl/ability
3. Create CaslModule + AbilityFactory (the rulebook)
4. Create JwtGuard + JwtStrategy (identity check)
5. Create PoliciesGuard + @CheckPolicies (permission check)
6. Apply to controllers as features are built
```

The order matters because:

- Roles must exist before you can write rules about them
- The AbilityFactory must exist before the PoliciesGuard can use it
- Both guards must exist before you can protect any route
