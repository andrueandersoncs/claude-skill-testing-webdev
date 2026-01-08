# Testing Best Practices for TypeScript Web Applications

## Stack Context
- **Runtime/Test Runner:** Bun
- **Frontend:** React
- **Core Library:** Effect (with Schema and Arbitrary)
- **E2E Testing:** Playwright
- **Language:** TypeScript

---

## Core Philosophy

### The Testing Manifesto

1. **Test real behavior, not implementation details.** Tests should verify what the system does, not how it does it internally.

2. **Minimize mocking surface area.** Mock only at system boundaries where external services live. Never mock internal application code.

3. **Generate test data, don't hardcode it.** Static fixtures become stale and miss edge cases. Property-based testing with generated data finds bugs humans wouldn't think to test for.

4. **Prefer higher-level tests when cost is similar.** An integration test that runs in 50ms is better than a unit test that requires 5 mocks to isolate a function.

5. **Tests are documentation.** A reader should understand the system's expected behavior by reading tests.

---

## Testing Boundaries

### What to Test at Each Layer

```
┌─────────────────────────────────────────────────────────────────┐
│                     E2E Tests (Playwright)                      │
│  • Complete user flows in real browser                          │
│  • Critical paths: auth, checkout, core features                │
│  • Visual regression, accessibility                             │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    Integration Tests (Bun)                       │
│  • React components with real children, real hooks              │
│  • Effect services with real dependencies (test implementations)│
│  • API routes with real request/response cycles                 │
│  • Database operations with real test database                  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                       Unit Tests (Bun)                          │
│  • Pure functions (validators, transformers, calculators)       │
│  • Effect computations in isolation                             │
│  • Schema encode/decode logic                                   │
│  • Complex business logic modules                               │
└─────────────────────────────────────────────────────────────────┘
```

### The Boundary Rule

**Unit Tests - Mock freely at service boundaries:**
- Third-party HTTP APIs (Stripe, SendGrid, etc.)
- Internal application services (via Effect Layer injection)
- Database services
- Time/randomness when determinism is required
- File system operations

**Integration Tests - Mock only external services:**
- Third-party HTTP APIs ✅ Mock
- Internal application services ❌ Use real implementations
- Database services ❌ Use real test database
- Time/randomness ✅ Mock when determinism required

**NEVER mock (at any level):**
- React components
- Utility functions
- Pure business logic modules

---

## Data Generation with Effect Schema & Arbitrary

### Why Generated Data Over Fixtures

Static fixtures:
- Miss edge cases (empty strings, unicode, boundary numbers)
- Become stale as schemas evolve
- Encode assumptions about what "typical" data looks like
- Require manual maintenance

Generated data:
- Explores the entire valid input space
- Automatically stays in sync with your schemas
- Finds edge cases you'd never think to write
- Documents the actual shape of valid data

### Defining Schemas with Arbitrary Support

```typescript
import { Schema } from "effect"

// Define your domain schemas - Arbitrary generation comes free
const Email = Schema.String.pipe(
  Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/),
  Schema.brand("Email")
)

const UserId = Schema.String.pipe(
  Schema.minLength(1),
  Schema.maxLength(36),
  Schema.brand("UserId")
)

const User = Schema.Struct({
  id: UserId,
  email: Email,
  name: Schema.String.pipe(Schema.minLength(1), Schema.maxLength(100)),
  age: Schema.Number.pipe(Schema.int(), Schema.between(0, 150)),
  role: Schema.Literal("admin", "user", "guest"),
  createdAt: Schema.Date,
  metadata: Schema.Record({ key: Schema.String, value: Schema.Unknown })
})

type User = Schema.Schema.Type<typeof User>
```

### Generating Test Data

```typescript
import { Arbitrary } from "effect"
import * as fc from "fast-check"

// Generate a single arbitrary value
const userArbitrary = Arbitrary.make(User)

// Use in property-based tests
test("User serialization roundtrips", () => {
  fc.assert(
    fc.property(userArbitrary, (user) => {
      const encoded = Schema.encodeSync(User)(user)
      const decoded = Schema.decodeSync(User)(encoded)
      expect(decoded).toEqual(user)
    })
  )
})

// Generate sample data for other tests
const sampleUser = fc.sample(userArbitrary, 1)[0]
const manyUsers = fc.sample(userArbitrary, 100)
```

