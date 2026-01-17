# Generate Test Command

Generate tests for the specified code following TypeScript web application testing best practices.

## Input

The user will provide one of:
- A file path to generate tests for
- Code in the current context
- A description of functionality to test

## Test Generation Rules

### 1. Choose Appropriate Test Type
| Code Type | Test Type | File Extension |
|-----------|-----------|----------------|
| Pure function, schema validation | Unit | `.test.ts` |
| Effect computation | Unit | `.test.ts` |
| Component with side effects | Integration | `.integration.ts` |
| Multiple services interacting | Integration | `.integration.ts` |
| Full user journey | E2E | `.spec.ts` |

### 2. Generate Test Data with Effect
```typescript
import { Schema, Arbitrary } from "effect"
import * as fc from "fast-check"

// Create arbitrary from schema
const dataArb = Arbitrary.make(YourSchema)
const sample = fc.sample(dataArb, 1)[0]
```

### 3. Mock Only at Boundaries
- External APIs: Use test Layer implementations
- HTTP: Use `msw` or route interception
- Never mock internal modules or React components

### 4. Use Test Layer Pattern for Effect Services
```typescript
// Test implementation - no real API calls
export const ServiceTest = Layer.succeed(Service, {
  method: (args) => Effect.succeed(testResult)
})

// Compose test environment
const TestEnv = Layer.mergeAll(ServiceTest, OtherServiceTest)
```

### 5. E2E Tests Must Be Idempotent
- Generate unique data per test
- Track created resources
- Clean up in `afterEach`

## Output

Generate complete, runnable test file(s) with:
- Proper imports
- Generated test data using Effect Arbitrary
- Appropriate test structure (describe/test blocks)
- Meaningful assertions
- Cleanup if needed

## Reference

See [SKILL.md](../SKILL.md) and [TESTING_BEST_PRACTICES.md](../TESTING_BEST_PRACTICES.md) for full testing guidelines.
