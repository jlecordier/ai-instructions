---
applyTo: '**'
description: 'Focused guide for testing legacy code safely using characterization tests'
---

# Testing Legacy Code

> **TL;DR**: Never change untested code. Write characterization tests first, refactor to extract testable units, then add unit tests. Coverage follows change.

**Cross-references**: 
- For introducing seams (making untestable code testable): see `make-legacy-testable.instructions.md`
- For minimal change philosophy: see `refactor-legacy.instructions.md`
- For general testing strategy: see `global-copilot-instructions.md`

---

## Fundamental Rule

**🚨 NEVER change code that is not tested. No exceptions.**

If code lacks tests, you must write characterization tests first, then refactor. This is non-negotiable.

---

## Core Principles

1. **Coverage follows change**: Test only what you're about to modify now, not the entire codebase
2. **Characterization first**: Document current behavior (even if wrong) before any refactoring
3. **Incremental path to 100%**: Each refactoring cycle adds coverage; goal is full coverage over time
4. **Match existing patterns**: Use the project's test framework, assertion style, and organization
5. **Tests are temporary scaffolding**: Characterization tests document legacy behavior; archive them after full unit/integration coverage exists

---

## The Incremental Path

```
┌─────────────────────────────────────────────────────────────┐
│ Untested Legacy Codebase (0% coverage)                      │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Need to change function A                                   │
│ → Use decision tree: "Should I characterize this?"          │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Write characterization tests for A                          │
│ → Document current behavior (with real dependencies)        │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌──────────────────────────────────────────────────────────────────┐
│ Refactor A: Extract pure logic, introduce seams                  │
│ → See make-legacy-testable.instructions.md for hard dependencies │
└────────────────────┬─────────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Write unit tests for extracted logic                        │
│ → Keep 1-2 integration tests at boundary                    │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Function A: Full coverage achieved                          │
│ → Archive characterization tests                            │
│ → Update coverage tracking                                  │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Repeat for function B, C, D...                              │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│ Eventually: Full Codebase Coverage (100%)                   │
└─────────────────────────────────────────────────────────────┘
```

---

## When to Characterize (Decision Tree)

```
Are you about to change this code?
│
├─ NO → Don't write tests yet (coverage follows change)
│
└─ YES → Is the code already tested?
           │
           ├─ YES → Verify tests pass, then proceed with changes
           │
           └─ NO → Can you understand the code by reading it?
                    │
                    ├─ YES (simple, low risk) → Use risk matrix below
                    │
                    └─ NO (complex, unclear) → MUST characterize
                                                (tests = documentation)
```

**Risk Matrix** (for simple code):

| Change Frequency | Risk Level      | Action                          |
|------------------|-----------------|---------------------------------|
| High             | High/Medium/Low | **Always characterize**         |
| Medium           | High            | **Always characterize**         |
| Medium           | Medium/Low      | Characterize if time allows     |
| Low              | High            | **Always characterize**         |
| Low              | Medium/Low      | Optional (document decision)    |

**Risk factors**: Critical path (payments, security), complex logic, external dependencies, known bugs

---

## When to Stop Characterizing

> **TL;DR**: Stop when you have enough tests to refactor safely, not when you've tested everything.

Stop writing more characterization tests when:

1. **All code paths you're modifying are covered** (not the entire function, just what you'll change)
2. **You understand the current behavior** (tests document it clearly)
3. **Tests pass consistently** (no flakiness—see "Dealing with Flaky Tests" below)
4. **Time-boxed limit reached** (if tests take >2x the refactoring time, consider alternatives)

**Alternative to characterization** (for untestable code):
- Introduce seams first (see `make-legacy-testable.instructions.md`)
- Then write characterization tests with injected test doubles

---

## Writing Characterization Tests

> **TL;DR**: Document what the code DOES, not what it SHOULD do.

### Golden Rule
**Characterization tests describe reality, not intent.** Assert on actual behavior, even if it seems wrong.

### Patterns

<details>
<summary><strong>Functions with Side Effects</strong></summary>

```typescript
it('should write to cache.txt and return count', () => {
    const result = legacyFunction('input.txt');
    
    expect(result).toBe(42);
    expect(fs.readFileSync('cache.txt', 'utf8')).toBe('42');
});
```
</details>

<details>
<summary><strong>Complex Business Logic (Unknown Behavior)</strong></summary>

```typescript
it('should calculate discount using legacy algorithm', () => {
    const result = calculateDiscount(100, 5, true);
    
    // Document actual result from execution (even if you don't understand why)
    expect(result).toBe(73.5);
});
```
</details>

<details>
<summary><strong>Hard-Coded Dependencies (Before Seam Introduction)</strong></summary>

```typescript
it('should read from real database and transform data', () => {
    // Test with real dependency initially
    const result = legacyFunction();
    
    expect(result.length).toBeGreaterThan(0);
    expect(result[0]).toHaveProperty('transformedField');
});
```

**After introducing seams** (see `make-legacy-testable.instructions.md`):

```typescript
it('should read from repository and transform data', () => {
    const repo = new InMemoryRepository([{ id: 1, raw: 'data' }]);
    const result = legacyFunction(repo);
    
    expect(result.length).toBe(1);
    expect(result[0].transformedField).toBe('PROCESSED_DATA');
});
```
</details>

---

## After Characterization

> **TL;DR**: Extract logic → Unit test extracted code → Keep 1-2 integration tests → Archive characterization tests.