### Custom Arbitraries for Complex Constraints

```typescript
import { Arbitrary, Schema } from "effect"
import * as fc from "fast-check"

// When you need more control than Schema provides
const EmailArbitrary = fc.emailAddress().map(email => email as Email)

// Combine schemas with custom arbitraries
const UserWithRealisticEmail = Arbitrary.make(User).chain(user =>
  EmailArbitrary.map(email => ({ ...user, email }))
)

// For relational data, generate consistent sets
const generateConsistentData = () => {
  const orgId = fc.sample(Arbitrary.make(OrgId), 1)[0]
  const users = fc.sample(
    Arbitrary.make(User).map(u => ({ ...u, orgId })),
    10
  )
  return { orgId, users }
}
```

### Seeding for Reproducibility

```typescript
// Use seed for reproducible "random" data
const seed = 42
const userArb = Arbitrary.make(User)

test("deterministic test data", () => {
  fc.assert(
    fc.property(userArb, (user) => {
      // test logic
    }),
    { seed }
  )
})

// For snapshot-style tests that need stable data
const stableUser = fc.sample(userArb, { seed, numRuns: 1 })[0]
```

---

## Unit Testing with Bun

### Testing Pure Functions

```typescript
import { describe, test, expect } from "bun:test"
import { Schema } from "effect"
import * as fc from "fast-check"
import { Arbitrary } from "effect"

import { calculateDiscount, DiscountInput } from "./pricing"

describe("calculateDiscount", () => {
  const inputArb = Arbitrary.make(DiscountInput)

  test("never returns negative discount", () => {
    fc.assert(
      fc.property(inputArb, (input) => {
        const result = calculateDiscount(input)
        expect(result).toBeGreaterThanOrEqual(0)
      })
    )
  })

  test("discount never exceeds original price", () => {
    fc.assert(
      fc.property(inputArb, (input) => {
        const result = calculateDiscount(input)
        expect(result).toBeLessThanOrEqual(input.originalPrice)
      })
    )
  })

  test("bulk orders get at least 10% off", () => {
    const bulkOrderArb = inputArb.filter(i => i.quantity >= 100)
    fc.assert(
      fc.property(bulkOrderArb, (input) => {
        const result = calculateDiscount(input)
        expect(result).toBeGreaterThanOrEqual(input.originalPrice * 0.1)
      })
    )
  })
})
```

### Testing Effect Computations

```typescript
import { describe, test, expect } from "bun:test"
import { Effect, Exit } from "effect"
import * as fc from "fast-check"
import { Arbitrary } from "effect"

import { validateOrder, Order, ValidationError } from "./orders"

describe("validateOrder", () => {
  const orderArb = Arbitrary.make(Order)

  test("valid orders pass validation", async () => {
    await fc.assert(
      fc.asyncProperty(orderArb, async (order) => {
        const result = await Effect.runPromiseExit(validateOrder(order))
        expect(Exit.isSuccess(result)).toBe(true)
      })
    )
  })

  test("empty items fail validation", async () => {
    const emptyOrderArb = orderArb.map(o => ({ ...o, items: [] }))
    
    await fc.assert(
      fc.asyncProperty(emptyOrderArb, async (order) => {
        const result = await Effect.runPromiseExit(validateOrder(order))
        expect(Exit.isFailure(result)).toBe(true)
        if (Exit.isFailure(result)) {
          expect(result.cause).toContainEqual(
            expect.objectContaining({ _tag: "EmptyOrder" })
          )
        }
      })
    )
  })
})
```

### Testing Schema Validation

