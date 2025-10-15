---
name: Progressive Test Composition
description: Build test setups progressively using tested components, with composable cleanup in reverse dependency order
when_to_use: Creating test helpers. Setting up test data. Writing integration tests. When test setup gets complex or cleanup order is tricky. Before duplicating setup code across tests. When domain function just passed its tests.
version: 1.0.0
---

# Progressive Test Composition

## Overview

Once a component is tested, it becomes a safe building block for test setup. Build test helpers that return `{setup, cleanup}` where cleanup automatically handles dependencies in reverse order.

**Core principle:** Test at each layer, then use that tested layer as setup for testing the next layer.

## When to Use

Use this pattern when:
- Creating test helpers for reuse
- Domain function just passed integration tests → Graduate it to a helper
- Test setup is duplicated across multiple tests
- Cleanup ordering is complex (foreign key constraints, dependencies)
- Testing higher-level features that depend on lower-level components

When NOT to use:
- Simple one-off tests with no dependencies
- Testing the base service layer itself (use direct DB)

## The Test Tree Concept

Tests form a dependency tree where each level builds on tested components:

```
Level 1: Service layer tested
  $$createDeal() → Safe to use in $setupDeal helper
  ├─ Tests pass → Proves it works
  └─ Helper uses it for setup → Reuses proven code

Level 2: Domain layer tested (uses Level 1)
  $createAndAttachPerson() → Graduates to $setupPersonViaDomain helper
  ├─ Uses $$createDeal from Level 1 (setup)
  ├─ Tests pass → Proves domain logic + permissions work
  └─ Helper uses it for E2E setup → Full integration tested

Level 3: Higher features (uses Level 1 + 2)
  $attachDocumentToPerson() → Safe to use in setup
  ├─ Uses $setupPersonViaDomain from Level 2 (setup)
  ├─ Tests pass → Proves document flow works
  └─ Used in E2E tests → Complete user journey tested
```

**Key insight:** Each level's helpers use the previous level's TESTED code. Setup mirrors production state with all business logic + validation + permissions running correctly.

## When to Graduate from Service to Domain

**Answer:** As soon as the happy path test passes.

- Don't need comprehensive coverage first
- Just need basic integration test proving it works
- Then use it everywhere for setup
- Add edge case tests later as bugs are found (RED → GREEN pattern)

**Example timeline:**
1. Write `$createAndAttachPerson` domain function
2. Write happy path test → PASSES ✅
3. **Graduate NOW** → Create `$setupPersonViaDomain` using the domain function
4. Use graduated helper in E2E tests
5. Later: Add edge case tests as bugs surface

## The {setup, cleanup} Pattern

### Structure

Each helper returns:
```typescript
{
  setup: ExtendedContext,          // Context + new entities
  cleanup: () => Promise<void>     // Async cleanup function
}
```

### Rules

1. **Each helper cleans up ONLY what IT creates**
   - The entity itself
   - Relationships it created
   - NOT parent entities (those have their own cleanup)

2. **Composite helpers chain cleanup calls**
   - Call child cleanup first
   - Call parent cleanup last
   - This creates reverse dependency order automatically

3. **Tests call single cleanup function**
   - No manual ordering needed
   - Cleanup chain handles dependencies

### Example: Base Helper (Service Layer)

```typescript
// e2e/helpers/$test-setup/deal.ts
export async function $setupDeal(
  ctx: TestSetupContext,
  overrides?: { type?: string }
): Promise<SetupResult<TestSetupContext & { deal: Deal }>> {
  const stubData = STDeal.one();

  // Use service function (Level 1 - service layer)
  const deal = await $$createDeal({
    type: overrides?.type || stubData.type,
    companyId: ctx.companyId,
    ownerId: ctx.user.userId,
  });

  return {
    setup: { ...ctx, deal },
    cleanup: async () => {
      // Clean up ONLY what this helper created
      // 1. Relationships first
      await db.dealLink.deleteMany({ where: { dealId: deal.id } });
      // 2. Then the entity
      await db.deal.delete({ where: { id: deal.id } });
    },
  };
}
```

### Example: Domain Helper (Graduated)

