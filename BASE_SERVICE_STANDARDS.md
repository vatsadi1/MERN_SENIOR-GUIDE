
---

# 🏛️ Base Service Layer Architecture Standards (Node.js & Mongoose)
> **Enterprise Engineering Playbook: Abstract Repository & Safe Query Engine**  
> *Authoritative standards for generic CRUD encapsulation, query safety, pagination engines, and ACID transactions in Mongoose.*

---

## 📑 Table of Contents
1. [Architectural Role & Mental Model](#1-architectural-role--mental-model)
2. [The 8 Golden Rules of Base Service](#2-the-8-golden-rules-of-base-service)
   - [Rule 1: Strict Dependency Injection & Resource Identity](#rule-1-strict-dependency-injection--resource-identity)
   - [Rule 2: Fail-Fast ID Sanitation (Zero CastError Leaks)](#rule-2-fail-fast-id-sanitation-zero-casterror-leaks)
   - [Rule 3: Deterministic Query Semantics (404 Guardrails vs Null)](#rule-3-deterministic-query-semantics-404-guardrails-vs-null)
   - [Rule 4: Bulletproof Pagination Engine (`findPaginated`)](#rule-4-bulletproof-pagination-engine-findpaginated)
   - [Rule 5: Universal ACID Transaction Forwarding (`{ session }`)](#rule-5-universal-acid-transaction-forwarding--session-)
   - [Rule 6: Mandatory Schema Validation on Updates (`runValidators: true`)](#rule-6-mandatory-schema-validation-on-updates-runvalidatorstrue)
   - [Rule 7: Enterprise Soft-Delete vs Hard-Delete Architecture](#rule-7-enterprise-soft-delete-vs-hard-delete-architecture)
   - [Rule 8: Zero Transport & Zero Domain Pollution](#rule-8-zero-transport--zero-domain-pollution)
3. [Complete Production-Grade `BaseService.js` Implementation](#3-complete-production-grade-baseservicejs-implementation)
4. [Real-World Subclassing Example (`OrderService.js`)](#4-real-world-subclassing-example-orderservicejs)
5. [Fatal Anti-Patterns & Common Traps](#5-fatal-anti-patterns--common-traps)
6. [Pre-Flight Production Checklist](#6-pre-flight-production-checklist)

---

## 1. Architectural Role & Mental Model

In a mature layered architecture, the **Service Layer** is responsible for business logic. However, 70-80% of database interactions are repetitive: finding a document by ID, validating that the ID is a 24-character hexadecimal string, checking existence, throwing standard 404 errors, calculating pagination offsets, and managing database sessions.

If every domain service (`OrderService`, `StoreService`, `UserService`, `CartService`) writes this boilerplate independently:
- **Code Duplication explodes.**
- **Error handling becomes inconsistent** (one service returns `null`, another throws `404`, another lets a raw `CastError` crash the server as `500`).
- **Memory leaks occur** from unbounded `.find()` queries lacking pagination limits.

```
┌────────────────────────────────────────────────────────┐
│                   HTTP Request                         │
└──────────────────────────┬─────────────────────────────┘
                           │
                           ▼
               Controller Layer (Transport)
                           │
                           ▼
             Domain Service (Business Rules)
           (e.g., OrderService, StoreService)
                           │
                  extends BaseService
                           │
                           ▼
          BaseService (Generic Data Access Layer)
   ┌───────────────────────┼───────────────────────┐
   ▼                       ▼                       ▼
ID Guardrails       Clamped Pagination        ACID Session
   └───────────────────────┬───────────────────────┘
                           ▼
                 Mongoose Model / Atlas
```

> **The Core Philosophy:**  
> `BaseService` acts as an **Abstract Generic Repository Pattern** wrapper over Mongoose. It handles standard database hygiene, validation, and querying safely so that Domain Services remain 100% focused on business logic.

---

## 2. The 8 Golden Rules of Base Service

---

### Rule 1: Strict Dependency Injection & Resource Identity
The `BaseService` must never directly require or instantiate a Mongoose Model. Instead, the specific Model must be injected via the constructor along with a human-readable `resourceName`.

* **The Senior Rule:** Subclasses must pass `super(Model, 'ResourceName')`.
* **Why it matters:** 
  1. Allows clear, business-meaningful error messages (e.g., `"Order not found with ID: xyz"` instead of `"Document not found with ID: xyz"`).
  2. Makes unit testing and mocking trivial without relying on global database connections.

```javascript
// ❌ BAD: Hardcoded model inside generic service
class BaseService {
  constructor() {
    this.model = require('./models/user.model'); // Completely broken abstraction
  }
}

// ✅ GOOD: Constructor injection with identity
class BaseService {
  constructor(model, resourceName = 'Resource') {
    if (!model) {
      throw new Error(`[BaseService] Model must be provided for resource "${resourceName}"`);
    }
    this.model = model;
    this.resourceName = resourceName;
  }
}
```

---

### Rule 2: Fail-Fast ID Sanitation (Zero CastError Leaks)
MongoDB ObjectIds are strictly 24-character hex strings. If a client sends an invalid string (e.g. `GET /api/v1/orders/undefined` or `GET /api/v1/orders/123-abc`), Mongoose queries throw a native `CastError`. Without an explicit guardrail, this bubbles up as an unhandled `500 Internal Server Error`.

* **The Senior Rule:** Every single method that accepts an ID (`findById`, `updateById`, `deleteById`, `softDeleteById`) must execute `this.assertValidId(id)` prior to touching the database.
* **Standard Error Response:** Must throw an operational `AppError` with HTTP `400 BAD_REQUEST` and error code `INVALID_ID_FORMAT`.

```javascript
// ✅ GOOD: Guardrail prevents CastError completely
assertValidId(id) {
  if (!id || !mongoose.Types.ObjectId.isValid(String(id))) {
    throw new AppError(
      `Invalid ${this.resourceName} ID format: "${id}"`,
      400,
      'INVALID_ID_FORMAT'
    );
  }
}
```

---

### Rule 3: Deterministic Query Semantics (404 Guardrails vs Null)
Different query operations have different semantic expectations:
1. **Targeted Lookup (`findById`):** The caller expects the document to exist. If it does not, it is inherently an exceptional condition. `findById` must **automatically throw a 404 AppError**.
2. **Conditional Search (`findOne`):** The caller is searching based on business criteria (e.g., finding an existing user by email or active cart). If not found, it is normal. `findOne` must **return `null`** without throwing.
3. **High-Speed Presence Check (`exists`):** Checking if an entity exists should never load thousands of bytes into memory. It must use `.exists()` or `.countDocuments({ _id: id })` to return a lean boolean `true/false`.

```javascript
// ❌ BAD: Loading whole document into RAM just to check if it exists
async checkOrderExists(orderId) {
  const order = await Order.findById(orderId); // Heavy RAM consumption!
  return !!order;
}

// ✅ GOOD: Ultra-fast index-only boolean lookup
async exists(filter = {}, options = {}) {
  const doc = await this.model.exists(filter).session(options.session || null);
  return Boolean(doc);
}
```

---

### Rule 4: Bulletproof Pagination Engine (`findPaginated`)
Unbounded queries (`Model.find({})`) are the #1 cause of Out-Of-Memory (OOM) crashes in production Node.js applications when database tables grow over 50,000 documents.

* **The Senior Rule:** The `BaseService.findPaginated` engine must enforce:
  1. **Hard Limit Clamping:** Enforce a maximum cap (e.g., maximum 100 items). If a malicious client sends `?limit=1000000`, clamp it to 100.
  2. **Safe Offsets:** Sanitize `page` (minimum 1) and calculate `skip = (page - 1) * limit`.
  3. **Parallel Execution:** Execute the data query (`.find()`) and the count query (`.countDocuments()`) concurrently via `Promise.all()`.
  4. **Lean Hydration:** Default list queries to `.lean()` to bypass Mongoose virtual/getter overhead for maximum throughput.

```javascript
// Standard Unified Pagination Envelope
{
  success: true,
  data: [...],
  pagination: {
    total: 1450,
    page: 2,
    limit: 20,
    totalPages: 73,
    hasNextPage: true,
    hasPrevPage: true
  }
}
```

---

### Rule 5: Universal ACID Transaction Forwarding (`{ session }`)
FinTech, E-Commerce, and Medical platforms require atomic multi-document updates (e.g. Deducting Wallet + Creating Order + Decrementing Inventory).

* **The Senior Rule:** Every write and read method in `BaseService` must accept an `options` object that can receive `{ session }`, and explicitly bind it to the Mongoose query.
* If a session is present, the query joins the transaction; if absent, it executes normally.

```javascript
// ✅ GOOD: Universal session support in every method
async updateById(id, updateData, options = {}) {
  this.assertValidId(id);
  const { session, runValidators = true, new: returnNew = true } = options;

  const doc = await this.model.findByIdAndUpdate(id, updateData, {
    new: returnNew,
    runValidators,
    session: session || null, // Seamlessly binds to active ACID transaction
  });
  ...
}
```

---

### Rule 6: Mandatory Schema Validation on Updates (`runValidators: true`)
By default, Mongoose's `findByIdAndUpdate`, `findOneAndUpdate`, and `updateOne` **do not run schema validators**! A developer could update an order total to a negative number or store an invalid enum status if validation is not enforced.

* **The Senior Rule:** All update methods in `BaseService` must explicitly configure `{ runValidators: true }`.
* Never allow unvalidated data to bypass your Mongoose schema rules during updates.

```javascript
// ❌ DANGEROUS: Mongoose default behavior ignores schema validators!
await Order.findByIdAndUpdate(id, { status: 'INVALID_STATUS' }); // Succeeds silently!

// ✅ SECURE: BaseService enforces schema validators on every mutation
await this.model.findByIdAndUpdate(id, updateData, {
  new: true,
  runValidators: true, // Throws ValidationError if rules are violated
  session: options.session || null,
});
```

---

### Rule 7: Enterprise Soft-Delete vs Hard-Delete Architecture
In enterprise, medical, and banking platforms, physical deletion (`DELETE FROM table` or `findByIdAndDelete`) violates compliance and audit mandates. Records must be soft-deleted.

* **The Senior Rule:** `BaseService` must provide two distinct mechanisms:
  - `deleteById(id)`: Physical hard purge (for ephemeral caches, test cleanup, or temporary draft data).
  - `softDeleteById(id, options)`: Sets `isDeleted: true`, `deletedAt: new Date()`, and optionally logs `deletedBy: userId`.
* Standard queries should automatically filter out soft-deleted documents unless explicitly instructed otherwise.

---

### Rule 8: Zero Transport & Zero Domain Pollution
`BaseService` is strictly an abstraction layer over your data store.

* **The Forbidden List:**
  - ❌ **No Express objects:** Never pass `req`, `res`, or `next` into `BaseService`.
  - ❌ **No HTTP Status Codes in Business Calls:** BaseService throws `AppError` with semantic codes; it does not handle HTTP cookies or redirect URLs.
  - ❌ **No Domain Business Logic:** Never put business calculations (like discounts, tax rates, or SMS triggers) inside `BaseService`.

---

## 3. Complete Production-Grade `BaseService.js` Implementation

Save this file at `src/shared/BaseService.js`:

```javascript
/**
 * @file BaseService.js
 * @description Enterprise-grade generic Base Service encapsulating safe Mongoose CRUD,
 * validation guardrails, clamped pagination, and ACID transaction forwarding.
 */

const mongoose = require('mongoose');
const { AppError } = require('./AppError');

class BaseService {
  /**
   * @param {mongoose.Model} model - Mongoose Model instance
   * @param {string} resourceName - Entity name for human-readable error messages (e.g. 'Order')
   */
  constructor(model, resourceName = 'Resource') {
    if (!model) {
      throw new Error(`[BaseService] Model must be provided for resource "${resourceName}"`);
    }
    this.model = model;
    this.resourceName = resourceName;
  }

  // ==========================================
  // 🛡️ ID & EXISTENCE GUARDRAILS
  // ==========================================

  /**
   * Asserts that an ID is a valid 24-character hexadecimal MongoDB ObjectId.
   * Throws 400 Bad Request if invalid to prevent raw 500 CastErrors.
   * @param {string|mongoose.Types.ObjectId} id
   * @throws {AppError}
   */
  assertValidId(id) {
    if (!id || !mongoose.Types.ObjectId.isValid(String(id))) {
      throw new AppError(
        `Invalid ${this.resourceName} ID format: "${id}"`,
        400,
        'INVALID_ID_FORMAT'
      );
    }
  }

  /**
   * Checks whether at least one document matches the specified filter.
   * Highly efficient: uses index scan without loading full documents into memory.
   * @param {Object} filter - MongoDB query filter
   * @param {Object} [options={}] - Query options { session }
   * @returns {Promise<boolean>}
   */
  async exists(filter = {}, options = {}) {
    const doc = await this.model.exists(filter).session(options.session || null);
    return Boolean(doc);
  }

  /**
   * Fast check to see if a document exists by its ID.
   * @param {string} id
   * @param {Object} [options={}] - Query options { session }
   * @returns {Promise<boolean>}
   */
  async existsById(id, options = {}) {
    this.assertValidId(id);
    return this.exists({ _id: id }, options);
  }

  /**
   * Counts the number of documents matching a filter.
   * @param {Object} filter
   * @param {Object} [options={}] - { session }
   * @returns {Promise<number>}
   */
  async count(filter = {}, options = {}) {
    return this.model
      .countDocuments(filter)
      .session(options.session || null)
      .exec();
  }

  // ==========================================
  // 🔍 READ OPERATIONS
  // ==========================================

  /**
   * Finds a document by its ID.
   * Throws a standard 404 AppError if the document is not found.
   * @param {string} id
   * @param {Object} [options={}] - { select, populate, session, lean }
   * @returns {Promise<Object>} Mongoose document or plain object
   * @throws {AppError} 404 if not found
   */
  async findById(id, options = {}) {
    this.assertValidId(id);

    const { select, populate, session, lean = false } = options;
    let query = this.model.findById(id);

    if (select) query = query.select(select);
    if (populate) query = query.populate(populate);
    if (session) query = query.session(session);
    if (lean) query = query.lean();

    const document = await query.exec();

    if (!document) {
      throw new AppError(
        `${this.resourceName} not found with ID: ${id}`,
        404,
        `${this.resourceName.toUpperCase()}_NOT_FOUND`
      );
    }

    return document;
  }

  /**
   * Finds a single document matching the filter criteria.
   * Returns null if not found (does NOT throw 404).
   * @param {Object} filter
   * @param {Object} [options={}] - { select, populate, session, sort, lean }
   * @returns {Promise<Object|null>}
   */
  async findOne(filter = {}, options = {}) {
    const { select, populate, session, sort, lean = false } = options;
    let query = this.model.findOne(filter);

    if (select) query = query.select(select);
    if (populate) query = query.populate(populate);
    if (sort) query = query.sort(sort);
    if (session) query = query.session(session);
    if (lean) query = query.lean();

    return query.exec();
  }

  /**
   * Finds multiple documents matching filter criteria without pagination.
   * Suitable for dropdowns, reference lookups, or bounded sets.
   * @param {Object} filter
   * @param {Object} [options={}] - { select, populate, session, sort, limit, lean }
   * @returns {Promise<Array<Object>>}
   */
  async findMany(filter = {}, options = {}) {
    const { select, populate, session, sort = { createdAt: -1 }, limit, lean = true } = options;
    let query = this.model.find(filter);

    if (select) query = query.select(select);
    if (populate) query = query.populate(populate);
    if (sort) query = query.sort(sort);
    if (limit) query = query.limit(limit);
    if (session) query = query.session(session);
    if (lean) query = query.lean();

    return query.exec();
  }

  /**
   * High-performance Paginated Query Engine.
   * Enforces hard limit clamping, concurrent counting, and standard metadata envelopes.
   * @param {Object} filter - MongoDB query filter
   * @param {Object} [paginationOptions={}] - { page, limit, sort, select, populate }
   * @param {Object} [queryOptions={}] - { session, maxLimit }
   * @returns {Promise<{ data: Array<Object>, pagination: Object }>}
   */
  async findPaginated(filter = {}, paginationOptions = {}, queryOptions = {}) {
    const maxLimit = queryOptions.maxLimit || 100;
    const page = Math.max(1, parseInt(paginationOptions.page, 10) || 1);
    const limit = Math.min(maxLimit, Math.max(1, parseInt(paginationOptions.limit, 10) || 20));
    const skip = (page - 1) * limit;

    const sort = paginationOptions.sort || { createdAt: -1 };
    const select = paginationOptions.select || null;
    const populate = paginationOptions.populate || null;
    const session = queryOptions.session || null;

    let dataQuery = this.model
      .find(filter)
      .sort(sort)
      .skip(skip)
      .limit(limit)
      .lean();

    if (select) dataQuery = dataQuery.select(select);
    if (populate) dataQuery = dataQuery.populate(populate);
    if (session) dataQuery = dataQuery.session(session);

    // Parallel execution for optimal latency
    const [data, total] = await Promise.all([
      dataQuery.exec(),
      this.model.countDocuments(filter).session(session).exec(),
    ]);

    const totalPages = Math.ceil(total / limit) || 1;

    return {
      data,
      pagination: {
        total,
        page,
        limit,
        totalPages,
        hasNextPage: page < totalPages,
        hasPrevPage: page > 1,
      },
    };
  }

  // ==========================================
  // ✍️ WRITE & MUTATION OPERATIONS
  // ==========================================

  /**
   * Creates a single new document with full schema validation.
   * @param {Object} payload - Document data
   * @param {Object} [options={}] - { session }
   * @returns {Promise<Object>} Created document
   */
  async create(payload, options = {}) {
    const { session } = options;

    if (session) {
      // Mongoose requires array syntax for session creation
      const [document] = await this.model.create([payload], { session });
      return document;
    }

    const document = new this.model(payload);
    return document.save();
  }

  /**
   * Bulk creates multiple documents in an atomic operation.
   * @param {Array<Object>} payloadArray
   * @param {Object} [options={}] - { session, ordered }
   * @returns {Promise<Array<Object>>}
   */
  async createMany(payloadArray, options = {}) {
    if (!Array.isArray(payloadArray) || payloadArray.length === 0) {
      return [];
    }
    return this.model.insertMany(payloadArray, {
      session: options.session || null,
      ordered: options.ordered !== undefined ? options.ordered : true,
    });
  }

  /**
   * Updates an existing document by its ID with validation enforced.
   * Throws 404 AppError if the document does not exist.
   * @param {string} id
   * @param {Object} updateData - Update operations ($set, etc.)
   * @param {Object} [options={}] - { session, runValidators, new, populate, select }
   * @returns {Promise<Object>} Updated document
   * @throws {AppError} 404 if not found
   */
  async updateById(id, updateData, options = {}) {
    this.assertValidId(id);

    const {
      session,
      runValidators = true,
      new: returnNew = true,
      populate,
      select,
    } = options;

    let query = this.model.findByIdAndUpdate(id, updateData, {
      new: returnNew,
      runValidators,
      session: session || null,
    });

    if (select) query = query.select(select);
    if (populate) query = query.populate(populate);

    const updatedDocument = await query.exec();

    if (!updatedDocument) {
      throw new AppError(
        `${this.resourceName} not found with ID: ${id}`,
        404,
        `${this.resourceName.toUpperCase()}_NOT_FOUND`
      );
    }

    return updatedDocument;
  }

  /**
   * Updates a single document matching a filter.
   * Returns the updated document or null if not found.
   * @param {Object} filter
   * @param {Object} updateData
   * @param {Object} [options={}] - { session, runValidators, new }
   * @returns {Promise<Object|null>}
   */
  async updateOne(filter, updateData, options = {}) {
    const { session, runValidators = true, new: returnNew = true } = options;

    return this.model.findOneAndUpdate(filter, updateData, {
      new: returnNew,
      runValidators,
      session: session || null,
    }).exec();
  }

  // ==========================================
  // 🗑️ DELETION OPERATIONS
  // ==========================================

  /**
   * Hard Deletes a document by ID.
   * Physical purge from the database. Throws 404 if not found.
   * @param {string} id
   * @param {Object} [options={}] - { session }
   * @returns {Promise<Object>} Deleted document
   * @throws {AppError} 404 if not found
   */
  async deleteById(id, options = {}) {
    this.assertValidId(id);

    const deletedDocument = await this.model
      .findByIdAndDelete(id, { session: options.session || null })
      .exec();

    if (!deletedDocument) {
      throw new AppError(
        `${this.resourceName} not found with ID: ${id}`,
        404,
        `${this.resourceName.toUpperCase()}_NOT_FOUND`
      );
    }

    return deletedDocument;
  }

  /**
   * Soft Deletes a document by setting isDeleted: true and deletedAt timestamp.
   * Preserves database integrity for compliance and audit trails.
   * @param {string} id
   * @param {Object} [options={}] - { session, deletedBy }
   * @returns {Promise<Object>} Soft-deleted document
   */
  async softDeleteById(id, options = {}) {
    return this.updateById(
      id,
      {
        $set: {
          isDeleted: true,
          deletedAt: new Date(),
          ...(options.deletedBy && { deletedBy: options.deletedBy }),
        },
      },
      options
    );
  }
}

module.exports = BaseService;
```

---

## 4. Real-World Subclassing Example (`OrderService.js`)

Here is how a real enterprise domain service extends `BaseService` without writing a single line of redundant CRUD code:

```javascript
/**
 * @file orders.service.js
 * @description Demonstrates clean domain service architecture extending BaseService.
 */

const BaseService = require('../shared/BaseService');
const Order = require('./orders.model');
const { ORDER_STATUS, CANCELLATION_REASONS } = require('./orders.constants');
const { AppError } = require('../shared/AppError');

class OrderService extends BaseService {
  constructor() {
    // Inject Mongoose Model and Human-Readable Resource Name
    super(Order, 'Order');
  }

  /**
   * Domain-Specific Workflow: Place Order within an ACID Transaction
   */
  async placeOrder(customerId, orderDTO, session) {
    this.assertValidId(customerId);

    // 1. Prepare domain payload
    const orderPayload = {
      customer: customerId,
      items: orderDTO.items,
      totalAmountPaise: orderDTO.totalAmountPaise,
      status: ORDER_STATUS.ORDER_PLACED,
    };

    // 2. BaseService create handles atomic session insertion automatically
    const newOrder = await this.create(orderPayload, { session });

    return newOrder;
  }

  /**
   * Domain-Specific Workflow: Cancel Order with FSM Guardrails
   */
  async cancelOrder(orderId, customerId, reasonCode, session) {
    // 1. BaseService findById automatically asserts ID format & throws 404 if missing
    const order = await this.findById(orderId, { session });

    // 2. Pure Domain Business Rule: Ownership check
    if (String(order.customer) !== String(customerId)) {
      throw new AppError('You are not authorized to cancel this order', 403, 'FORBIDDEN_ACCESS');
    }

    // 3. FSM Transition Verification
    if (order.status !== ORDER_STATUS.ORDER_PLACED) {
      throw new AppError(
        `Order cannot be cancelled in "${order.status}" status`,
        409,
        'INVALID_STATE_TRANSITION'
      );
    }

    // 4. BaseService updateById safely sets the state with runValidators: true
    return this.updateById(
      orderId,
      {
        $set: {
          status: ORDER_STATUS.CANCELLED,
          cancellationReason: reasonCode,
          cancelledAt: new Date(),
        },
      },
      { session }
    );
  }

  /**
   * Domain-Specific Query: List Active Orders with Clamped Pagination
   */
  async getActiveCustomerOrders(customerId, paginationParams) {
    this.assertValidId(customerId);

    const filter = {
      customer: customerId,
      status: { $in: [ORDER_STATUS.ORDER_PLACED, ORDER_STATUS.ORDER_CONFIRMED] },
      isDeleted: false,
    };

    // BaseService handles skip, limits, concurrent count, and metadata envelope
    return this.findPaginated(filter, paginationParams);
  }
}

module.exports = new OrderService();
```

---

## 5. Fatal Anti-Patterns & Common Traps

| Anti-Pattern | Why It Breaks Production | The Solution |
| :--- | :--- | :--- |
| **Bypassing `assertValidId`** | Malformed IDs crash Node.js with raw `CastError` (HTTP 500) and expose internal stack traces. | Call `this.assertValidId(id)` as the very first line of any ID-based method. |
| **Omitting `runValidators: true`** | `findByIdAndUpdate` skips schema validation by default, allowing invalid enums or corrupt data into the DB. | Always enforce `{ runValidators: true }` in `updateById`. |
| **Using `.find()` without limits** | A growing table loads hundreds of megabytes of JSON into RAM, causing immediate server OOM crashes. | Use `this.findPaginated()` with max limit clamping (max 100). |
| **Leaking `req` / `res` into BaseService** | Couples data access to HTTP transport, destroying testability and reusability in background workers/cron jobs. | Pass plain DTOs and options objects; never pass Express `req`. |
| **Using `findById` just for existence** | Serializes the entire document, fills the buffer, and wastes DB network bandwidth. | Use `this.exists({ _id: id })` which performs an index-only lookup. |

---

## 6. Pre-Flight Production Checklist

Before deploying any feature service that extends `BaseService`, verify:

- [ ] **Constructor Contract:** Does the subclass call `super(Model, 'EntityName')`?
- [ ] **Zero Raw CastErrors:** Are all incoming IDs validated with `this.assertValidId(id)`?
- [ ] **Deterministic 404s:** Does `findById` throw a semantic `{RESOURCE}_NOT_FOUND` error?
- [ ] **ACID Session Forwarding:** Does every mutating method forward `{ session }` to `BaseService`?
- [ ] **Safe Limit Clamping:** Are all listing endpoints routed through `findPaginated` with an upper bound?
- [ ] **Strict Schema Validation:** Are all update operations running with `{ runValidators: true }`?
- [ ] **Audit Trail Compliance:** Is sensitive financial/user data using `softDeleteById` rather than physical deletion?
- [ ] **Lean Performance:** Are read-only listing queries executing with `.lean()` enabled?

---

Ab aap seedhe is message ke niche **Copy** icon dabakar isse copy kar sakte ho! 

Aur jab aap bolenge, hum turant **Order Module** ke agle step (`orders.validation.js`) par aage badhenge!