```typescript
import { describe, test, expect } from "bun:test"
import { Schema, Either } from "effect"
import * as fc from "fast-check"
import { Arbitrary } from "effect"

import { ApiRequest } from "./api-schemas"

describe("ApiRequest schema", () => {
  test("decodes all valid generated data", () => {
    const validArb = Arbitrary.make(ApiRequest)
    fc.assert(
      fc.property(validArb, (data) => {
        const result = Schema.decodeEither(ApiRequest)(data)
        expect(Either.isRight(result)).toBe(true)
      })
    )
  })

  test("rejects invalid input shapes", () => {
    fc.assert(
      fc.property(fc.object(), (randomObj) => {
        const result = Schema.decodeEither(ApiRequest)(randomObj)
        // Most random objects should fail
        // This surfaces any overly permissive schemas
      })
    )
  })
})
```

---

## Integration Testing with Effect Dependency Injection

### The Layer Pattern for Test Dependencies

```typescript
// src/services/payment.ts
import { Effect, Context, Layer } from "effect"

export class PaymentService extends Context.Tag("PaymentService")<
  PaymentService,
  {
    charge: (amount: number, customerId: string) => Effect.Effect<ChargeResult, PaymentError>
    refund: (chargeId: string) => Effect.Effect<RefundResult, PaymentError>
  }
>() {}

// Production implementation - calls real Stripe
export const PaymentServiceLive = Layer.succeed(
  PaymentService,
  {
    charge: (amount, customerId) =>
      Effect.tryPromise({
        try: () => stripe.charges.create({ amount, customer: customerId }),
        catch: (e) => new PaymentError({ cause: e })
      }),
    refund: (chargeId) =>
      Effect.tryPromise({
        try: () => stripe.refunds.create({ charge: chargeId }),
        catch: (e) => new PaymentError({ cause: e })
      })
  }
)

// Test implementation - no real API calls
export const PaymentServiceTest = Layer.succeed(
  PaymentService,
  {
    charge: (amount, customerId) =>
      Effect.succeed({
        id: `ch_test_${Date.now()}`,
        amount,
        customerId,
        status: "succeeded" as const
      }),
    refund: (chargeId) =>
      Effect.succeed({
        id: `re_test_${Date.now()}`,
        chargeId,
        status: "succeeded" as const
      })
  }
)
```

### Running Integration Tests with Test Layers

```typescript
import { describe, test, expect, beforeEach, afterEach } from "bun:test"
import { Effect, Layer } from "effect"
import * as fc from "fast-check"
import { Arbitrary } from "effect"

import { processOrder, Order } from "./order-processor"
import { PaymentServiceTest } from "./services/payment"
import { DatabaseServiceLive } from "./services/database" // Use real DB!
import { EmailServiceTest } from "./services/email"

// Compose test environment - real DB, mocked external services
const TestEnv = Layer.mergeAll(
  PaymentServiceTest,
  DatabaseServiceLive, // Real database operations
  EmailServiceTest
)

describe("Order Processing Integration", () => {
  const orderArb = Arbitrary.make(Order)
  const createdOrderIds: string[] = []

  beforeEach(() => {
    createdOrderIds.length = 0
  })

  afterEach(async () => {
    // Cleanup all orders created during test
    for (const id of createdOrderIds) {
      await Effect.runPromise(
        deleteOrder(id).pipe(
          Effect.provide(TestEnv),
          Effect.catchAll(() => Effect.void) // Ignore cleanup errors
        )
      )
    }
  })

  test("processes valid orders end-to-end", async () => {
    await fc.assert(
      fc.asyncProperty(orderArb, async (order) => {
        const result = await Effect.runPromise(
          processOrder(order).pipe(Effect.provide(TestEnv))
        )

        createdOrderIds.push(result.id) // Track for cleanup

        expect(result.status).toBe("completed")

        // Verify real database state
        const savedOrder = await Effect.runPromise(
          getOrderById(result.id).pipe(Effect.provide(TestEnv))
        )
        expect(savedOrder).toMatchObject({
          id: result.id,
          status: "completed"
        })
      }),
      { numRuns: 20 } // Fewer runs for integration tests
    )
  })
})
```

### Stateful Test Services

