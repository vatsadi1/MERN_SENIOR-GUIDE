Yeh raha **`TESTING_STANDARDS.md`** direct copy-paste format me.

Aap seedhe is message ke niche **Copy** button dabakar isse copy kar
sakte ho aur apne GitHub repository (`MERN_SENIOR-GUIDE`) me
**`TESTING_STANDARDS.md`** naam se save kar sakte ho:

------------------------------------------------------------------------

# 🧪 Unit & Integration Testing Standards (Node.js Native Runner)

> **Authoritative Playbook for Enterprise Backend Testing**\
> *Writing lightning-fast, zero-dependency unit and integration tests
> using `node:test` and `node:assert/strict`.*

------------------------------------------------------------------------

## 📑 Table of Contents

1.  [Core Philosophy & The AAA
    Pattern](#1-core-philosophy--the-aaa-pattern)
2.  [Modern Architecture: `node:test` vs
    Jest](#2-modern-architecture-nodetest-vs-jest)
3.  [The 6 Essential Assertion
    Primitives](#3-the-6-essential-assertion-primitives)
4.  [Pattern 1: Pure Functions & Financial
    Arithmetic](#4-pattern-1-pure-functions--financial-arithmetic)
5.  [Pattern 2: Mongoose Schema Constraints (Zero-DB
    `validateSync`)](#5-pattern-2-mongoose-schema-constraints-zero-db-validatesync)
6.  [Pattern 3: Zod Contract & Boundary Testing
    (`safeParse`)](#6-pattern-3-zod-contract--boundary-testing-safeparse)
7.  [Pattern 4: Async Services & Security Guardrails
    (`assert.rejects`)](#7-pattern-4-async-services--security-guardrails-assertrejects)
8.  [Pattern 5: Isolated Mocking & Method
    Stubbing](#8-pattern-5-isolated-mocking--method-stubbing)
9.  [CLI Runner Execution & Debugging
    Commands](#9-cli-runner-execution--debugging-commands)
10. [The 7 Fatal Testing
    Anti-Patterns](#10-the-7-fatal-testing-anti-patterns)
11. [Pre-Flight PR Testing
    Checklist](#11-pre-flight-pr-testing-checklist)

------------------------------------------------------------------------

## 1. Core Philosophy & The AAA Pattern

In enterprise backend engineering, untested code is considered broken
code by default. Unit testing verifies that discrete units of software
(functions, models, validators, services) perform as designed in total
isolation.

Every production test must follow the **AAA (Arrange-Act-Assert)**
pattern without exception:

    ┌────────────────────────────────────────────────────────┐
    │ 1. ARRANGE (State Setup)                               │
    │    Prepare test inputs, mocks, and fixtures.           │
    └──────────────────────────┬─────────────────────────────┘
                               │
                               ▼
    ┌────────────────────────────────────────────────────────┐
    │ 2. ACT (Execution)                                     │
    │    Execute the target unit of code under test.         │
    └──────────────────────────┬─────────────────────────────┘
                               │
                               ▼
    ┌────────────────────────────────────────────────────────┐
    │ 3. ASSERT (Verification)                               │
    │    Validate actual output against expected invariant.  │
    └────────────────────────────────────────────────────────┘

------------------------------------------------------------------------

## 2. Modern Architecture: `node:test` vs Jest

Modern Node.js (v20+) contains a battle-tested, high-performance native
test runner built into the C++ runtime.

  -----------------------------------------------------------------------
  Metric                  🐢 Legacy Frameworks    ⚡ Native Runner
                          (Jest / Mocha)          (`node:test`)
  ----------------------- ----------------------- -----------------------
  **Dependencies**        50MB--100MB of npm      **Zero dependencies**
                          dependencies (`jest`,   (100% built into
                          `babel-jest`,           Node.js core)
                          `ts-jest`)              

  **Boot Latency**        2.5s -- 4.0s startup    **\< 0.05s** instant
                          overhead per run        execution

  **Native ESM**          Requires complex        Native `import` /
                          Babel/Experimental      `export` out of the box
                          flags                   

  **Memory Footprint**    Heavy V8 heap overhead  Ultra-lightweight
                                                  native C++ execution
  -----------------------------------------------------------------------

### Standard Import Contract

``` javascript
import { describe, it, beforeEach, afterEach } from 'node:test';
import assert from 'node:assert/strict';
```

------------------------------------------------------------------------

## 3. The 6 Essential Assertion Primitives

Always import from `node:assert/strict` to enforce strict type and value
equality (`===` instead of `==`).

  ------------------------------------------------------------------------------------------------------------------------------------------
  Primitive                                    Usage                   Production Example
  -------------------------------------------- ----------------------- ---------------------------------------------------------------------
  `assert.strictEqual(actual, expected)`       Primitives (Numbers,    `assert.strictEqual(cart.deliveryFeePaise, 4000);`
                                               Strings, Booleans)      

  `assert.notStrictEqual(a, b)`                Proving values must     `assert.notStrictEqual(order.status, 'CANCELLED');`
                                               differ                  

  `assert.deepStrictEqual(actual, expected)`   Objects and Arrays      `assert.deepStrictEqual(cart.items, []);`
                                               (Deep property          
                                               inspection)             

  `assert.ok(condition)`                       Boolean truthiness      `assert.ok(result.token);`
                                               checks                  

  `assert.rejects(asyncFn, errorPattern)`      **Async functions**     `await assert.rejects(async () => await fn(), { message: '...' });`
                                               throwing errors         

  `assert.throws(syncFn, errorPattern)`        **Sync functions**      `assert.throws(() => parse(raw), /Invalid JSON/);`
                                               throwing errors         
  ------------------------------------------------------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 4. Pattern 1: Pure Functions & Financial Arithmetic

Pure functions have no side effects, making them the fastest and most
deterministic tests in your codebase.

### Source Function (`cart.utils.js`):

``` javascript
export function calculateDeliveryFee(itemsTotalPaise) {
  if (itemsTotalPaise === 0) return 0;
  if (itemsTotalPaise >= 50000) return 0; // >= ₹500 Free Delivery Threshold
  return 4000; // Standard ₹40 delivery fee
}
```

### Test Specification (`cart.utils.test.js`):

``` javascript
describe('calculateDeliveryFee()', () => {
  it('returns 0 when cart is empty (0 paise)', () => {
    // Arrange
    const subtotal = 0;

    // Act
    const fee = calculateDeliveryFee(subtotal);

    // Assert
    assert.strictEqual(fee, 0);
  });

  it('charges standard 4000 paise (₹40) fee below 50000 paise threshold', () => {
    const fee = calculateDeliveryFee(49900); // ₹499.00
    assert.strictEqual(fee, 4000);
  });

  it('waives delivery fee (0 paise) at exact 50000 paise (₹500) threshold', () => {
    const fee = calculateDeliveryFee(50000);
    assert.strictEqual(fee, 0);
  });
});
```

------------------------------------------------------------------------

## 5. Pattern 2: Mongoose Schema Constraints (Zero-DB `validateSync`)

Never connect to a live MongoDB cluster to test schema validation rules.
Use Mongoose's synchronous `validateSync()` method to validate schemas
in-memory within milliseconds.

### Test Specification (`order.model.test.js`):

``` javascript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { Order } from '../../../src/modules/orders/orders.model.js';

describe('Order Model Schema Constraints', () => {
  it('requires customer reference field', () => {
    // 1. Arrange: Create doc without required 'customer'
    const invalidOrder = new Order({
      items: [{ medicine: '65f1a2b3c4d5e6f7a8b9c0d1', quantity: 2, pricePaise: 3000 }],
      totalAmountPaise: 6000,
    });

    // 2. Act: Run sync schema validation in memory
    const error = invalidOrder.validateSync();

    // 3. Assert: Verify rejection details
    assert.ok(error);
    assert.ok(error.errors.customer);
    assert.strictEqual(error.errors.customer.kind, 'required');
  });

  it('rejects floating-point currency values (FinTech integer paise rule)', () => {
    const decimalOrder = new Order({
      customer: '65f1a2b3c4d5e6f7a8b9c0d2',
      totalAmountPaise: 1250.50, // Floating point violation
    });

    const error = decimalOrder.validateSync();

    assert.ok(error);
    assert.ok(error.errors.totalAmountPaise);
    assert.strictEqual(error.errors.totalAmountPaise.kind, 'user defined');
  });
});
```

------------------------------------------------------------------------

## 6. Pattern 3: Zod Contract & Boundary Testing (`safeParse`)

Zod schemas guard the boundaries of your application against malicious
or malformed client payloads. Test both positive inputs and edge-case
boundary violations.

### Test Specification (`cart.validation.test.js`):

``` javascript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { addItemSchema } from '../../../src/modules/cart/cart.validation.js';

describe('addItemSchema Zod Contract', () => {
  it('accepts compliant payload with 24-character hex ObjectId', () => {
    const payload = {
      medicineId: '65f1a2b3c4d5e6f7a8b9c0d1',
      quantity: 5,
    };

    const result = addItemSchema.safeParse(payload);

    assert.strictEqual(result.success, true);
    assert.strictEqual(result.data.quantity, 5);
  });

  it('rejects non-positive whole number quantities (quantity: 0 or negative)', () => {
    const payload = {
      medicineId: '65f1a2b3c4d5e6f7a8b9c0d1',
      quantity: 0,
    };

    const result = addItemSchema.safeParse(payload);

    assert.strictEqual(result.success, false);
    const targetError = result.error.issues.find((issue) => issue.path.includes('quantity'));
    assert.ok(targetError);
  });

  it('strips unauthorized / unknown properties automatically', () => {
    const payloadWithInjection = {
      medicineId: '65f1a2b3c4d5e6f7a8b9c0d1',
      quantity: 1,
      adminOverride: true, // Malicious injected property
    };

    const result = addItemSchema.safeParse(payloadWithInjection);

    assert.strictEqual(result.success, true);
    assert.strictEqual(result.data.adminOverride, undefined); // Stripped!
  });
});
```

------------------------------------------------------------------------

## 7. Pattern 4: Async Services & Security Guardrails (`assert.rejects`)

When testing asynchronous business rules that throw operational errors
(`AppError`, `ApiError`), use `assert.rejects()`.

### The Canonical Formula:

``` javascript
await assert.rejects(
  async () => {
    await targetServiceMethod(arguments);
  },
  {
    message: 'Expected error message substring or exact match',
    statusCode: 400, // optional property matcher
  }
);
```

### Test Specification (`cart.service.test.js`):

``` javascript
describe('cartService.addItem Security Guardrails', () => {
  const dummyCustomerId = '65f1a2b3c4d5e6f7a8b9c0d1';
  const dummyMedicineId = '65f1a2b3c4d5e6f7a8b9c0d2';

  it('rejects non-integer / decimal quantity (quantity: 2.5)', async () => {
    await assert.rejects(
      async () => {
        await cartService.addItem(dummyCustomerId, {
          medicineId: dummyMedicineId,
          quantity: 2.5,
        });
      },
      {
        message: 'Quantity must be a positive whole number (minimum 1)',
      }
    );
  });

  it('rejects malformed / non-ObjectId medicineId format', async () => {
    await assert.rejects(
      async () => {
        await cartService.addItem(dummyCustomerId, {
          medicineId: 'malicious-string-123',
          quantity: 1,
        });
      },
      {
        message: 'Invalid medicine ID format',
      }
    );
  });
});
```

------------------------------------------------------------------------

## 8. Pattern 5: Isolated Mocking & Method Stubbing

When a service method interacts with an external dependency
(e.g. database query, payment gateway, SMS provider), stub the target
method to guarantee isolated execution.

### Test Specification (`medicine.stub.test.js`):

``` javascript
import { describe, it } from 'node:test';
import assert from 'node:assert/strict';
import { Medicine } from '../../../src/modules/medicines/medicines.model.js';
import { cartService } from '../../../src/modules/cart/cart.service.js';

describe('cartService with Stubbed Model', () => {
  it('throws 404 NOT_FOUND when medicine does not exist in inventory', async () => {
    // 1. Arrange: Hijack findById to simulate DB miss
    const originalFindById = Medicine.findById;
    Medicine.findById = async () => null;

    try {
      // 2. Act & Assert: Execute without live database connection
      await assert.rejects(
        async () => {
          await cartService.addItem('65f1a2b3c4d5e6f7a8b9c0d1', {
            medicineId: '65f1a2b3c4d5e6f7a8b9c0d2',
            quantity: 1,
          });
        },
        {
          message: 'Medicine with ID 65f1a2b3c4d5e6f7a8b9c0d2 not found',
        }
      );
    } finally {
      // 3. Cleanup: Always restore original method in finally block
      Medicine.findById = originalFindById;
    }
  });
});
```

------------------------------------------------------------------------

## 9. CLI Runner Execution & Debugging Commands

Always run individual target test suites during development instead of
executing the entire project test suite on every keystroke.

``` bash
# 1. Execute single test suite
node --test --import ./test/setup.js "test/modules/cart/cart.service.test.js"

# 2. Filter tests by name pattern (Regex matching)
node --test --import ./test/setup.js --test-name-pattern="rejects invalid" "test/modules/cart/cart.service.test.js"

# 3. Watch Mode (Auto re-run on file save)
node --test --watch --import ./test/setup.js "test/modules/cart/cart.service.test.js"

# 4. Run entire test suite across all modules
npm test
```

------------------------------------------------------------------------

## 10. The 7 Fatal Testing Anti-Patterns

  ------------------------------------------------------------------------------
  Anti-Pattern            The Consequence         The Solution
  ----------------------- ----------------------- ------------------------------
  **Omitting `await` on   Test passes silently    Always prepend
  `assert.rejects`**      even if the target      `await assert.rejects(...)`.
                          method does not throw,  
                          producing a false       
                          positive.               

  **Omitting `await`      The closure completes   Always `await` the inner
  inside the async        before the promise      target method call.
  closure**               rejects, leaking an     
                          unhandled promise       
                          rejection.              

  **Using `strictEqual`   Always fails            Always use
  on Objects/Arrays**     (`{a:1} !== {a:1}`) due `assert.deepStrictEqual()`.
                          to pointer reference    
                          inequality in JS.       

  **Hitting live external Tests become slow,      Use `validateSync()`, memory
  databases in Unit       flakey, and fail when   stubs, or mock adapters.
  Tests**                 running offline or in   
                          CI/CD environments.     

  **Test State            One test modifies       Reset state in `beforeEach` or
  Pollution**             global state, causing   `afterEach` hooks.
                          subsequent independent  
                          tests to fail.          

  **Testing               Tests break during      Test public inputs and
  Implementation          minor refactoring even  outputs, not private
  Details**               though public behavior  variables.
                          remains correct.        

  **Missing `finally` on  A failing test leaves   Always restore original
  Mock Restorations**     monkey-patched mocks    methods inside a `finally`
                          active, corrupting the  block.
                          rest of the suite.      
  ------------------------------------------------------------------------------

------------------------------------------------------------------------

## 11. Pre-Flight PR Testing Checklist

Before submitting a Pull Request for any new feature or bugfix, ensure:

-   [ ] **Pure Invariants:** All calculations and utilities have 100%
    boundary test coverage.
-   [ ] **Model Validation:** Required fields, enums, and FinTech
    integer rules are covered via `validateSync()`.
-   [ ] **Contract Validation:** Zod schemas reject invalid inputs and
    strip malicious keys via `safeParse()`.
-   [ ] **Guardrails Verified:** Every `throw ApiError.badRequest()` or
    `404` branch has an accompanying `assert.rejects()` test.
-   [ ] **Zero Live DB Dependency:** Unit tests execute completely
    offline without network latency.
-   [ ] **Clean Suite Execution:** All tests pass with 0 warnings, 0
    unhandled rejections, and clean teardowns.

------------------------------------------------------------------------

Aap is guide ko copy karke repository me commit kar sakte ho!

Ab jab aap testing ke saare patterns samajh chuke ho, **kya hum wapas
`cart.service.test.js` ke "Invalid Medicine ID" wale test par chalein?**
🚀