```typescript
// e2e/helpers/$test-setup/person.ts
import { call } from "@orpc/server";
import { appRouter } from "~/api/router";

/**
 * Setup person using DOMAIN function (graduated - tested integration)
 *
 * Use when: E2E tests, testing higher-level features
 * Uses: $createAndAttachPerson (domain layer with full business logic)
 */
export async function $setupPersonViaDomain(
  ctx: TestSetupContext & { deal: Deal },
  overrides?: {
    first_name?: string;
    email?: string | null;
    attachToOrganization?: boolean;
  }
): Promise<SetupResult<TestSetupContext & { deal: Deal; person: Person }>> {
  const stubData = STPerson.one();
  const userContext = STSessionUser.one({
    userId: ctx.user.userId,
    companyId: ctx.user.companyId,
    type: ctx.user.type,
  });

  // Use tested domain function (Level 2 - includes permissions + business logic)
  const result = await call(
    appRouter.entities.createAndAttachPerson,
    {
      dealUuid: ctx.deal.uuid,
      first_name: overrides?.first_name || stubData.first_name || "",
      last_name: stubData.last_name || "",
      email: overrides?.email ?? stubData.email ?? null,
      phone: stubData.phone ?? null,
      attachToOrganization: overrides?.attachToOrganization ?? false,
    },
    { context: { user: userContext } }
  );

  const person = await db.person.findUnique({
    where: { uuid: result.personUuid },
  });

  if (!person) throw new Error("Person creation failed in domain function");

  return {
    setup: { ...ctx, person },
    cleanup: async () => {
      // Clean up ONLY what this helper created
      // Domain function created these relationships
      // 1. PersonOnOrganizations first
      await db.personOnOrganizations.deleteMany({
        where: { personId: person.id },
      });
      // 2. DealLinks
      await db.dealLink.deleteMany({ where: { personId: person.id } });
      // 3. Person itself
      await db.person.delete({ where: { id: person.id } });
      // Note: We DON'T clean up the deal - that's the parent's job
    },
  };
}
```

### Example: Service Helper (For Testing Domain)

```typescript
// e2e/helpers/$test-setup/person.ts

/**
 * Setup person using SERVICE functions
 *
 * Use when: Testing domain layer functions, need low-level control
 * Uses: $$createPerson + $$attachEntityToDeal (service layer - no business logic)
 */
export async function $setupPersonViaService(
  ctx: TestSetupContext & { deal: Deal },
  overrides?: {
    first_name?: string;
    roles?: string[];
  }
): Promise<SetupResult<TestSetupContext & { deal: Deal; person: Person }>> {
  const stubData = STPerson.one();

  // Use service functions directly (bypassing domain logic)
  const person = await db.$transaction(async (tx) => {
    const p = await $$createPerson(tx, {
      first_name: overrides?.first_name || stubData.first_name || "",
      last_name: stubData.last_name || "",
      email: stubData.email || null,
      phone: stubData.phone || null,
      creatorId: ctx.user.userId,
    });

    await $$attachEntityToDeal(tx, {
      dealId: ctx.deal.id,
      personId: p.id,
      roles: overrides?.roles || ["Applicant"],
    });

    return p;
  });

  return {
    setup: { ...ctx, person },
    cleanup: async () => {
      // Same cleanup as domain helper - it's the data that matters
      await db.personOnOrganizations.deleteMany({
        where: { personId: person.id },
      });
      await db.dealLink.deleteMany({ where: { personId: person.id } });
      await db.person.delete({ where: { id: person.id } });
    },
  };
}

// Default export - use domain helper (graduated, battle-tested)
export const $setupPerson = $setupPersonViaDomain;
```

### Example: Composite Helper (Chained Cleanup)

```typescript
// e2e/helpers/$test-setup/index.ts

export async function $setupCompleteScenario(overrides?: {
  deal?: Parameters<typeof $setupDeal>[1];
  organization?: Parameters<typeof $setupOrganization>[1];
  person?: Parameters<typeof $setupPersonViaDomain>[1];
}) {
  return await $runSetup(async (ctx) => {
    // Setup deal (Level 1)
    const { setup: withDeal, cleanup: cleanupDeal } = await $setupDeal(
      ctx,
      overrides?.deal
    );

    // Setup organization (uses Level 1)
    const { setup: withOrg, cleanup: cleanupOrg } = await $setupOrganization(
      withDeal,
      overrides?.organization
    );

    // Setup person (uses Level 2 - graduated domain function)
    const { setup: withPerson, cleanup: cleanupPerson } = await $setupPersonViaDomain(
      withOrg,
      overrides?.person
    );

    return {
      setup: withPerson,
      cleanup: async () => {
        // Cleanup in reverse order: child first, parent last
        await cleanupPerson(); // Cleans person + person's relationships
        await cleanupOrg(); // Cleans org + org's relationships
        await cleanupDeal(); // Cleans deal + deal's relationships
      },
    };
  });
}
```

### Test Usage

```typescript
describe("feature tests", () => {
  let cleanup: () => Promise<void>;
  let deal: Deal;
  let person: Person;

  beforeEach(async () => {
    const result = await $setupCompleteScenario({
      person: { first_name: "Jane" },
    });
    deal = result.setup.deal;
    person = result.setup.person;
    cleanup = result.cleanup;
  });

  afterEach(async () => {
    await cleanup(); // Automatically cleans person → org → deal
  });

  it("tests something", () => {
    // Test using deal and person
  });
});
```