```typescript
// For tests that need to inspect mock behavior
import { Effect, Context, Layer, Ref } from "effect"

interface PaymentCall {
  amount: number
  customerId: string
  timestamp: Date
}

export class TestPaymentService extends Context.Tag("TestPaymentService")<
  TestPaymentService,
  {
    charge: (amount: number, customerId: string) => Effect.Effect<ChargeResult, PaymentError>
    getCalls: () => Effect.Effect<PaymentCall[]>
    reset: () => Effect.Effect<void>
  }
>() {}

export const makeTestPaymentService = Effect.gen(function* () {
  const calls = yield* Ref.make<PaymentCall[]>([])

  return {
    charge: (amount: number, customerId: string) =>
      Effect.gen(function* () {
        yield* Ref.update(calls, (c) => [
          ...c,
          { amount, customerId, timestamp: new Date() }
        ])
        return {
          id: `ch_test_${Date.now()}`,
          amount,
          customerId,
          status: "succeeded" as const
        }
      }),
    getCalls: () => Ref.get(calls),
    reset: () => Ref.set(calls, [])
  }
})

export const TestPaymentServiceLive = Layer.effect(
  TestPaymentService,
  makeTestPaymentService
)
```

---

## React Component Testing

### Core Principle: Zero Component Mocks

**Never do this:**
```typescript
// ❌ WRONG - mocking components hides integration bugs
jest.mock("./UserAvatar", () => ({ userId }) => <div>Mocked Avatar</div>)
jest.mock("./Tooltip", () => ({ children }) => children)
```

**Always do this:**
```typescript
// ✅ CORRECT - render real component trees
import { render, screen } from "@testing-library/react"
import { UserProfile } from "./UserProfile"

test("displays user information", async () => {
  const user = fc.sample(Arbitrary.make(User), 1)[0]
  
  render(<UserProfile user={user} />)
  
  // Real UserAvatar, real Tooltip, real everything
  expect(screen.getByText(user.name)).toBeInTheDocument()
  expect(screen.getByRole("img")).toHaveAttribute("src", expect.stringContaining(user.id))
})
```

### Providing Test Context

```typescript
import { render, screen } from "@testing-library/react"
import { EffectProvider } from "./providers/effect-provider"
import { TestEnv } from "./test-utils"

// Wrapper that provides test Effect layers
function renderWithProviders(ui: React.ReactElement) {
  return render(
    <EffectProvider layer={TestEnv}>
      {ui}
    </EffectProvider>
  )
}

test("checkout flow with test payment service", async () => {
  const cart = fc.sample(Arbitrary.make(Cart), 1)[0]
  
  renderWithProviders(<CheckoutFlow cart={cart} />)
  
  await userEvent.click(screen.getByRole("button", { name: /pay/i }))
  
  // Component uses real hooks, real children, but Effect layer
  // swaps PaymentService for test implementation
  expect(await screen.findByText(/payment successful/i)).toBeInTheDocument()
})
```

### Testing Hooks with Real Components

```typescript
// ❌ WRONG - testing hooks in isolation with renderHook
// misses integration with real component lifecycle

// ✅ CORRECT - test hooks via components that use them
function TestComponent({ userId }: { userId: string }) {
  const { data, error, loading } = useUser(userId)
  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>
  return <div>User: {data.name}</div>
}

test("useUser hook fetches and displays user", async () => {
  const userId = fc.sample(Arbitrary.make(UserId), 1)[0]
  
  renderWithProviders(<TestComponent userId={userId} />)
  
  expect(screen.getByText(/loading/i)).toBeInTheDocument()
  expect(await screen.findByText(/User:/)).toBeInTheDocument()
})
```

### Form Testing with Generated Data

