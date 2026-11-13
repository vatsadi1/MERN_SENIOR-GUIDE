# 🛡️ Backend Engineering Standards: The Senior Developer's Guide to Schemas & Constants

> **Author / Reference:** MDP Engineering Architecture  
> **Target Environment:** Node.js, Express, MongoDB (Mongoose), FinTech & E-Commerce  
> **Purpose:** Battle-tested architectural patterns, invariants, and design rules to build production-grade, zero-debt backends that scale without data corruption.

---

## 📑 Table of Contents
1. [Part I: The 7 Golden Rules of Schema Design](#part-i-the-7-golden-rules-of-schema-design)
2. [Part II: The 7 Golden Rules of Constants & Invariants](#part-ii-the-7-golden-rules-of-constants--invariants)
3. [Part III: Standard Production Templates](#part-iii-standard-production-templates)
4. [Part IV: The 60-Second Pre-Commit Architecture Checklist](#part-iv-the-60-second-pre-commit-architecture-checklist)

---

# Part I: The 7 Golden Rules of Schema Design

### Rule 1: The "Snapshot vs Reference" Law (Historical Immutability)
In transaction systems (Orders, Invoices, Payments), **never rely on dynamic foreign references for completed actions**.
- **Reference (`ObjectId` + `ref`):** Use only for live, mutable relationships (`customer`, `store`, `driver`).
- **Snapshot (Embedded Subdocument):** Freeze copies of data that must never change retroactively.

> [!WARNING]
> If a store changes its trade name/address or a medicine increases in MRP next month, past customer invoices must **NOT** change. Freeze address, price, and item snapshots inside the order document.

```javascript
// ❌ BAD: Storing only reference (If medicine price changes, old order bill changes!)
items: [{ medicineId: { type: ObjectId, ref: 'Medicine' }, quantity: 2 }]

// ✅ SENIOR STANDARD: Snapshotting details at time of purchase
items: [{
  medicine: { type: ObjectId, ref: 'Medicine', required: true },
  name: { type: String, required: true },
  quantity: { type: Number, required: true },
  pricePaise: { type: Number, required: true }, // Frozen price
  mrpPaise: { type: Number, required: true },
  lineTotalPaise: { type: Number, required: true }
}]
```

---

### Rule 2: The "1 : Few vs 1 : Many" Rule (Embedding vs Referencing)
- **Embed as Subdocuments (`[subSchema]`):** When the array is strictly **bounded** (e.g., Cart items $\le 30$, weekly operating hours $= 7$, status transition history $\le 20$).
- **Separate Collection:** When the array can grow **unboundedly** (e.g., store reviews, customer orders, application audit logs).

> [!NOTE]
> MongoDB has a hard limit of **16MB per document**. Unbounded arrays inside a single document will eventually cause database write crashes.

---

### Rule 3: The Zero-Decimal FinTech Rule (Integer Paise Standard)
Never store currency amounts as `Float` or `Decimal` (e.g., `23.50`). Floating-point arithmetic in software leads to IEEE 754 precision errors ($0.1 + 0.2 = 0.30000000000000004$).

- **Golden Standard:** Always store currency in the smallest fractional unit (India: **Integer Paise**, US: **Integer Cents**).
- **Naming Rule:** Embed the unit directly in the schema key name (`pricePaise`, `totalPayablePaise`).

```javascript
// FinTech Integer Validator
const wholeNumber = {
  validator: Number.isInteger,
  message: '{PATH} must be a whole number of paise (no decimals allowed)',
};

// Field definition
totalPayablePaise: {
  type: Number,
  required: true,
  min: [0, 'Total payable cannot be negative'],
  validate: wholeNumber,
}
```

---

### Rule 4: Security & Privacy by Default (`select: false`)
Sensitive and private data must be protected at the database schema edge so accidental controller returns never leak them to the public API.

Always apply `select: false` to:
- Passwords, OTPs, hashes, and API secrets.
- Full bank account numbers (store only masked `last4` for display).
- Internal verification tokens.

```javascript
// ✅ Protected field
deliveryOtp: {
  type: String,
  required: true,
  select: false, // Omitted from Order.find() by default
}
```

---

### Rule 5: Concurrency & The "Double Click" Guard (Idempotency)
Always design schemas anticipating network retries, race conditions, and rapid double-clicks from users.

- Identify the **Natural Unique Business Key** and enforce `{ unique: true }`.
- *Order Creation:* `cart: { type: ObjectId, unique: true }` $\rightarrow$ Prevents two orders from being generated from the same cart session.
- *Payment Webhook:* `gatewayPaymentId: { type: String, unique: true, sparse: true }`.

---

### Rule 6: Query-Driven Compound Indexing (The ESR Rule)
Do not create random single-field indexes. Create compound indexes based on how frontend screens and background workers actually query the data using the **ESR Rule**:
1. **E (Equality):** Exact matches (`store: id`)
2. **S (Sort):** Sorting order (`createdAt: -1`)
3. **R (Range):** Range queries and status filters (`status: { $in: [...] }`)

```javascript
// Customer app order history: Equality on customer, Sort on date
orderSchema.index({ customer: 1, createdAt: -1 });

// Chemist live dashboard: Equality on store, Equality on status, Sort on date
orderSchema.index({ store: 1, status: 1, createdAt: -1 });

// Background SLA auto-cancel worker: Equality on status, Range on deadline
orderSchema.index({ status: 1, storeAcceptanceDeadlineAt: 1 });
```

---

### Rule 7: Schema Hygiene & Defensive Typing
1. **`trim: true`:** Mandatory on all String fields (prevents `" user@example.com "`).
2. **`uppercase: true` / `lowercase: true`:** Enforce on codes, GSTIN, PAN, and emails.
3. **`timestamps: true`:** Every schema must track `createdAt` and `updatedAt`.
4. **`{ _id: false }`:** Add to embedded sub-schemas that are never queried or deleted independently to save BSON memory.

---

# Part II: The 7 Golden Rules of Constants & Invariants

### Rule 1: The "Unit in the Name" Law (Self-Documenting Units)
Never declare raw numbers without specifying their measurement unit in the variable identifier.

```javascript
// ❌ BAD: Ambiguous units
SLA: 15,
MIN_ORDER: 200,
MAX_DISTANCE: 4,

// ✅ SENIOR STANDARD: Units embedded in identifier
STORE_ACCEPTANCE_SLA_MINUTES: 15,
MIN_ORDER_PAISE: 20000,
MAX_DISTANCE_METERS: 4000,
MAX_FILE_SIZE_BYTES: 10 * 1024 * 1024,
DEFAULT_SHELF_LIFE_DAYS: 30,
```

---

### Rule 2: Runtime Immutability via `Object.freeze()`
JavaScript objects are mutable by reference. A single accidental assignment can mutate global state across the entire Node.js event loop.
- **Rule:** Always wrap constant dictionaries and arrays with `Object.freeze()`.

```javascript
export const PAYMENT_METHOD = Object.freeze({
  ONLINE: 'ONLINE',
  COD: 'COD',
});
```

---

### Rule 3: Key-Value Symmetry (UPPER_SNAKE_CASE)
Keep keys and string values strictly identical. This guarantees seamless integration with Mongoose enums, Zod validation, and Swagger schemas:
```javascript
export const ORDER_STATUS = Object.freeze({
  ORDER_PLACED: 'ORDER_PLACED',
  STORE_ACCEPTED: 'STORE_ACCEPTED',
  DELIVERED: 'DELIVERED',
});
```

---

### Rule 4: The Finite State Machine (FSM) Matrix
Never just list status names. Define the **legal transition graph** in the constants file. This prevents illegal workflow skips in the service layer:

```javascript
export const ORDER_STATUS_TRANSITIONS = Object.freeze({
  [ORDER_STATUS.ORDER_PLACED]: Object.freeze([
    ORDER_STATUS.STORE_ACCEPTED,
    ORDER_STATUS.STORE_REJECTED,
    ORDER_STATUS.CANCELLED,
  ]),
  [ORDER_STATUS.STORE_ACCEPTED]: Object.freeze([
    ORDER_STATUS.PACKING,
    ORDER_STATUS.CANCELLED,
  ]),
  [ORDER_STATUS.DELIVERED]: Object.freeze([]), // Terminal state
});
```

---

### Rule 5: Specialized Subset Arrays (1-Line Service Guards)
Group statuses into business action sets so services can evaluate permissions with a clean 1-line check:

```javascript
// States where customer is legally allowed to cancel
export const CANCELLABLE_ORDER_STATUSES = Object.freeze([
  ORDER_STATUS.PAYMENT_PENDING,
  ORDER_STATUS.ORDER_PLACED,
  ORDER_STATUS.STORE_ACCEPTED,
]);

// Service layer implementation:
// if (!CANCELLABLE_ORDER_STATUSES.includes(order.status)) throw ApiError.badRequest('Cannot cancel order at this stage');
```

---

### Rule 6: Structured Reason Codes (Zero Free-Text on Operations)
Critical operational events (rejections, cancellations, refund requests) must never store arbitrary strings. Always provide structured, categorical taxonomies:

```javascript
export const ORDER_CANCELLATION_REASON = Object.freeze({
  CUSTOMER_CHANGED_MIND: 'CUSTOMER_CHANGED_MIND',
  ORDERED_BY_MISTAKE: 'ORDERED_BY_MISTAKE',
  DELIVERY_DELAYED: 'DELIVERY_DELAYED',
  PAYMENT_TIMEOUT: 'PAYMENT_TIMEOUT',
  OTHER: 'OTHER',
});
```

---

### Rule 7: Zero Dependency Law (Pure Vanilla JavaScript)
Constants files must **NEVER** import external libraries, Express, Mongoose, or database models.
- **Why:** `constants.js` is the base layer of the dependency tree. Keeping it pure eliminates circular dependencies and allows it to be shared directly with frontend clients (React/Flutter) or test suites.

---

# Part III: Standard Production Templates

### Template A: `*.constants.js`
```javascript
/**
 * Module Constants & State Machine
 */

// 1. Status Enum
export const MODULE_STATUS = Object.freeze({
  ACTIVE: 'ACTIVE',
  SUSPENDED: 'SUSPENDED',
});

// 2. FSM Transitions
export const MODULE_TRANSITIONS = Object.freeze({
  [MODULE_STATUS.ACTIVE]: Object.freeze([MODULE_STATUS.SUSPENDED]),
  [MODULE_STATUS.SUSPENDED]: Object.freeze([MODULE_STATUS.ACTIVE]),
});

// 3. Operational Subsets
export const EDITABLE_STATUSES = Object.freeze([MODULE_STATUS.ACTIVE]);

// 4. Categorical Taxonomies
export const REJECTION_REASONS = Object.freeze({
  INVALID_DOCUMENT: 'INVALID_DOCUMENT',
  OTHER: 'OTHER',
});

// 5. System Configurations
export const MODULE_CONFIG = Object.freeze({
  CODE_PREFIX: 'MOD',
  TTL_MINUTES: 15,
  MAX_LIMIT: 50,
});
```

---

### Template B: `*.model.js`
```javascript
import { Schema, model } from 'mongoose';
import { MODULE_STATUS } from './module.constants.js';

const wholeNumber = {
  validator: Number.isInteger,
  message: '{PATH} must be a whole number',
};

// 1. Embedded Sub-Schema (Bounded & Frozen)
const lineItemSchema = new Schema(
  {
    itemId: { type: Schema.Types.ObjectId, ref: 'Item', required: true },
    pricePaise: { type: Number, required: true, validate: wholeNumber },
  },
  { _id: false }
);

// 2. Main Schema
const moduleSchema = new Schema(
  {
    code: { type: String, required: true, unique: true, uppercase: true, trim: true },
    status: { type: String, enum: Object.values(MODULE_STATUS), default: MODULE_STATUS.ACTIVE, index: true },
    items: { type: [lineItemSchema], default: [] },
    secretToken: { type: String, select: false },
  },
  { timestamps: true }
);

// 3. Compound Indexes
moduleSchema.index({ status: 1, createdAt: -1 });

export const Module = model('Module', moduleSchema);
```

---

# Part IV: The 60-Second Pre-Commit Architecture Checklist

Before submitting a Pull Request with a new Model or Constants file, verify:

- [ ] **FinTech Check:** Are all financial amounts in integer Paise with `Number.isInteger` validation?
- [ ] **Snapshot Check:** Are historical addresses, item names, and line prices frozen inside the document?
- [ ] **Security Check:** Are secrets, OTPs, and private tokens tagged with `select: false`?
- [ ] **Idempotency Check:** Is there a unique constraint preventing duplicate transactions?
- [ ] **Immutability Check:** Is every constant object and array wrapped in `Object.freeze()`?
- [ ] **Unit Suffix Check:** Do all constant configuration values have units in their names (`_PAISE`, `_MINUTES`, `_METERS`, `_BYTES`)?
- [ ] **Zero-Dependency Check:** Is the `*.constants.js` file 100% pure JavaScript with zero database/framework imports?
