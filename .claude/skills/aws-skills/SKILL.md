```markdown
# aws-skills Development Patterns

> Auto-generated skill from repository analysis

## Overview
This skill introduces the core development patterns and conventions used in the `aws-skills` TypeScript repository. You'll learn how to structure files, write imports/exports, follow commit message guidelines, and organize tests. This guide ensures consistency and maintainability across contributions.

## Coding Conventions

### File Naming
- Use **PascalCase** for file names.
  - Example: `AwsResource.ts`, `UserManager.test.ts`

### Import Style
- Use **relative imports** for referencing modules.
  - Example:
    ```typescript
    import { AwsResource } from './AwsResource';
    ```

### Export Style
- Use **named exports** for all modules.
  - Example:
    ```typescript
    // AwsResource.ts
    export function createResource() { ... }
    export const RESOURCE_TYPE = 'S3';
    ```

### Commit Messages
- Follow **conventional commit** format.
- Use prefixes like `feat` for features and `docs` for documentation.
  - Example:
    ```
    feat: add support for new AWS Lambda triggers
    docs: update README with usage examples
    ```

## Workflows

### Adding a New Feature
**Trigger:** When implementing new functionality.
**Command:** `/add-feature`

1. Create a new file using PascalCase (e.g., `NewFeature.ts`).
2. Implement the feature using named exports.
3. Write or update relevant tests (`NewFeature.test.ts`).
4. Commit changes using the `feat:` prefix.
    ```
    feat: implement new feature for resource tagging
    ```

### Updating Documentation
**Trigger:** When modifying or adding documentation.
**Command:** `/update-docs`

1. Edit or add documentation files as needed.
2. Ensure code examples follow import/export conventions.
3. Commit changes using the `docs:` prefix.
    ```
    docs: add section on resource cleanup
    ```

### Running Tests
**Trigger:** Before pushing changes or verifying code.
**Command:** `/run-tests`

1. Identify test files matching `*.test.*` pattern.
2. Run tests using your preferred TypeScript-compatible test runner.
3. Ensure all tests pass before merging.

## Testing Patterns

- Test files use the `*.test.*` naming convention (e.g., `AwsResource.test.ts`).
- Testing framework is not specified; use a TypeScript-compatible runner (such as Jest or Mocha).
- Place tests alongside or near the modules they cover.
- Example test file:
  ```typescript
  // AwsResource.test.ts
  import { createResource } from './AwsResource';

  describe('createResource', () => {
    it('should create an S3 resource', () => {
      expect(createResource('S3')).toBeDefined();
    });
  });
  ```

## Commands
| Command        | Purpose                                    |
|----------------|--------------------------------------------|
| /add-feature   | Scaffold and commit a new feature module   |
| /update-docs   | Update or add documentation                |
| /run-tests     | Run all tests in the repository            |
```