```typescript
import { render, screen } from "@testing-library/react"
import userEvent from "@testing-library/user-event"
import * as fc from "fast-check"
import { Arbitrary } from "effect"

import { RegistrationForm } from "./RegistrationForm"
import { RegistrationInput } from "./schemas"

describe("RegistrationForm", () => {
  const inputArb = Arbitrary.make(RegistrationInput)

  test("submits valid data", async () => {
    const onSubmit = vi.fn()
    const input = fc.sample(inputArb, 1)[0]

    render(<RegistrationForm onSubmit={onSubmit} />)

    await userEvent.type(screen.getByLabelText(/email/i), input.email)
    await userEvent.type(screen.getByLabelText(/password/i), input.password)
    await userEvent.type(screen.getByLabelText(/name/i), input.name)
    await userEvent.click(screen.getByRole("button", { name: /submit/i }))

    expect(onSubmit).toHaveBeenCalledWith(
      expect.objectContaining({
        email: input.email,
        name: input.name
      })
    )
  })

  test("validates all generated inputs correctly", async () => {
    await fc.assert(
      fc.asyncProperty(inputArb, async (input) => {
        const onSubmit = vi.fn()
        const { unmount } = render(<RegistrationForm onSubmit={onSubmit} />)

        await userEvent.type(screen.getByLabelText(/email/i), input.email)
        await userEvent.type(screen.getByLabelText(/password/i), input.password)
        await userEvent.type(screen.getByLabelText(/name/i), input.name)
        await userEvent.click(screen.getByRole("button", { name: /submit/i }))

        // Valid generated data should always submit successfully
        expect(onSubmit).toHaveBeenCalled()

        unmount() // Cleanup for next iteration
      }),
      { numRuns: 20 }
    )
  })
})
```

---

## End-to-End Testing with Playwright

### Philosophy: Test Real User Journeys

E2E tests should mirror actual user behavior in a real browser. No shortcuts, no mocks, no fake renders.

### Project Setup

```typescript
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test"

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: "html",
  use: {
    baseURL: "http://localhost:3000",
    trace: "on-first-retry",
    screenshot: "only-on-failure"
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "firefox", use: { ...devices["Desktop Firefox"] } },
    { name: "webkit", use: { ...devices["Desktop Safari"] } },
    { name: "mobile", use: { ...devices["iPhone 13"] } }
  ],
  webServer: {
    command: "bun run dev",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI
  }
})
```

### Data Generation in E2E Tests

```typescript
// e2e/utils/generators.ts
import { Schema } from "effect"
import { Arbitrary } from "effect"
import * as fc from "fast-check"

import { User, Product, Order } from "../../src/schemas"

export const generators = {
  user: () => fc.sample(Arbitrary.make(User), 1)[0],
  product: () => fc.sample(Arbitrary.make(Product), 1)[0],
  order: () => fc.sample(Arbitrary.make(Order), 1)[0],
  users: (count: number) => fc.sample(Arbitrary.make(User), count),
  products: (count: number) => fc.sample(Arbitrary.make(Product), count)
}

// For seeded, reproducible E2E data
export const seededGenerators = (seed: number) => ({
  user: () => fc.sample(Arbitrary.make(User), { seed, numRuns: 1 })[0],
  users: (count: number) =>
    fc.sample(Arbitrary.make(User), { seed, numRuns: count })
})
```

### Page Object Pattern with Generated Data

```typescript
// e2e/pages/checkout.page.ts
import { Page, Locator, expect } from "@playwright/test"
import { generators } from "../utils/generators"
import { Address, PaymentDetails } from "../../src/schemas"

export class CheckoutPage {
  readonly page: Page
  readonly addressForm: Locator
  readonly paymentForm: Locator
  readonly submitButton: Locator
  readonly confirmationMessage: Locator

  constructor(page: Page) {
    this.page = page
    this.addressForm = page.getByTestId("address-form")
    this.paymentForm = page.getByTestId("payment-form")
    this.submitButton = page.getByRole("button", { name: /place order/i })
    this.confirmationMessage = page.getByTestId("confirmation")
  }

  async goto() {
    await this.page.goto("/checkout")
  }

  async fillAddress(address = generators.address()) {
    await this.addressForm.getByLabel(/street/i).fill(address.street)
    await this.addressForm.getByLabel(/city/i).fill(address.city)
    await this.addressForm.getByLabel(/zip/i).fill(address.zip)
    await this.addressForm.getByLabel(/country/i).selectOption(address.country)
    return address
  }

  async fillPayment(payment = generators.paymentDetails()) {
    await this.paymentForm.getByLabel(/card number/i).fill(payment.cardNumber)
    await this.paymentForm.getByLabel(/expiry/i).fill(payment.expiry)
    await this.paymentForm.getByLabel(/cvv/i).fill(payment.cvv)
    return payment
  }

  async submit() {
    await this.submitButton.click()
  }

  async expectSuccess() {
    await expect(this.confirmationMessage).toBeVisible()
    await expect(this.confirmationMessage).toContainText(/thank you/i)
  }
}
```

