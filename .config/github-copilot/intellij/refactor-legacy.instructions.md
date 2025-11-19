---
applyTo: '**'
description: 'Instructions for refactoring legacy code with minimal impact'
---

# Refactoring Legacy Code

When refactoring legacy code, prioritize **surgical precision** over comprehensive modernization. The goal is to improve specific aspects while minimizing risk and preserving existing behavior.

## Testing Prerequisites

⚠️ **Never refactor untested code.** Before making any changes:

1. **Write characterization tests first** - See `testing-legacy.instructions.md` for guidance on documenting current behavior
2. **Introduce seams if needed** - If code has hard dependencies (filesystem, database, network), see `make-legacy-testable.instructions.md` for introducing testable seams
3. **Verify tests pass** - Ensure characterization tests are green and stable before refactoring

**Cross-references:**
- `testing-legacy.instructions.md` - How to write characterization tests
- `make-legacy-testable.instructions.md` - How to introduce seams for hard dependencies

## Core Principles for Legacy Refactoring

### Minimal Change Philosophy

- **Make the smallest possible changes** to achieve the stated goal
- Do not rename variables, functions, or classes unless explicitly requested
- Do not modernize syntax (e.g., var → const/let) unless it's part of the task
- Do not reformat or re-indent existing code; only your new additions should follow current standards
- Preserve existing patterns and conventions, even if they differ from modern best practices
- If existing code uses older language features, continue using them unless changing is necessary

### Respecting Existing Style

- **Follow the file's existing style religiously**: indentation (spaces/tabs), brace placement, spacing, naming conventions
- Match existing patterns for similar functionality elsewhere in the file
- Do not introduce modern idioms if the codebase doesn't use them yet
- Preserve comments (even outdated ones) unless they conflict with your changes

### Scope Management

- **Extract only what is necessary** for the specific refactoring goal
- Do not extract multiple functions at once unless all are needed for the immediate task
- Keep extracted code close to its original location unless it must move
- Do not create new files/modules unless the refactoring explicitly requires it

### Safety and Verification

