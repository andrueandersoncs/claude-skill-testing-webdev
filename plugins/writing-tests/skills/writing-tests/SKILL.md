---
name: writing-tests
description: Guides testing for TypeScript web apps using Bun, React, Effect, and Playwright. Use when writing tests, adding test coverage, debugging test failures, or setting up test infrastructure.
---

# Testing TypeScript Web Applications

Stack: Bun (runtime/test runner), React, Effect (Schema/Arbitrary), Playwright (E2E)

## Core Rules

1. **Test behavior, not implementation** - Verify what the system does, not how
2. **Mock only at system boundaries** - Third-party APIs, not internal code
3. **Generate test data** - Use Effect Arbitrary, never static fixtures
4. **Never mock React components** - Render real component trees
5. **Prefer integration tests** - When cost is similar to unit tests

See [Core Philosophy](references/TESTING_BEST_PRACTICES.md#core-philosophy) for the full testing manifesto.

## What to Mock by Test Type

| Layer | Mock | Don't Mock |
|-------|------|------------|
| **Unit** | External APIs, DB, time, randomness | Components, pure functions, utilities |
| **Integration** | External APIs only | Internal services, DB, components |
| **E2E** | External APIs (network intercept) | Everything else |

See [Testing Boundaries](references/TESTING_BEST_PRACTICES.md#testing-boundaries) and [What to Mock](references/TESTING_BEST_PRACTICES.md#what-to-mock) for details.

## Data Generation with Effect

```typescript
import { Schema, Arbitrary } from "effect"
import * as fc from "fast-check"

// Schema defines shape, Arbitrary comes free
const User = Schema.Struct({
  id: Schema.String.pipe(Schema.minLength(1)),
  email: Schema.String.pipe(Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)),
  age: Schema.Number.pipe(Schema.int(), Schema.between(0, 150))
})

// Generate test data
const userArb = Arbitrary.make(User)
const sampleUser = fc.sample(userArb, 1)[0]

// Property-based testing
test("serialization roundtrips", () => {
  fc.assert(fc.property(userArb, (user) => {
    const encoded = Schema.encodeSync(User)(user)
    const decoded = Schema.decodeSync(User)(decoded)
    expect(decoded).toEqual(user)
  }))
})
```

See [Data Generation with Effect Schema & Arbitrary](references/TESTING_BEST_PRACTICES.md#data-generation-with-effect-schema--arbitrary) for custom arbitraries, seeding, and advanced patterns.

## Test Layer Pattern (Effect DI)

```typescript
// Production - calls real Stripe
export const PaymentServiceLive = Layer.succeed(PaymentService, {
  charge: (amount, customerId) => Effect.tryPromise({
    try: () => stripe.charges.create({ amount, customer: customerId }),
    catch: (e) => new PaymentError({ cause: e })
  })
})

// Test - no API calls
export const PaymentServiceTest = Layer.succeed(PaymentService, {
  charge: (amount, customerId) => Effect.succeed({
    id: `ch_test_${Date.now()}`, amount, customerId, status: "succeeded"
  })
})

// Compose: real DB + mocked externals
const TestEnv = Layer.mergeAll(PaymentServiceTest, DatabaseServiceLive, EmailServiceTest)
```

See [Integration Testing with Effect Dependency Injection](references/TESTING_BEST_PRACTICES.md#integration-testing-with-effect-dependency-injection) for the full Layer pattern and stateful test services.

## React Component Testing

```typescript
// WRONG - mocking hides integration bugs
jest.mock("./UserAvatar", () => () => <div>Mocked</div>)

// CORRECT - render real trees
test("displays user", async () => {
  const user = fc.sample(Arbitrary.make(User), 1)[0]
  render(<UserProfile user={user} />)
  expect(screen.getByText(user.name)).toBeInTheDocument()
})
```

Test hooks via components that use them, not `renderHook` in isolation.

See [React Component Testing](references/TESTING_BEST_PRACTICES.md#react-component-testing) for providing test context, testing hooks, and form testing with generated data.

## E2E Testing with Playwright

Each test must be idempotent:
1. Generate unique data per test
2. Track all created resources
3. Clean up in `afterEach` (dependency order)
4. Never rely on pre-existing DB state

```typescript
test.afterEach(async () => {
  await cleanupTestContext(ctx) // Deletes orders, then products, then users
})
```

See [End-to-End Testing with Playwright](references/TESTING_BEST_PRACTICES.md#end-to-end-testing-with-playwright) for:
- [Project Setup](references/TESTING_BEST_PRACTICES.md#project-setup) - Playwright config
- [Page Object Pattern](references/TESTING_BEST_PRACTICES.md#page-object-pattern-with-generated-data) - Reusable page classes
- [Test Isolation and Cleanup](references/TESTING_BEST_PRACTICES.md#test-isolation-and-cleanup) - Fixtures and transaction rollback
- [API Mocking for External Services](references/TESTING_BEST_PRACTICES.md#api-mocking-for-external-services-only) - Route interception

## Anti-Patterns

- `jest.mock("./internal-module")` - Mock boundaries, not internals
- `import fixture from "./__fixtures__/user.json"` - Generate data instead
- `jest.mock("./Modal", () => ...)` - Render real components
- `expect(render(<Complex />)).toMatchSnapshot()` - Snapshots become noise
- 4+ mocks in one test - Write integration test instead

See [Anti-Patterns to Avoid](references/TESTING_BEST_PRACTICES.md#anti-patterns-to-avoid) for detailed explanations.

## Quick Reference

| Scenario | Type |
|----------|------|
| Pure function, schema validation | Unit |
| Effect computation | Unit |
| Component with side effects | Integration |
| Multiple services interacting | Integration |
| Full user journey, cross-browser | E2E |

See [Quick Reference](references/TESTING_BEST_PRACTICES.md#quick-reference) for checklists:
- [Data Generation Checklist](references/TESTING_BEST_PRACTICES.md#data-generation-checklist)
- [Test Idempotency Checklist](references/TESTING_BEST_PRACTICES.md#test-idempotency-checklist)

## File Organization

```
src/
├── features/checkout/
│   ├── checkout.ts
│   ├── checkout.test.ts          # Unit tests
│   └── checkout.integration.ts   # Integration tests
├── services/payment/
│   ├── payment.ts
│   ├── payment.test.ts
│   └── payment.test-impl.ts      # Test implementation
└── components/checkout-form/
    ├── checkout-form.tsx
    └── checkout-form.test.tsx

e2e/
├── checkout.spec.ts
├── pages/checkout.page.ts
└── utils/generators.ts
```

See [Test File Organization](references/TESTING_BEST_PRACTICES.md#test-file-organization) for the full structure.