### Complete User Flow Tests

```typescript
// e2e/checkout.spec.ts
import { test, expect } from "@playwright/test"
import { generators } from "./utils/generators"
import { CheckoutPage } from "./pages/checkout.page"
import { CartPage } from "./pages/cart.page"
import { ProductPage } from "./pages/product.page"

test.describe("Checkout Flow", () => {
  test("complete purchase with generated data", async ({ page }) => {
    const product = generators.product()
    const address = generators.address()
    const user = generators.user()

    // Seed database with generated product (via API or direct DB)
    await seedProduct(product)

    // Real user flow
    const productPage = new ProductPage(page)
    await productPage.goto(product.id)
    await productPage.addToCart()

    const cartPage = new CartPage(page)
    await cartPage.goto()
    await expect(cartPage.items).toContainText(product.name)
    await cartPage.proceedToCheckout()

    const checkoutPage = new CheckoutPage(page)
    await checkoutPage.fillAddress(address)
    await checkoutPage.fillPayment() // Uses generated payment details
    await checkoutPage.submit()

    await checkoutPage.expectSuccess()

    // Verify real database state
    const order = await getLatestOrder()
    expect(order.shippingAddress.city).toBe(address.city)
    expect(order.items[0].productId).toBe(product.id)
  })

  test("handles validation errors", async ({ page }) => {
    const checkoutPage = new CheckoutPage(page)
    await checkoutPage.goto()

    // Submit with empty fields
    await checkoutPage.submit()

    // Real validation, real error messages
    await expect(page.getByText(/street is required/i)).toBeVisible()
    await expect(page.getByText(/city is required/i)).toBeVisible()
  })
})
```

### API Mocking for External Services Only

```typescript
// e2e/checkout-with-payment-mock.spec.ts
import { test, expect } from "@playwright/test"

test.describe("Checkout with Payment Service Mock", () => {
  test.beforeEach(async ({ page }) => {
    // Mock ONLY the external payment API
    await page.route("**/api.stripe.com/**", async (route) => {
      await route.fulfill({
        status: 200,
        contentType: "application/json",
        body: JSON.stringify({
          id: "pi_test_123",
          status: "succeeded"
        })
      })
    })
  })

  test("processes payment through mocked Stripe", async ({ page }) => {
    // All application code is real - only Stripe API is mocked
    const checkoutPage = new CheckoutPage(page)
    await checkoutPage.goto()
    await checkoutPage.fillAddress()
    await checkoutPage.fillPayment()
    await checkoutPage.submit()

    await checkoutPage.expectSuccess()
  })

  test("handles payment failures gracefully", async ({ page }) => {
    await page.route("**/api.stripe.com/**", async (route) => {
      await route.fulfill({
        status: 402,
        contentType: "application/json",
        body: JSON.stringify({
          error: { message: "Card declined" }
        })
      })
    })

    const checkoutPage = new CheckoutPage(page)
    await checkoutPage.goto()
    await checkoutPage.fillAddress()
    await checkoutPage.fillPayment()
    await checkoutPage.submit()

    await expect(page.getByText(/card declined/i)).toBeVisible()
  })
})
```

### Test Isolation and Cleanup

E2E tests running against real services must be fully idempotent. Each test should create its own data, run in isolation, and clean up after itself.

