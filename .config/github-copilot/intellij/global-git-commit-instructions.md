# Git Commit Message Guidelines

When generating git commit messages, follow these conventions:

## Structure

```
[type: ]main category: sub category: filename: brief description
```

## Rules

- **Base message on git history**: Analyze previous commit messages in the repository to match the existing style and conventions.
- **Categories from folder structure**: Derive logical categories from the folder path, using colons as separators.
- **Skip low-value folders**: Omit common prefixes like `src/`, `lib/`, `dist/` that don't add semantic meaning.
- **Filename without extension**: For programming languages and HTML files, omit the extension. For config files (package.json, tsconfig.json, .eslintrc, etc.), keep the extension.
- **Type prefix**: Add a type prefix followed by a colon and space, EXCEPT for features (new functionality).

## Category Mapping Examples

- `src/api/users/controller.ts` → `api: users: controller`
- `src/components/Button/Button.tsx` → `components: Button`
- `lib/services/auth/validator.js` → `services: auth: validator`
- `src/utils/date.ts` → `utils: date`
- `package.json` → `package.json` (config file, no categories)

## Types

- `fix:` - Bug fixes
- `clean code:` - Refactoring, code improvements without changing behavior
- `log:` - Logging changes
- `security:` - Security-related changes
- `docs:` - Documentation changes
- _(no prefix)_ - Features (new functionality)

## Examples

- `fix: api: users: controller: handle null email addresses`
- `clean code: services: auth: validator: extract validation logic`
- `log: controllers: order: add debug logging for failed transactions`
- `security: middleware: auth: sanitize user input`
- `docs: README: add installation instructions`
- `api: products: controller: add filtering by category` (feature, no prefix)
- `fix: package.json: update vulnerable dependencies`
- `components: Button: add loading state` (feature, no prefix)

## Additional Guidelines

- Keep the description brief and imperative mood (e.g., "add", "fix", "update", not "added", "fixed", "updated")
- Focus on **what** changed and **why**, not **how**
- If multiple files are changed for the same logical change, you may group them or create separate commits
- Be critical: if the changes don't fit a clear category or seem to mix concerns, suggest splitting into multiple commits