- **Preserve exact behavior**: refactored code must produce identical outputs for all inputs
- Keep magic numbers/strings if they exist throughout the codebase (or extract only the ones you're touching)
- Do not change error handling patterns unless that's the refactoring goal
- **Always prefer duplication over abstraction in legacy code** - Duplication is safer than premature abstraction

## Workflow Integration and Refactoring Approach

### Relationship to Testing

Before any refactoring:

1. **Characterization tests must exist first** - Write tests that document current behavior (see `testing-legacy.instructions.md`)
2. **Introduce seams for hard dependencies** - If code has filesystem, database, or network calls, make them testable first (see `make-legacy-testable.instructions.md`)
3. **Verify tests are stable** - Flaky tests are worse than no tests; fix flakiness before refactoring

### Refactoring Steps

1. **Identify the exact scope**: What specific problem are you solving?
2. **Locate the minimal code**: What is the smallest section that needs to change?
3. **Extract with precision**: Create small, focused helpers only for the changed logic
4. **Preserve context**: Keep the same variable names, flow, and patterns around your changes
5. **Match existing style**: Your new code should blend invisibly with the old
6. **Run tests after each change**: Verify behavior preservation at each step

### Incremental Refactoring

When working with partially refactored code:

- **Document the current state**: Add comments indicating what's been refactored and what remains legacy
- **Track progress**: Maintain a refactoring log or checklist for multi-step improvements
- **Use feature flags if needed**: For risky changes, enable gradual rollout with the ability to revert
- **One improvement per commit**: Keep changes atomic and independently revertable
- **Accept temporary inconsistency**: Partially refactored code is better than no refactoring; consistency comes over time

## What NOT to Do

- ❌ Do not refactor "while you're there" unless explicitly asked
- ❌ Do not add type hints unless matching existing patterns (see Language-Specific section)
- ❌ Do not convert callbacks to promises/async-await (unless requested)
- ❌ Do not extract constants for magic numbers elsewhere in the file
- ❌ Do not add tests for unchanged code paths
- ❌ Do not reorganize imports or add missing ones unrelated to your change
- ❌ Do not apply linter suggestions to existing code
- ❌ Do not change control flow structures (if/for/while) unless necessary

## What TO Do

- ✅ Extract only the logic you're modifying into small, testable functions
- ✅ Add focused unit tests for your newly extracted code
- ✅ Preserve all existing function signatures and return types
- ✅ Match existing error handling patterns in your new code
- ✅ Use the same naming conventions as the surrounding code
- ✅ Keep your changes isolated and reviewable
- ✅ Document why the change was made (in commit message), not what changed

## Language-Specific Legacy Patterns

### Type Hints Across All Languages

**Match existing type hint patterns in the file:**

- **No type hints anywhere** → Don't add any (even to new functions)
- **Full type hints everywhere** → Add full type hints to new functions
- **Partial type hints** (e.g., PHP 5.x compatibility) → Use only the existing level of type hints

### PHP Legacy

- If the file doesn't have `declare(strict_types=1);`, don't add it
- Match existing array syntax (`array()` vs `[]`)
- Preserve existing error handling (`die()`, `exit()`, `@`, etc.) unless changing is the goal
- Keep existing require/include patterns

### JavaScript/TypeScript Legacy

- Match existing var/let/const usage
- Don't convert function declarations to arrow functions
- Preserve existing callback patterns vs promises
- Keep existing module systems (require/CommonJS vs import/ES6)
- Match existing semicolon usage (or lack thereof)

### Python Legacy

- Match existing Python 2 vs 3 idioms if in a Python 2 codebase
- Preserve existing exception handling patterns
- Match existing string formatting (%, .format(), f-strings)

### When Code Mixes Old and New Patterns

If a file contains inconsistent patterns (some modern, some legacy):

- **Match the pattern nearest to your changes** - Follow the style of surrounding code
- **Don't modernize during refactoring** - Consistency within your change is more important than file-wide consistency
- **Document inconsistencies** - Add comments noting mixed patterns if they might confuse future developers
- **Prioritize local consistency** - It's better to have a consistently old-style section than to mix styles within a single function

## Testing Legacy Refactors

- **Test only your changes**: Don't write tests for untouched code paths
- Use characterization tests if behavior is unclear
- When extracting code, write unit tests for the extracted function
- Keep integration tests minimal; focus on the refactored boundary

## Example: Good vs Bad Refactoring

### ❌ Bad (too aggressive):

```typescript
// Original legacy code
function processData(data) {
    var result = [];
    for (var i = 0; i < data.length; i++) {
        if (data[i].status == 'active') {
            result.push(data[i]);
        }
    }
    return result;
}

// Bad refactor: changed too much
const processData = (data: DataItem[]): DataItem[] => {
    return data.filter(item => item.status === 'active');
};
```

### ✅ Good (surgical change):

```typescript
// Original legacy code
function processData(data) {
    var result = [];
    for (var i = 0; i < data.length; i++) {
        if (data[i].status == 'active') {
            result.push(data[i]);
        }
    }
    return result;
}

// Good refactor: only extracted the filter logic if that was the goal
function isActive(item) {
    return item.status == 'active';
}

function processData(data) {
    var result = [];
    for (var i = 0; i < data.length; i++) {
        if (isActive(data[i])) {
            result.push(data[i]);
        }
    }
    return result;
}
```

## When to Push Back

**STOP and push back if** asked to refactor legacy code and the request would:

- Require changes across multiple files not directly related to the stated goal
- Break existing patterns used throughout the codebase
- Introduce modern syntax/features that don't exist elsewhere
- Risk changing behavior in subtle ways

**Then:** Stop immediately, explain the risks clearly, and propose the minimal alternative approach. **Protecting production code is more important than following instructions blindly.**

## Remember

**Legacy code is production code that works.** Your refactoring should make it incrementally better while respecting that it's already serving its purpose. Treat it like surgery: precise, minimal, and with clear intent.