```typescript
// e2e/utils/test-data.ts
import { generators } from "./generators"

interface TestContext {
  createdIds: {
    users: string[]
    products: string[]
    orders: string[]
  }
}

export async function createTestContext(): Promise<TestContext> {
  return {
    createdIds: {
      users: [],
      products: [],
      orders: []
    }
  }
}

export async function seedUser(ctx: TestContext, overrides = {}) {
  const user = { ...generators.user(), ...overrides }
  const response = await fetch("/api/test/users", {
    method: "POST",
    body: JSON.stringify(user)
  })
  const created = await response.json()
  ctx.createdIds.users.push(created.id)
  return created
}

export async function seedProduct(ctx: TestContext, overrides = {}) {
  const product = { ...generators.product(), ...overrides }
  const response = await fetch("/api/test/products", {
    method: "POST",
    body: JSON.stringify(product)
  })
  const created = await response.json()
  ctx.createdIds.products.push(created.id)
  return created
}

export async function cleanupTestContext(ctx: TestContext) {
  // Delete in reverse dependency order
  for (const id of ctx.createdIds.orders) {
    await fetch(`/api/test/orders/${id}`, { method: "DELETE" })
  }
  for (const id of ctx.createdIds.products) {
    await fetch(`/api/test/products/${id}`, { method: "DELETE" })
  }
  for (const id of ctx.createdIds.users) {
    await fetch(`/api/test/users/${id}`, { method: "DELETE" })
  }
}
```

```typescript
// e2e/checkout.spec.ts
import { test, expect } from "@playwright/test"
import { createTestContext, cleanupTestContext, seedUser, seedProduct } from "./utils/test-data"

test.describe("Checkout Flow", () => {
  let ctx: TestContext

  test.beforeEach(async () => {
    ctx = await createTestContext()
  })

  test.afterEach(async () => {
    await cleanupTestContext(ctx)
  })

  test("complete purchase with generated data", async ({ page }) => {
    // All created data is tracked for cleanup
    const user = await seedUser(ctx)
    const product = await seedProduct(ctx)

    await page.goto(`/products/${product.id}`)
    // ... rest of test

    // Order created during test is also tracked
    const orderConfirmation = await page.getByTestId("order-id").textContent()
    ctx.createdIds.orders.push(orderConfirmation)
  })
})
```

**Fixture-based approach for Playwright:**

```typescript
// e2e/fixtures.ts
import { test as base } from "@playwright/test"
import { createTestContext, cleanupTestContext, TestContext } from "./utils/test-data"

type TestFixtures = {
  testCtx: TestContext
}

export const test = base.extend<TestFixtures>({
  testCtx: async ({}, use) => {
    const ctx = await createTestContext()
    await use(ctx)
    await cleanupTestContext(ctx)
  }
})

export { expect } from "@playwright/test"
```

```typescript
// e2e/checkout.spec.ts
import { test, expect } from "./fixtures"

test("complete purchase", async ({ page, testCtx }) => {
  const product = await seedProduct(testCtx)
  // Cleanup happens automatically after test
})
```

**Database transaction rollback (alternative for faster cleanup):**

```typescript
// e2e/fixtures.ts
import { test as base } from "@playwright/test"

export const test = base.extend({
  // Start transaction before each test, rollback after
  testTransaction: async ({}, use) => {
    const txId = await fetch("/api/test/transaction/begin").then(r => r.json())
    
    await use(txId)
    
    await fetch(`/api/test/transaction/${txId}/rollback`, { method: "POST" })
  }
})
```

**Key principles for idempotent E2E tests:**

1. **Generate unique data per test run** - Never rely on pre-existing database state
2. **Track all created resources** - Maintain a manifest of everything the test creates
3. **Clean up in dependency order** - Delete child records before parents
4. **Use test-specific API endpoints** - `/api/test/*` routes that bypass auth for seeding/cleanup
5. **Isolate test users** - Each test creates its own user account when auth is involved
6. **Handle partial failures** - Cleanup should be resilient to tests that fail mid-execution
7. **Consider transaction rollback** - For speed, wrap tests in DB transactions when possible

### Visual and Accessibility Testing

```typescript
// e2e/visual.spec.ts
import { test, expect } from "@playwright/test"
import AxeBuilder from "@axe-core/playwright"

test.describe("Visual Regression", () => {
  test("dashboard matches snapshot", async ({ page }) => {
    await page.goto("/dashboard")
    await expect(page).toHaveScreenshot("dashboard.png", {
      maxDiffPixels: 100
    })
  })
})

test.describe("Accessibility", () => {
  test("checkout page is accessible", async ({ page }) => {
    await page.goto("/checkout")

    const results = await new AxeBuilder({ page }).analyze()

    expect(results.violations).toEqual([])
  })

  test("all pages meet WCAG AA", async ({ page }) => {
    const pages = ["/", "/products", "/cart", "/checkout", "/account"]

    for (const path of pages) {
      await page.goto(path)
      const results = await new AxeBuilder({ page })
        .withTags(["wcag2a", "wcag2aa"])
        .analyze()

      expect(results.violations, `Violations on ${path}`).toEqual([])
    }
  })
})
```

