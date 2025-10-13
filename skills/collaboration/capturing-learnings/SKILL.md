---
name: Capturing Learnings
description: Document non-trivial solutions to build institutional memory and reduce trial-and-error over time
when_to_use: After completing each task in subagent-driven-development. When stuck and searching for solutions. When you tried 3+ approaches before finding what worked. After debugging non-obvious issues.
version: 1.0.0
---

# Capturing Learnings

## Overview

Build institutional memory by documenting non-trivial solutions as you discover them.

**Core principle:** Search existing learnings when stuck. Capture new learnings after solving. Review and promote patterns to best practices.

**Three-tier hierarchy:**
1. **Task level**: Append to `docs/learning-queue.md` after each task
2. **Feature level**: Review queue → identify patterns → promote to CLAUDE.md
3. **Internal memory**: Search queue + CLAUDE.md when stuck during implementation

## When to Use

**During task (when stuck):**
- "Wait, this isn't right - it should have worked but it doesn't"
- Getting unexpected errors after following standard patterns
- Type errors, test failures, authorization bugs that aren't obvious
- **Action**: Search `docs/learning-queue.md` + `CLAUDE.md` for existing solutions

**After task (always ask):**
- "What non-trivial challenges (where I tried 3+ different approaches) did I encounter?"
- "What was the correct solution and how would I search for it in future?"
- **Action**: Append structured entry to `docs/learning-queue.md`

**After feature (human review):**
- Review all learnings from the feature
- Spot patterns across multiple tasks
- Promote recurring patterns to CLAUDE.md best practices
- Archive processed learnings

## Learning Queue Entry Format

Append to `docs/learning-queue.md`:

```markdown
## [Task Name] - YYYY-MM-DD

**Problem:** [Clear description of what went wrong or was non-obvious]
**Layer:** [db/service/domain/ui/test/e2e] - `path/to/file.ts:linenum`
**Search keywords:** [Keywords you'd use to find this solution]
**Solution:** [What actually worked and why]

---
```

**Example:**

```markdown
## Person Endpoint Authorization - 2025-01-10

**Problem:** First implementation allowed any user to access any person's data. Authorization check was missing.
**Layer:** domain - `app/api/server/endpoints/$people.us.server.ts:45`
**Search keywords:** authorization, access control, withPerson, loadPerson, DealAccess
**Solution:** Use `withPerson()` middleware that provides `loadPerson(uuid)` function. Service layer checks: user created person OR person is on a deal user can access (via DealAccess pattern). See `$$getPersonForUser` in service layer.

---
```

## Quick Reference

| Action | When | What to Document |
|--------|------|------------------|
| **Search** | Stuck during implementation | Check learning-queue.md + CLAUDE.md first |
| **Capture** | After each task | Non-trivial challenges (3+ attempts) |
| **Review** | After feature complete | Patterns across multiple learnings |
| **Promote** | Patterns identified | Move to appropriate CLAUDE.md section |

## Implementation

### During Task: Search First

**When stuck, before trial-and-error:**

```bash
# Search learning queue
grep -i "authorization\|access control" docs/learning-queue.md

# Search CLAUDE.md
grep -i "authorization" CLAUDE.md
```

Or announce: "I'm searching existing learnings for [keywords]..."

### After Task: Reflect and Capture

**Before reporting task complete, ask yourself:**

> "What non-trivial challenges (where I tried 3+ different approaches) did I encounter, and what was the correct solution?"

**If none:** Report task complete.

**If any:** Append to `docs/learning-queue.md` using format above.

**Critical fields:**
- **Problem**: What was wrong? Be specific.
- **Layer**: Where in architecture? (db/service/domain/ui/test/e2e)
- **Search keywords**: How would future you search for this?
- **Solution**: What worked and why? Reference files/patterns.

### After Feature: Pattern Recognition (Human-Led)

Human reviews `docs/learning-queue.md` for completed feature:

1. **Identify patterns**: Similar solutions across multiple tasks?
2. **Promote to CLAUDE.md**: Add to appropriate section (API Architecture, Testing, Debugging)
3. **Archive**: Remove processed learnings from queue

## Common Mistakes

### Teaching vs Archiving

**Problem:** Explain pattern to human but don't capture for future agents

```markdown
❌ BAD:
"Here's the best practice: use explicit error handling instead of `!`"
[explains to human, doesn't document]

✅ GOOD:
"Here's the solution: [explains]"
[Appends to learning-queue with search keywords]
```

### No Search Documentation

**Problem:** Found solution but didn't document HOW you found it

```markdown
❌ BAD:
**Search keywords:** [empty or vague]

✅ GOOD:
**Search keywords:** non-null assertion, possibly null, type narrowing, explicit error handling
```

### Missing Layer Context

**Problem:** Don't specify where in architecture this applies

```markdown
❌ BAD:
**Layer:** [empty]

✅ GOOD:
**Layer:** test - `app/api/server/endpoints/$people.us.server.test.ts:45`
```

### Time-Focused Reporting

**Problem:** Report "done in 15min" not "learned X pattern"

```markdown
❌ BAD:
"Implementation complete. Took 15 minutes as requested."

✅ GOOD:
"Implementation complete. Non-trivial challenge: authorization pattern discovery.
[Appends learning to queue]"
```

## Red Flags

**Never:**
- Skip reflection after task with challenges
- Document solution without search keywords
- Explain pattern without capturing it
- Report time spent without documenting learnings gained

**Always:**
- Search existing learnings when stuck
- Document layer + search keywords + solution
- Reflect before reporting task complete
- Make learnings findable for future agents

## Integration with Other Skills

**Called by:**
- `skills/collaboration/subagent-driven-development` - After each task
- `skills/collaboration/writing-plans` - Reads queue to synthesize context per-task

**Pairs with:**
- `skills/collaboration/remembering-conversations` - Search past work
- `skills/debugging/systematic-debugging` - Capture root causes found
