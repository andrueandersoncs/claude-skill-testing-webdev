# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Claude Skill repository that provides testing guidance for TypeScript web applications using Bun, React, Effect, and Playwright.

## Repository Structure

- `SKILL.md` - Main skill definition (frontmatter with name/description, core rules, quick reference)
- `TESTING_BEST_PRACTICES.md` - Extended reference documentation with detailed patterns and examples
- `.claude/skills/` - Contains skill authoring reference submodule

## Core Testing Principles

When writing tests for TypeScript web apps, follow these rules:

1. **Test behavior, not implementation** - Verify what the system does, not how
2. **Mock only at system boundaries** - Third-party APIs, not internal code
3. **Generate test data** - Use Effect `Arbitrary.make(Schema)` with `fast-check`, never static fixtures
4. **Never mock React components** - Render real component trees
5. **Prefer integration tests** - When cost is similar to unit tests

## Test File Conventions

| Extension | Purpose |
|-----------|---------|
| `.test.ts` | Unit tests |
| `.integration.ts` | Integration tests |
| `.spec.ts` | E2E tests (Playwright) |
| `.test-impl.ts` | Test service implementations |

## Key Patterns

**Data generation with Effect:**
```typescript
const userArb = Arbitrary.make(UserSchema)
const sampleUser = fc.sample(userArb, 1)[0]
```

**Effect Layer pattern for DI:**
- Production: `ServiceLive` - calls real external APIs
- Test: `ServiceTest` - returns test data, no API calls
- Compose with `Layer.mergeAll()` - real DB + mocked externals

**E2E test isolation:**
- Generate unique data per test
- Track created resources in test context
- Clean up in `afterEach` (dependency order)
