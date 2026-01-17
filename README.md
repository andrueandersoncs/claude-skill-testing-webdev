# Claude Skill: Writing Tests for TypeScript Web Applications

A Claude Skill that provides comprehensive guidance on testing TypeScript web applications using modern tooling including Bun, React, Effect, and Playwright.

## Overview

This skill helps Claude write high-quality tests by establishing clear testing principles, boundaries, and patterns. It emphasizes behavior-driven testing, proper mocking boundaries, and generated test data over static fixtures.

## Core Testing Principles

1. **Test behavior, not implementation** - Focus on what code does, not how it does it
2. **Mock only at system boundaries** - HTTP, filesystem, external services
3. **Generate test data** - Use Effect Arbitrary instead of static fixtures
4. **Never mock React components** - Test with real component trees
5. **Prefer integration tests** - When cost is similar to unit tests

## Testing Stack

| Tool | Purpose |
|------|---------|
| **Bun** | Runtime and test runner |
| **React** | Frontend framework |
| **Effect** | Schema validation and test data generation |
| **Playwright** | End-to-end testing |
| **TypeScript** | Type-safe development |

## Project Structure

```
├── SKILL.md                    # Main skill definition
├── TESTING_BEST_PRACTICES.md   # Extended reference documentation
└── .claude/skills/
    └── claude-skill-authoring-skill/   # Skill authoring reference (submodule)
```

## Testing Layers

### Unit Tests (`.test.ts`)
- Pure functions and computations
- Schema validation logic
- Isolated Effect programs

### Integration Tests (`.integration.ts`)
- React components with real children
- Effect services with test layers
- API routes with mocked boundaries

### E2E Tests (`.spec.ts`)
- Complete user flows with Playwright
- Idempotent tests with proper cleanup
- Real browser interactions

## Usage

This skill activates when Claude is asked to write tests for TypeScript web applications. It provides guidance on:

- Choosing appropriate test types
- Structuring test files
- Generating test data with Effect Schema
- Mocking external dependencies
- Writing maintainable assertions

## Plugin Marketplace

This repository includes a Claude Code plugin marketplace. Install the testing plugin to get commands for reviewing and generating tests.

### Installation

```bash
# Add the marketplace
/plugin marketplace add owner/claude-skill-testing-webdev

# Install the plugin
/plugin install writing-tests@typescript-testing-tools
```

### Available Commands

| Command | Description |
|---------|-------------|
| `/test-review` | Review test code for best practices compliance |
| `/generate-test` | Generate tests for specified code |

### Marketplace Structure

```
.claude-plugin/
└── marketplace.json          # Marketplace manifest

plugins/writing-tests/
├── .claude-plugin/
│   └── plugin.json           # Plugin manifest
├── commands/
│   ├── test-review.md        # /test-review command
│   └── generate-test.md      # /generate-test command
├── SKILL.md                  # Testing skill definition
└── TESTING_BEST_PRACTICES.md # Extended documentation
```

## Related

- [SKILL.md](./SKILL.md) - Full skill definition
- [TESTING_BEST_PRACTICES.md](./TESTING_BEST_PRACTICES.md) - Detailed testing patterns and examples

## License

MIT
