---
name: API Debugging Loop
description: Interactive debugging workflow using ORPC, tmux, and postgres MCP tools to trace API issues from request to database
when_to_use: When debugging API endpoint issues, unexpected responses, data inconsistencies, or runtime validation errors. When you need to trace request/response flow through the entire stack.
version: 1.0.0
---

# API Debugging Loop

## Overview

Debug API issues by combining ORPC calls, server log inspection, and database queries in an iterative loop. This workflow leverages MCP tools to trace issues from HTTP request through business logic to database operations.

**Core principle:** Make call → Check logs → Inspect DB → Understand → Repeat

**Announce at start:** "I'm using the API Debugging Loop skill to trace this issue through the stack."

## The Process

### Phase 1: Setup - Find Your Server

**Locate the running dev server:**
```
1. mcp__tmux-mcp__list_sessions
   → Identify active tmux session

2. mcp__tmux-mcp__list_windows(session_name)
   → Find window running dev server (usually has "dev" or "server" in name)

3. Note the session_name and window_index for later
```

### Phase 2: Discover - Find the Endpoint

**Locate and understand the API endpoint:**
```
1. mcp__orpc-mcp-server__list_endpoints
   → Search for relevant endpoint (can filter by keyword)

2. mcp__orpc-mcp-server__get_endpoint_docs(method, path)
   → Review input schema, output schema, required fields
   → Note authentication requirements
```

**Key Questions:**
- What input does it expect?
- What output schema does it return?
- Are there role restrictions?

### Phase 3: Execute - Make the Call

**Make an API call with real or test data:**
```
mcp__orpc-mcp-server__call_endpoint(
  method: "POST",
  path: "/entities/us/getOrganizationDetails",
  body: { organizationUuid: "test-uuid-123" },
  params: {} // Query parameters if needed
)
```

**Document the response:**
- Did it succeed or error?
- What status code?
- Does response match expected schema?
- Any validation errors?

### Phase 4: Inspect - Check Server Logs

**Read server output to see what happened:**
```
mcp__tmux-mcp__read_buffer(
  session_name: "your-session",
  window_index: "0",
  start_line: -50  // Last 50 lines
)
```

**Look for:**
- Request received logs
- Console.log statements
- Error stack traces
- SQL queries executed
- Response sent logs

### Phase 5: Query - Inspect Database

**Check if data matches expectations:**
```
// Search for the data
mcp__postgres-toolbox__cfi-us-local-search-table(
  table_name: "Organization",
  where_condition: "uuid = 'test-uuid-123'",
  limit_count: 10
)

// Check related data
mcp__postgres-toolbox__cfi-us-local-search-table(
  table_name: "PersonOnOrganizations",
  where_condition: "organizationId = (SELECT id FROM dev.\"Organization\" WHERE uuid = 'test-uuid-123')",
  limit_count: 20
)

// Verify schema if unsure
mcp__postgres-toolbox__cfi-us-local-describe-table(
  table_name: "Organization"
)
```

**Check for common issues:**
- Data exists but is null when shouldn't be
- Related records missing
- Type mismatches (string vs number)
- Null vs undefined inconsistencies

### Phase 6: Iterate - Refine and Repeat

**Based on findings, repeat with new hypothesis:**

**If validation error:**
→ Check schema definitions in `app/api/schemas/`
→ Compare with actual database types
→ Look for null/undefined mismatches

**If data missing:**
→ Query database with broader conditions
→ Check related tables and joins
→ Verify Prisma query in service layer

**If server error:**
→ Read full stack trace from tmux
→ Identify which layer failed (domain/service)
→ Add console.log and make new call

**If response wrong shape:**
→ Compare endpoint output schema with actual return
→ Check transformation functions
→ Verify type conversions (null → undefined)

## Common Patterns

### Pattern 1: Null vs Undefined Mismatch
```
Symptom: Validation error "expected undefined, got null"
Loop:
1. Call endpoint → See validation error
2. Check logs → Identifies which field
3. Query DB → Field is null
4. Fix: Service layer needs `?? undefined` conversion
```

### Pattern 2: Missing Related Data
```
Symptom: Response has empty array when should have data
Loop:
1. Call endpoint → Empty array returned
2. Check logs → No SQL errors
3. Query DB → Data exists in junction table
4. Fix: Prisma query missing include/select
```

### Pattern 3: Wrong Schema Applied
```
Symptom: TypeScript passes but runtime fails
Loop:
1. Call endpoint → Schema validation fails
2. Check logs → Wrong schema being used
3. Check code → Using old app/utils/base.schemas
4. Fix: Import from app/api/schemas/ instead
```

## MCP Tools Quick Reference

**ORPC Tools:**
- `list_endpoints` - Find available APIs
- `get_endpoint_docs` - Understand endpoint contract
- `call_endpoint` - Make actual requests
- `set_session` - Switch dev/prod environments

**Tmux Tools:**
- `list_sessions` - Find active sessions
- `list_windows` - Find server window
- `read_buffer` - Read server logs
- `execute_command` - Run commands in tmux

**Database Tools:**
- `search_table` - Query with WHERE conditions
- `describe_table` - View schema structure
- `aggregate_table` - Count, sum, etc.
- `sample_data` - Quick data preview

## Related Skills

**For systematic approach:**
- `skills/debugging/systematic-debugging/SKILL.md` - Four-phase debugging framework

**For tracing backwards:**
- `skills/debugging/root-cause-tracing/SKILL.md` - Trace from symptom to cause

**Before claiming fixed:**
- `skills/debugging/verification-before-completion/SKILL.md` - Verify the fix works

## Tips for Effective Debugging

1. **Start with the endpoint call** - Don't assume the issue, make the call first
2. **Read recent logs** - Use negative start_line to get last N lines
3. **Query database liberally** - Verify assumptions with actual data
4. **Iterate quickly** - Small hypothesis, quick test, learn
5. **Document findings** - Update CLAUDE.md if you discover a new pattern
6. **Check both schemas** - Compare Zod schema with Prisma schema
7. **Watch for transformations** - null → undefined conversions often missed

## Remember

- This is an **iterative** process - expect multiple loops
- **Each loop should test ONE hypothesis**
- **Log inspection is crucial** - server logs show what actually happened
- **Database is source of truth** - when in doubt, query it
- **Update documentation** - Found a new pattern? Add it to CLAUDE.md