## What Domain Tests Should Cover

Focus on **business logic integration**, not framework validation:

✅ **Test these:**
- Happy path (basic creation)
- Complex scenarios (multi-step operations)
- Transaction atomicity (rollback on failure)
- Each conditional branch in domain logic

❌ **Don't test these:**
- Framework behavior ("does Zod validate strings?")
- All permission combinations (focus on access layer tests)
- Mock behavior (test real code)

## Benefits

**Progressive composition:**
- Tested functions become building blocks
- Each level builds on previous verified layer
- Setup mirrors production with all business logic

**Performance:**
- 49% faster than manual cleanup (benchmarked: 40ms vs 79ms)
- Targeted cleanup queries vs complex WHERE clauses

**Context accumulation:**
- Each setup adds to test state
- Type-safe chaining
- Clear dependencies

**Automatic cleanup:**
- Reverse dependency order handled automatically
- No manual ordering
- Hard to get wrong

**Works with limitations:**
- No nested transaction support needed
- No production code changes
- Clean test syntax

## Helper Variants: When to Use Each

### Domain Helpers (Default for E2E)

**Use:** E2E tests, testing higher-level features

**Why:** Full business logic + permissions tested together

```typescript
const { setup, cleanup } = await $setupPerson(ctx, {...});
// Uses $createAndAttachPerson domain function
// Includes all validation, permissions, business rules
```

### Service Helpers (For Testing Domain Layer)

**Use:** Testing domain functions themselves, need low-level control

**Why:** Bypass business logic to set up test scenarios

```typescript
const { setup, cleanup } = await $setupPersonViaService(ctx, {...});
// Uses $$createPerson + $$attachEntityToDeal directly
// No domain logic - pure data setup
```

## Creating New Helpers

### Checklist

When creating a new test helper:

- [ ] Name it `$setupX` (matches entity name)
- [ ] Accept `TestSetupContext` (or extended version)
- [ ] Accept `overrides` parameter for test customization
- [ ] Use `STX` stub for realistic defaults
- [ ] Return `{setup: ExtendedContext, cleanup: () => Promise<void>}`
- [ ] Cleanup ONLY what THIS helper creates
- [ ] Document when to use domain vs service variant (if both exist)

### Decision Tree

```
Domain function exists and tested?
├─ YES → Create domain helper ($setupXViaDomain)
│         └─ Use for E2E tests, higher-level features
└─ NO → Create service helper ($setupXViaService)
          └─ Use for domain tests, low-level control

Common pattern emerged?
└─ YES → Create composite helper ($setupCompleteScenario)
          └─ Chains multiple helpers with automatic cleanup
```

## Red Flags

**Never:**
- Manual cleanup ordering in tests (use composite helper)
- Cleaning up parent in child helper (each helper only cleans what it creates)
- Not using tested functions for setup (defeats progressive composition)
- Skipping tests before using in setup (must test happy path first)
- Using domain helpers to test domain functions (circular - use service helpers)

**If you see:**
- Complex cleanup with numbered steps → Extract to composite helper
- Same setup duplicated across tests → Extract to helper
- Domain function just passed tests → Graduate to helper NOW
- Manual transaction rollback in tests → Let helpers handle cleanup

## Common Patterns

**Single entity:**
```typescript
const { setup, cleanup } = await $setupDeal(ctx);
const { deal } = setup;
```

**Chained entities (manual):**
```typescript
const { setup: s1, cleanup: c1 } = await $setupDeal(ctx);
const { setup: s2, cleanup: c2 } = await $setupPerson(s1);
// In afterEach: await c2(); await c1();
```

**Composite helper (recommended):**
```typescript
const { setup, cleanup } = await $setupCompleteScenario(ctx);
// Single cleanup() chains everything automatically
```

## Module Organization

```
e2e/helpers/$test-setup/
├── types.ts           - TestSetupContext, SetupResult
├── deal.ts            - $setupDeal (service-based)
├── organization.ts    - $setupOrganization (service-based)
├── person.ts          - $setupPerson + variants (domain + service)
└── index.ts           - Exports + composite helpers
```

**Why modular:**
- Clear separation by entity type
- Easy to find relevant helper
- Can import specific variants
- Supports domain + service variants in same module

## References

**Implementation:**
- See `e2e/helpers/$test-setup/` for full implementation
- See `e2e/tests/create-person-form.e2e.ts` for E2E usage examples
- See `app/api/server/endpoints/$entities.server.test.ts` for domain test examples

**Documentation:**
- See `CLAUDE.md` Testing Best Practices section
- See `e2e/helpers/$test-setup/index.ts` for pattern documentation

**Related Skills:**
- `skills/testing/test-driven-development` - RED-GREEN-REFACTOR cycle
- `skills/testing/testing-anti-patterns` - Never test mock behavior