### Step-by-Step Refactoring (with Tests)

```typescript
// Step 1: Characterization test (before any changes)
describe('LegacyUserService - characterization', () => {
    it('should hash password with md5 when creating user', () => {
        const service = new LegacyUserService();
        const result = service.createUser('test@example.com', 'password123');
        
        // Document current (bad) behavior
        expect(result.passwordHash).toBe('482c811da5d5b4bc6d497ffa98491e38');
    });
});

// Step 2: Extract pure logic → Unit test
describe('PasswordHasher', () => {
    it('should hash password with bcrypt', () => {
        const hasher = new PasswordHasher();
        const hash = hasher.hash('password123');
        
        expect(hash).toMatch(/^\$2[aby]\$/);
    });
});

// Step 3: Integration test at boundary
describe('LegacyUserService - integration', () => {
    it('should create user with hashed password', () => {
        const service = new LegacyUserService(new PasswordHasher());
        const result = service.createUser('test@example.com', 'password123');
        
        expect(result.passwordHash).toMatch(/^\$2[aby]\$/);
    });
});

// Step 4: Archive characterization test (move to tests/legacy/ or delete)
```

### When to Remove Characterization Tests

```
Do comprehensive unit + integration tests exist for this code?
│
├─ NO → Keep characterization test until full coverage exists
│
└─ YES → Does characterization test add unique value?
          │
          ├─ NO → Archive or delete characterization test
          │
          └─ YES → Keep as documentation (rename to "legacy behavior")
```

---

## Measuring Progress

> **TL;DR**: Track coverage with tools, prioritize high-risk/high-change code, update coverage debt tracker regularly.

### Coverage Tools

| Language   | Tool          | Command                          | Target   |
|------------|---------------|----------------------------------|----------|
| PHP        | PHPUnit       | `phpunit --coverage-html report` | 100%     |
| JavaScript | Vitest        | `vitest --coverage`              | 100%     |
| TypeScript | Vitest        | `vitest --coverage`              | 100%     |
| Python     | coverage.py   | `coverage run -m unittest`       | 100%     |


---

## Dealing with Flaky Tests

> **TL;DR**: Fix flakiness before refactoring. Flaky tests are worse than no tests.

### Common Causes & Fixes

| Cause                    | Symptom                        | Fix                                     |
|--------------------------|--------------------------------|-----------------------------------------|
| Non-deterministic time   | Tests fail randomly            | Inject date provider (see seams.md)    |
| Random data              | Assertions fail sporadically   | Inject random seed or use fixed data    |
| Async race conditions    | Intermittent failures          | Add explicit waits, avoid sleep()       |
| External dependencies    | Network/DB timeouts            | Use test doubles (after seam intro)     |
| Global state pollution   | Test order matters             | Reset state in teardown, isolate tests  |
| Filesystem timing        | File not found errors          | Use synchronous I/O in tests            |

### Flaky Test Decision Tree

```
Is the test flaky?
│
├─ NO → Proceed
│
└─ YES → Is the flakiness in production code or test code?
           │
           ├─ Test code → Fix test (add waits, reset state)
           │
           └─ Production code → Is it time/random/network related?
                                │
                                ├─ YES → Introduce seam (see seams.md)
                                │
                                └─ NO → Investigate root cause
                                         (may indicate real bug)
```

---

## Language-Specific Patterns

| Language   | Framework | Assertion Style       | Pattern to Match          | Notes                          |
|------------|-----------|----------------------|---------------------------|--------------------------------|
| PHP        | PHPUnit   | `assertEquals()`     | Existing base class       | Keep DB tests if they exist    |
| JavaScript | Vitest    | `expect().toBe()`    | Callbacks vs async/await  | Match existing pattern         |
| TypeScript | Vitest    | `expect().toBe()`    | Callbacks vs async/await  | Match existing pattern         |
| Python     | unittest  | `assertEqual()`      | Class hierarchy           | No decorators unless used      |

**Key**: Always match the project's existing test style, even if outdated.

---

## Quick Reference: Do's and Don'ts

### ❌ Don't

| Action                                    | Why                                      |
|-------------------------------------------|------------------------------------------|
| Change untested code                      | No safety net; high risk of breakage     |
| Test entire codebase upfront              | Wastes time; coverage should follow need |
| Introduce mocking frameworks              | Unless already used in project           |
| Refactor before characterizing            | You'll change behavior without knowing   |
| Test private/internal details             | Tests should verify public API only      |
| Replace integration with unit tests       | Both serve different purposes            |

### ✅ Do

| Action                                    | Why                                      |
|-------------------------------------------|------------------------------------------|
| Write characterization tests first        | Documents current behavior safely        |
| Test only what you're changing            | Efficient; coverage grows organically    |
| Use simplest test doubles                 | In-memory fakes, not mocks               |
| Match existing test style                 | Consistency with codebase                |
| Track coverage debt                       | Prioritize high-risk, high-change areas  |
| Archive characterization tests            | After full unit/integration coverage     |

---

## Remember

**Characterization tests are temporary scaffolding.** Their purpose is to document legacy behavior so you can refactor safely. Once you have comprehensive unit and integration tests, archive or remove characterization tests. The end goal is 100% test coverage achieved incrementally through the cycle: **characterize → extract → unit test → integrate → archive**.

Legacy code survived without tests, but it cannot safely evolve without them.
