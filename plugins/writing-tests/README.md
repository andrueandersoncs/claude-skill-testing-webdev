# Writing Tests Plugin

A Claude Code plugin providing testing guidance and commands for TypeScript web applications using Bun, React, Effect, and Playwright.

## Features

- **Skill**: Auto-activates when writing tests, adding coverage, or debugging test failures
- **Commands**: Two slash commands for reviewing and generating tests

## Commands

### `/test-review [file-path]`

Review test code for adherence to testing best practices. Checks for:
- Testing behavior vs implementation
- Proper mocking boundaries
- Generated test data usage
- Component rendering patterns
- Appropriate test type selection

### `/generate-test <file-path>`

Generate tests for specified code following best practices:
- Chooses appropriate test type (unit/integration/E2E)
- Uses Effect Schema + Arbitrary for test data
- Applies the Test Layer pattern for mocking
- Creates idempotent E2E tests with cleanup

## Skill Triggers

The skill activates automatically when you mention:
- "writing tests" or "add tests"
- "test coverage"
- "debugging test failures"
- "test infrastructure"

## Core Testing Principles

1. **Test behavior, not implementation** - Verify what the system does
2. **Mock only at system boundaries** - External APIs, not internal code
3. **Generate test data** - Use Effect Arbitrary, never static fixtures
4. **Never mock React components** - Render real component trees
5. **Prefer integration tests** - When cost is similar to unit tests

## Installation

Add to your Claude Code plugins or include in your project's `.claude-plugin/` directory.

## File Conventions

| Extension | Purpose |
|-----------|---------|
| `.test.ts` | Unit tests |
| `.integration.ts` | Integration tests |
| `.spec.ts` | E2E tests (Playwright) |
| `.test-impl.ts` | Test service implementations |

## Author

Andrue Anderson