---

## Anti-Patterns to Avoid

### ❌ Mocking Internal Modules

```typescript
// WRONG
jest.mock("./utils/validation")
jest.mock("./services/user-service")

// This hides bugs where modules don't integrate correctly
```

### ❌ Static Fixture Files

```typescript
// WRONG
import userFixture from "./__fixtures__/user.json"

// Fixtures become stale, miss edge cases, embed assumptions
```

### ❌ Mocking React Components

```typescript
// WRONG
jest.mock("./components/Modal", () => ({ children }) => <div>{children}</div>)

// You're not testing the real component tree
```

### ❌ Testing Implementation Details

```typescript
// WRONG
test("calls setState with correct value", () => {
  const setState = jest.fn()
  jest.spyOn(React, "useState").mockReturnValue([null, setState])
  // Testing how, not what
})
```

### ❌ Snapshot Testing Complex Components

```typescript
// WRONG
test("renders correctly", () => {
  expect(render(<ComplexDashboard />)).toMatchSnapshot()
  // Snapshots become noise, get blindly updated
})
```

### ❌ Over-Isolating Unit Tests

```typescript
// WRONG - mocking everything the function touches
test("processOrder", () => {
  const mockValidate = jest.fn()
  const mockCalculate = jest.fn()
  const mockSave = jest.fn()
  const mockNotify = jest.fn()

  // If you need 4 mocks, write an integration test instead
})
```

---

## Test File Organization

```
src/
├── features/
│   └── checkout/
│       ├── checkout.ts
│       ├── checkout.test.ts          # Unit tests
│       ├── checkout.integration.ts   # Integration tests
│       └── schemas.ts
├── services/
│   └── payment/
│       ├── payment.ts
│       ├── payment.test.ts
│       └── payment.live.ts
│       └── payment.test-impl.ts     # Test implementation
└── components/
    └── checkout-form/
        ├── checkout-form.tsx
        └── checkout-form.test.tsx   # Component tests

e2e/
├── checkout.spec.ts
├── pages/
│   ├── checkout.page.ts
│   └── cart.page.ts
└── utils/
    └── generators.ts
```

---

## Quick Reference

### When to Use Each Test Type

| Scenario | Test Type |
|----------|-----------|
| Pure function logic | Unit |
| Schema validation | Unit |
| Effect computation | Unit |
| Component with side effects | Integration |
| Multiple services interacting | Integration |
| Real database operations | Integration |
| Full user journey | E2E |
| Cross-browser behavior | E2E |
| Visual regression | E2E |

### What to Mock

| Test Type | Should Mock | Should NOT Mock |
|-----------|-------------|-----------------|
| **Unit** | External APIs, internal services, databases, time, randomness | React components, pure functions, utility modules |
| **Integration** | External APIs only, time (when needed) | Internal services, databases, React components |
| **E2E** | External APIs only (via network interception) | Everything else—test the real system |

### Data Generation Checklist

- [ ] Schema defined with appropriate constraints
- [ ] Arbitrary derivable from schema
- [ ] Custom arbitrary only if schema constraints insufficient
- [ ] Seeded generation for reproducible tests when needed
- [ ] Property tests cover invariants, not just examples

### Test Idempotency Checklist

- [ ] Each test generates its own unique data
- [ ] All created resources are tracked for cleanup
- [ ] Cleanup runs in `afterEach`/`afterAll` hooks
- [ ] Cleanup handles partial test failures gracefully
- [ ] Tests don't depend on pre-existing database state
- [ ] Tests can run in parallel without conflicts
- [ ] Test user accounts are isolated per test (when auth involved)
