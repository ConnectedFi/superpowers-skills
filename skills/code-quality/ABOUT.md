# Code Quality Skills

Best practices for writing clean, maintainable, type-safe code that applies to both production and test code.

## Skills in this Category

**typescript-type-safety**
- NO non-null assertions (`!`) without runtime validation
- NO type casts (`as`) without runtime checks
- USE explicit type narrowing: `if (!x) throw new Error()`
- Clear error messages for setup failures
- Applies to ALL TypeScript code (tests, domain, service, UI)

## Why This Category Exists

While testing and architecture have dedicated categories, there was no home for low-level coding practices that apply universally:
- Type safety patterns
- Error handling conventions
- Naming standards
- Code formatting rules

These are tactical, day-to-day coding rules that ensure consistency and catch bugs early.

## When to Add Skills Here

Add skills for:
- Language-specific best practices (TypeScript, Python, etc.)
- Code patterns that prevent common bugs
- Standards that apply across production and test code
- Low-level syntax and style guidelines

Don't add:
- High-level design decisions (use `architecture/`)
- Test-specific patterns (use `testing/`)
- Debugging techniques (use `debugging/`)
