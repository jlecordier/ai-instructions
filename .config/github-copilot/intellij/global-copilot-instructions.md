# Project-wide Principles

- Maintainability, clarity, and testability are first priorities.
- Small functions with single responsibility; compose behaviors instead of building monoliths.
- Prefer pure functions; isolate side effects (I/O, network, time, randomness) at the edges.
- No global mutable state. Only named constants at file scope; pass data via parameters/returns.
- Use early returns; avoid deep nesting and one-liners (even for early returns).
- Eliminate magic numbers/strings; introduce intention-revealing constants.
- Follow the existing style of each file; do not reformat unrelated code.
- Prefer dependency injection through narrow interfaces; use in-memory test doubles.
- Decompose time values explicitly (e.g., 5 * 60 * 60).
- Use closed-interval comparisons: `if (!(1 <= x && x <= 10))` instead of `if (x <= 0 || x >= 11)`.
- Do not add external packages unless already used in the project or explicitly requested.

## CLI Utilities (all languages)

- If a script is CLI-only, enforce it and exit early otherwise. Do not emit HTTP headers in CLI.
- Do not use query strings for CLI parameters; parse argv only.
- Provide a parse_cli_args function (or equivalent) supporting --flag=value and --flag value, plus --help.
- Defaults must be explicit and documented in --help (including bounds like 1..10).
- When writing files, ensure parent directories exist; fail gracefully with clear messages.
- Use deterministic temp paths per run to avoid collisions across modes.

## I/O and Networking (all languages)

- Wrap I/O and network calls in focused helpers; do not leak resource handles.
- Validate inputs early (throw/return clear programmer-error messages for invalid inputs).
- Use standardized result objects/records for network transfers: { url, file/path, httpStatus, success, error, bytes }.
- On failure, clean up partial outputs.

## Output and Observability (CLI)

- Keep CLI output minimal and consistent; no transient debug logs.
- Summaries should include: duration, total bytes, throughput, success counts; per-item lines should be uniform.

## Testing Strategy

- Unit tests: maximize coverage of code that is not an external dependency; inject in-memory doubles.
- Integration tests: cover external dependencies only; keep minimal to avoid overlap.
- End-to-end tests: cover the entire flow with real-world scenarios.
- No monkey patching; use DI and in-memory doubles instead.

## Definition of Done (applies to all languages)

- File/module begins with required directives (e.g., PHP strict types) and proper namespace/module structure.
- No global mutable state; no top-level executable code.
- Main orchestrator delegates to small, focused helpers.
- All magic numbers/strings replaced by named constants.
- CLI argument parsing implemented with clear defaults; --help reflects reality.
- Errors are explicit and actionable; partial artifacts cleaned on failure.
- Output format is consistent and documented.
- Lint/build/test pass with the repo tools (eslint/prettier, composer/phpunit, vitest, etc.).

# Commands

## Build/Lint/Test

- **Build**: `npm run build` (TypeScript compilation)
- **Lint**: `npm run lint .` (JavaScript/TypeScript), `composer run lint` (PHP with php-cs-fixer)
- **Test all**: `npm test` (Vitest), `composer run test` (PHPUnit)
- **Test single**: `npm test -- path/to/test.spec.ts`, `phpunit --filter TestClass::testMethod`
- **Test unit/integration/e2e** (PHP): `composer run test-unit`, `composer run test-integration`, `composer run test-e2e`

## Code Style Guidelines

### General

- **Formatting**: Use Prettier for JS/TS, indent with 4 spaces (code) / 2 spaces (md/yaml)
- **Imports**: Group by external/internal, sort alphabetically
- **Error handling**: Use try/catch with specific error types, avoid generic catch blocks
- Follow single responsibility and composition over inheritance.
- Do not comment what is obvious by reading the code.

### TypeScript/JavaScript

- Always use semicolons.
- Prefer `const` over `let`; use template literals over concatenation.
- Strict typing: no `any`; use interfaces for objects, union types for variants.
- ESLint rules: prefer-template, no-useless-concat, prefer-const, one-var never.
- **Naming**: camelCase for variables/functions, PascalCase for classes/types, UPPER_SNAKE for constants.

### PHP

