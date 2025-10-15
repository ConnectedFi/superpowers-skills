---
name: TypeScript Type Safety
description: Avoid non-null assertions and type hacks. Use proper type narrowing with runtime checks in ALL code.
when_to_use: Writing any TypeScript code (tests, domain, service, UI). Fixing Biome warnings about non-null assertions. When tempted to use `!` or `as` casts. Before committing code.
version: 2.0.0
---

# TypeScript Type Safety

## Overview

All TypeScript code must have proper type safety. Non-null assertions (`!`) and type casts (`as`) are code smells that hide potential bugs.

**Core principle:** If the value could be null at runtime, don't pretend it can't be.

## The Iron Rules

```
1. NO non-null assertions (!) in tests
2. NO type casts (as) without runtime validation
3. USE explicit runtime checks for type narrowing
```

## Anti-Pattern: Non-Null Assertions

**The violation:**
```typescript
// ❌ BAD: Non-null assertion
const person = await db.person.findFirst({ where: { id: personId } });
expect(result.uuid).toBe(person!.uuid);  // What if person is null?

const user = await db.user.findFirst({ where: { type: "DEALER" } });
context.user = STSessionUser.one({ userId: user!.userId });  // Crash if null!
```

**Why this is wrong:**
- Hides the fact that value could be null
- Test passes with `!`, but crashes at runtime if null
- Biome correctly warns: "Forbidden non-null assertion"
- False confidence - you're asserting something you haven't verified

**The fix:**
```typescript
// ✅ GOOD: Explicit runtime check
const person = await db.person.findFirst({ where: { id: personId } });
if (!person) throw new Error("Test setup failed: Person not found");
expect(result.uuid).toBe(person.uuid);  // TypeScript knows it's not null

const user = await db.user.findFirst({ where: { type: "DEALER" } });
if (!user) throw new Error("Test setup failed: No DEALER user found");
context.user = STSessionUser.one({ userId: user.userId });  // Safe!
```

## Pattern: Type Narrowing with Runtime Checks

**Always use if-throw pattern for type narrowing:**

```typescript
// Query returns potentially null value
const entity = await db.entity.findUnique({ where: { uuid } });

// Narrow type with runtime check
if (!entity) {
  throw new Error("Entity not found in test setup");
}

// TypeScript now knows entity is non-null
expect(entity.name).toBe("Expected Name");
entity.someMethod();  // No errors!
```

**For nested properties:**
```typescript
const result = await someFunction();

// Check each level
if (!result.data) {
  throw new Error("Expected result.data to exist");
}

if (!result.data.address) {
  throw new Error("Expected address to exist");
}

// Now TypeScript knows both exist
expect(result.data.address[0].street).toBe("123 Main St");
```

## Pattern: Optional Properties in Tests

**When property is actually optional:**

```typescript
// ❌ BAD: Assertion on optional property
expect(person.middleName!).toBe("Marie");

// ✅ GOOD: Check if it exists first
if (person.middleName) {
  expect(person.middleName).toBe("Marie");
} else {
  throw new Error("Expected middleName to be present");
}

// OR use optional chaining if it's okay for it to be missing
expect(person.middleName).toBeUndefined();  // Explicitly test it's missing
```

## Pattern: Array Access

**When accessing array elements:**

```typescript
// ❌ BAD: Assume array has elements
expect(result.addresses![0].street).toBe("123 Main St");

// ✅ GOOD: Verify array and length first
if (!result.addresses || result.addresses.length === 0) {
  throw new Error("Expected at least one address");
}

expect(result.addresses[0].street).toBe("123 Main St");
```

## Pattern: Test Setup Failures

**Make test setup failures explicit:**

```typescript
// ❌ BAD: Silent failure with !
const user = users.find(u => u.role === "ADMIN")!;

// ✅ GOOD: Clear error message
const user = users.find(u => u.role === "ADMIN");
if (!user) {
  throw new Error(
    "Test setup failed: No ADMIN user in database. " +
    "Run seed script or check test data."
  );
}
```

**Benefits:**
- Test fails with clear message about what's wrong
- Helps debug test setup issues
- Documents test requirements

## Common Scenarios

### Scenario 1: Database Query Results

```typescript
// Query user for test
const dealer = await db.user.findFirst({
  where: { type: "DEALER" },
});

if (!dealer) {
  throw new Error(
    "Test requires DEALER user. Run: bun run seed:test-users"
  );
}

// Now safely use dealer
const context = { user: STSessionUser.one({ userId: dealer.userId }) };
```

### Scenario 2: Testing API Responses

```typescript
// Call endpoint
const result = await $getPersonDetailsUS.callable({ input, context });

// Verify structure exists before accessing
expect(result).toBeDefined();
expect(result.address).toBeDefined();

if (!result.address || result.address.length === 0) {
  throw new Error("Expected person to have at least one address");
}

expect(result.address[0].state).toBe("NY");
```

### Scenario 3: Testing Updates

```typescript
// Update entity
await $updatePerson.callable({ input, context });

// Fetch to verify
const updated = await db.person.findUnique({
  where: { uuid: personUuid },
  include: { address: true },
});

if (!updated) {
  throw new Error("Person should exist after update");
}

expect(updated.first_name).toBe("Jane");

if (!updated.address || updated.address.length === 0) {
  throw new Error("Expected updated person to have address");
}

expect(updated.address[0].street).toBe("456 Oak Ave");
```

## Why This Matters

**Type safety in tests prevents:**
1. **Silent null/undefined crashes** - Test passes, production breaks
2. **Misleading test results** - Test "passes" but doesn't actually test anything
3. **Hard-to-debug failures** - "Cannot read property 'uuid' of null" at line 127
4. **False confidence** - Think code works because tests pass with `!`

**Proper type narrowing gives:**
1. **Clear error messages** - "Test setup failed: No DEALER user" vs generic crash
2. **Type safety** - TypeScript catches errors at compile time
3. **Self-documenting** - Tests show exactly what they require
4. **Maintainable** - Next developer knows what test expects

## Red Flags

- Any `!` in test files (except TypeScript non-null after explicit check)
- `as` casts without runtime validation
- Tests that pass but crash on certain data
- "Cannot read property X of null/undefined" in test failures
- Biome warnings about non-null assertions

## Integration with Other Skills

- **testing-anti-patterns** - Don't test mock behavior
- **test-driven-development** - Write tests first, watch them fail
- **verification-before-completion** - Run tests before claiming done

## Quick Checklist

Before committing test code:
- [ ] No `!` non-null assertions
- [ ] No `as` type casts without validation
- [ ] All potentially-null values have runtime checks
- [ ] Clear error messages for test setup failures
- [ ] Biome/TypeScript diagnostics clear
- [ ] Tests pass and are type-safe

## The Bottom Line

**If you need `!` in a test, you haven't verified the value exists.**

Fix: Add explicit runtime check, narrow the type properly, get clear errors when assumptions break.
