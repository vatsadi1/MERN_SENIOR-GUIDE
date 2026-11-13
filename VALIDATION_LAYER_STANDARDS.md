# 🛡️ Backend Engineering Standards: The Senior Developer's Guide to Validation Layer Architecture (Zod)

> **Author / Reference:** MDP Engineering Architecture  
> **Target Environment:** Node.js, Express 5, Zod Schema Validation  
> **Purpose:** Architectural laws for building strict, sanitizing, type-coercing, and attack-proof input validation at the HTTP network edge.

---

## 📑 Table of Contents
1. [The Role of the Validation Layer](#the-role-of-the-validation-layer)
2. [The 7 Golden Rules of Validation Design](#the-7-golden-rules-of-validation-design)
3. [Standard Production Validation Template](#standard-production-validation-template)
4. [The 60-Second Pre-Commit Validation Checklist](#the-60-second-pre-commit-validation-checklist)

---

## The Role of the Validation Layer

In enterprise backends, the Validation Layer is the **First Line of Defense**:
- **Fail Fast at the Network Edge:** Invalid payloads are rejected in $\approx 0.1\text{ms}$ of CPU time before hitting controllers, services, or databases.
- **Sanitizer & Normalizer:** It doesn't just inspect data; it cleans it (`trim`, `toLowerCase`, `toUpperCase`, number coercion).
- **Type Safety Guarantee:** Strips unknown/prototype pollution fields, ensuring downstream services receive 100% clean, trusted DTOs.

---

# The 7 Golden Rules of Validation Design

---

### Rule 1: The Three-Target Separation Law (Params, Query, Body)
Express receives data from three distinct sources: URL parameters (`req.params`), URL query strings (`req.query`), and JSON payloads (`req.body`).
- Never mix them into one giant schema.
- Always author separate schemas named by their target:
  - `*ParamSchema` (for route parameters like `:id`, `:storeId`)
  - `*QuerySchema` (for listing, filtering, pagination)
  - `*BodySchema` (for create, update, action payloads)

```javascript
// ✅ SENIOR STANDARD: Explicit target separation
export const orderIdParamSchema = z.object({
  id: z.string().regex(/^[0-9a-fA-F]{24}$/, 'Invalid Order ObjectId format'),
});

export const listOrdersQuerySchema = z.object({
  status: z.enum(Object.values(ORDER_STATUS)).optional(),
});

export const createOrderBodySchema = z.object({
  paymentMethod: z.enum(Object.values(PAYMENT_METHOD)),
});
```

---

### Rule 2: The Normalization Law (`trim`, `toLowerCase`, `toUpperCase`)
Validation is not merely type checking; it is **Data Hygiene**.
- **Every string** must be `.trim()` (eliminates leading/trailing spaces).
- **Emails** must be `.trim().toLowerCase()`.
- **Codes, GSTIN, PAN, and Coupons** must be `.trim().toUpperCase()`.

> [!WARNING]
> Without `.trim()`, a user typing `" WELCOME50 "` with accidental spaces will fail coupon matching in the service layer, leading to bad customer UX.

```javascript
// ❌ BAD: Raw string validation allows leading/trailing whitespace
couponCode: z.string().min(3),

// ✅ SENIOR STANDARD: Sanitized and auto-cased
couponCode: z
  .string()
  .trim()
  .toUpperCase()
  .min(3, 'Coupon code must be at least 3 characters')
  .max(20, 'Coupon code cannot exceed 20 characters'),
```

---

### Rule 3: Single Source of Truth (Zero Duplicate Regex & Enums)
Never hardcode string enums or regex patterns inside `*.validation.js` files.
- Always import Enums from `*.constants.js`.
- Always import Regex patterns from `constants/patterns.js`.

```javascript
// ❌ BAD: Hardcoding duplicate status strings (If a status is added, this breaks!)
status: z.enum(['ORDER_PLACED', 'STORE_ACCEPTED', 'DELIVERED']),

// ✅ SENIOR STANDARD: Linked directly to frozen domain constants
import { ORDER_STATUS } from './orders.constants.js';
import { PATTERNS } from '../../constants/patterns.js';

status: z.enum(Object.values(ORDER_STATUS)),
mobile: z.string().trim().regex(PATTERNS.MOBILE, 'Invalid 10-digit Indian mobile number'),
```

---

### Rule 4: Query String Type Coercion (`z.coerce`)
In HTTP GET requests, **all query parameters arrive as Strings** (`"?page=2&limit=10"` $\rightarrow$ `req.query.page = "2"`).
- Using `z.number()` on `req.query` will throw validation errors because `"2"` is a string, not a number!
- Always use `z.coerce.number()` or reusable pagination helpers for query parameters.

```javascript
// ❌ BAD: Fails on GET requests because req.query values are strings
page: z.number().int().min(1),

// ✅ SENIOR STANDARD: Auto-coerces string "2" to integer 2 with default fallback
page: z.coerce
  .number()
  .int('Page must be an integer')
  .min(1, 'Page must be at least 1')
  .default(1),
```

---

### Rule 5: Cross-Field Logic via `.refine()`
When validation rules depend on the relationship between multiple fields, use Zod's `.refine()` or `.superRefine()`. Always specify the exact `path` so the error highlights the culprit field.

- *Examples:* `storePricePaise <= mrpPaise`, `confirmPassword === password`, `endDate >= startDate`.

```javascript
// ✅ SENIOR STANDARD: Cross-field business rule enforcement
export const singleInventorySchema = z
  .object({
    mrpPaise: z.number().int().min(0),
    storePricePaise: z.number().int().min(0),
  })
  .refine((data) => data.storePricePaise <= data.mrpPaise, {
    message: 'Store offer price cannot exceed printed MRP',
    path: ['storePricePaise'], // 🎯 Error points directly to this field in API response
  });
```

---

### Rule 6: Defensive Upper Bounds (DDoS & Buffer Overflow Shield)
Never allow open-ended inputs. Every input must have strict minimum and maximum boundaries.

- **Strings:** Every string must have a `.max(N)` to prevent memory exhaustion from megabyte-long strings.
- **Numbers:** Quantities, percentages, and pagination limits must have a `.max(N)`.

```javascript
// ❌ BAD: Vulnerable to 10MB string injection attacks
addressLine1: z.string(),
quantity: z.number().min(1),

// ✅ SENIOR STANDARD: Strictly bounded
addressLine1: z.string().trim().min(3).max(200, 'Address cannot exceed 200 characters'),
quantity: z.number().int().min(1).max(50, 'Cannot order more than 50 units per medicine'),
```

---

### Rule 7: Client-Safe, Human-Readable Error Messages
Validation error messages are directly surfaced to client applications (Flutter / React) to inform real human users why their action was rejected.

Never leave cryptic default errors like `"Invalid string"` or `"Failed refinement"`. Provide clear, actionable instructions.

```javascript
// ❌ BAD: Confusing default messages
pinCode: z.string().regex(/^\d{6}$/),

// ✅ SENIOR STANDARD: Actionable human guidance
pinCode: z
  .string()
  .trim()
  .regex(/^[1-9][0-9]{5}$/, 'Please enter a valid 6-digit Indian PIN code (cannot start with 0)'),
```

---

# Standard Production Validation Template

```javascript
/**
 * Order Validation Schemas
 * ------------------------
 * Frontline HTTP validation schemas for the Order module.
 */

import { z } from 'zod';
import { ORDER_STATUS, PAYMENT_METHOD, ORDER_CANCELLATION_REASON } from './orders.constants.js';

// ---------------------------------------------------------------------------------------------
// 1. Param Schemas
// ---------------------------------------------------------------------------------------------
export const orderIdParamSchema = z.object({
  id: z
    .string()
    .trim()
    .regex(/^[0-9a-fA-F]{24}$/, 'Invalid Order ObjectId format'),
});

// ---------------------------------------------------------------------------------------------
// 2. Query Schemas (Using z.coerce for string-to-number transformation)
// ---------------------------------------------------------------------------------------------
export const listOrdersQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  status: z.enum(Object.values(ORDER_STATUS)).optional(),
});

// ---------------------------------------------------------------------------------------------
// 3. Body Schemas
// ---------------------------------------------------------------------------------------------
export const createOrderBodySchema = z.object({
  paymentMethod: z.enum(Object.values(PAYMENT_METHOD), {
    errorMap: () => ({ message: 'Payment method must be ONLINE or COD' }),
  }),
});

export const cancelOrderBodySchema = z.object({
  reason: z.enum(Object.values(ORDER_CANCELLATION_REASON), {
    errorMap: () => ({ message: 'A valid cancellation reason must be selected' }),
  }),
  note: z.string().trim().max(300, 'Note cannot exceed 300 characters').optional(),
});
```

---

# The 60-Second Pre-Commit Validation Checklist

Before committing a Validation file, audit your schemas against this checklist:

- [ ] **Target Separation:** Are `params`, `query`, and `body` schemas authored as separate objects?
- [ ] **Hygiene Check:** Are all string fields sanitized with `.trim()`? Are codes uppercased and emails lowercased?
- [ ] **Zero Hardcoded Enums:** Are all enums imported directly from `*.constants.js` via `Object.values(...)`?
- [ ] **Query Coercion:** Do query string numbers use `z.coerce.number()` to avoid HTTP GET string type mismatch?
- [ ] **Bounded Limits:** Does every string have a `.max(N)` and every quantity have a sensible ceiling?
- [ ] **Cross-Field Refinement:** If two fields correlate (e.g. `price <= mrp`), is `.refine()` implemented with a target `path`?
- [ ] **Actionable Errors:** Are regex and boundary checks backed by clear, human-readable error messages?