- Use type hints and return types everywhere; `declare(strict_types=1);` at file top.
- Follow existing patterns in legacy code; smallest necessary change.
- **Naming**: snake_case for variables/functions, camelCase for methods, PascalCase for classes/types, UPPER_SNAKE for constants.
- Never leave code outside a function; wrap in a `main`/`execute` function and call it.
- In PHP, never use `\function()`; just use `function()`; likewise for other imports.

### Python

- Type hints required; docstrings for functions.
- Use descriptive names; no underscore prefixes for "private" functions.
- `unittest` framework, not `pytest`.
- **Naming**: snake_case for variables/functions, PascalCase for classes/types, UPPER_SNAKE for constants.
- No top-level code; wrap execution in `main()` and guard `if __name__ == "__main__": main()`.

### SQL

Format SQL this way:

```sql
SELECT
    column1,
    column2
FROM table1
    JOIN table2
        ON table1.id = table2.table1_id
WHERE condition1 = value1
    AND condition2 = value2
ORDER BY ...
LIMIT ...
```

## Workflow

- **File editing policy**: If it is not explicit that file edits are requested, do not edit files. Instead:
  - Answer the question directly if it's informational
  - Ask for confirmation before proposing file changes
  - Only generate file edits when the user clearly requests modifications (e.g., "add", "change", "fix", "implement", "refactor")
- **Be critical and validate requests**: Never assume the user is right. If you spot a mistake, error, antipattern, or suboptimal approach in what is being asked, stop and:
  1. Point out the issue clearly
  2. Explain why it's problematic
  3. Suggest the correct or better solution
  4. Ask for confirmation whether to proceed with the original request or adopt the suggested approach
- If you think you are missing information to complete a task, ask first (unless the prompt starts with `FREE`).
- If there are two possible approaches, ask which one to choose (unless `FREE`).
- Do not take initiatives beyond what is asked; propose and wait for approval.
- If the requested approach is not ideal, explain why, suggest alternatives, and wait for the answer.
- Do not add comments that just state what changed.

## Practices

- Use best practices, clean code, clean/hexagonal architecture, design patterns, SOLID.
- Prefer functional programming, but prioritize readability; avoid `reduce` when it harms clarity.
- Prefer small functions, pure functions, and composition over inheritance.
- Use the extract until drop technique.
- Follow the single responsibility principle.
- Do not comment what is obvious just by reading the code.
- Use early returns; never one-liners.
- Always decompose time values (e.g., 5 * 60 * 60).
- Types in TS, Python, and PHP; never use `any` in TS.
- Do not add external packages unless asked or already used in the project/file.
- Isolate external dependencies; use interfaces and in-memory test doubles.
- Follow the existing coding style and conventions of each file.
- Python: never create "private" underscore functions.
- PHP: never use leading backslashes for functions/imports.
- Never leave code outside a function; scripts must have a `main`/`execute` entry point.
- Use maths-like closed intervals for comparisons, ex: `if (!(1 <= x && x <= 10))` instead of `if (x <= 0 || x >= 11)`.

## Legacy

- Make the smallest possible changes; do not rename or modernize beyond the request.
- Do not change indentation of existing code; you may correctly indent only your new code.
- For added code, follow the existing style and conventions.

## Tools

- Use Prettier for formatting, not ESLint's `--fix`.
- Code indentation: 4 spaces; markdown/yaml: 2 spaces.
- JS/TS: always use semicolons
- Prefer string interpolation over concatenation.
- Tests: Vitest (never Jest); Python: `unittest` (never `pytest`).
- Always check the most recent documentation before suggesting configurations.
- When using external APIs/packages, always check the most recent documentation.

Format SQL this way:

```sql
SELECT
    column1,
    column2
FROM table1
    JOIN table2
        ON table1.id = table2.table1_id
WHERE condition1 = value1
    AND condition2 = value2
ORDER BY ...
LIMIT ...
```

## Tests

- Never use monkey patching.
- Unit tests: maximize coverage of code that is not an external dependency; inject in-memory doubles.
- Integration tests: cover external dependencies only; keep minimal to avoid overlap.
- End-to-end tests: cover the entire application flow, including all external dependencies, and be as close to real user scenarios as possible.

## Terminal

- When suggesting terminal commands, detail every argument and use long options.

## System administration

- For destructive actions, present a clear warning, steps to verify current state beforehand, and steps to recover from mistakes after the action.