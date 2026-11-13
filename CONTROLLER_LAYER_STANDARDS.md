# 🎮 Backend Engineering Standards: The Senior Developer's Guide to Controller Layer Architecture

> **Author / Reference:** MDP Engineering Architecture  
> **Target Environment:** Node.js, Express 5, RESTful APIs, Clean Architecture  
> **Purpose:** Principles and architectural laws for building razor-thin, secure, and standardized HTTP transport controllers.

---

## 📑 Table of Contents
1. [The Role of the Controller Layer](#the-role-of-the-controller-layer)
2. [The 7 Golden Rules of Controller Design](#the-7-golden-rules-of-controller-design)
3. [Standard Production Controller Template](#standard-production-controller-template)
4. [The 60-Second Pre-Commit Controller Checklist](#the-60-second-pre-commit-controller-checklist)

---

## The Role of the Controller Layer

In enterprise backend systems, the Controller is an **HTTP Transport Adapter**:
- **What it DOES:** Translates incoming HTTP requests (headers, params, JWT context) into domain arguments, invokes the Service method, and wraps the result in a standardized HTTP response envelope.
- **What it NEVER DOES:** It contains **ZERO database queries**, **ZERO business logic**, and **ZERO mathematical calculations**.

```text
HTTP Request ──▶ Route Validation ──▶ Controller (Adapter) ──▶ Service (Brain) ──▶ Database
                                           │
HTTP Response ◀── Uniform Envelope ◀───────┘
```

---

# The 7 Golden Rules of Controller Design

---

### Rule 1: The "Thin Controller" Law (Zero Database Queries)
A controller must **NEVER** import Mongoose models or execute database queries (`Order.findById()`, `User.findOne()`).

> [!WARNING]
> If a database query or business `if-else` condition is written inside a controller, that logic is trapped inside the HTTP transport. A background worker, cron job, or test runner can never execute it.

```javascript
// ❌ BAD: Fat Controller (Database query and business check trapped in controller)
async cancelOrder(req, res, next) {
  try {
    const order = await Order.findById(req.params.id); // 🛑 DB Query in controller!
    if (order.status === 'PACKING') {                  // 🛑 Business logic in controller!
      return res.status(400).json({ error: 'Too late to cancel' });
    }
    order.status = 'CANCELLED';
    await order.save();
    return res.json(order);
  } catch (err) {
    next(err);
  }
}

// ✅ SENIOR STANDARD: Thin Adapter (Delegates 100% to service)
async cancelOrder(req, res, next) {
  try {
    const { id } = req.validated.params;
    const { reason } = req.validated.body;
    const customerId = req.customer.id;
    const actor = getActor(req);

    const result = await orderService.cancelOrder(id, customerId, reason, actor);
    return sendSuccess(res, result);
  } catch (err) {
    next(err);
  }
}
```

---

### Rule 2: The Trusted Input Law (Read `req.validated` Exclusively)
Never read raw, un-sanitized data from `req.body`, `req.params`, or `req.query` if your route has validation middleware (e.g., Zod).

- Always read from **`req.validated`**.
- **Why:** Guarantees that prototype pollution keys are stripped, string trimming is applied, type coercion (strings to numbers) is completed, and default values are populated before reaching your service.

```javascript
// ❌ BAD: Vulnerable to unstripped payload properties
const { quantity } = req.body;

// ✅ SENIOR STANDARD: Sanitized, validated, and type-checked
const { quantity } = req.validated.body;
const { orderId } = req.validated.params;
```

---

### Rule 3: The Identity Extraction Law (Prevent IDOR Vulnerabilities)
Never trust a `customerId` or `userId` passed in the HTTP request body or query params for operations modifying private customer data.

Always extract the user's identity strictly from the **authenticated JWT token context** (`req.customer.id` or `req.user.id`).

> [!IMPORTANT]
> **IDOR Vulnerability (Insecure Direct Object Reference):** If your controller reads `const customerId = req.body.customerId;`, a malicious User A can pass User B's `customerId` in Postman and alter User B's cart or cancel their orders!

```javascript
// ❌ BAD: User A can spoof User B's ID
async addItem(req, res, next) {
  const { customerId, medicineId } = req.body; // 🚨 Security Vulnerability!
  await cartService.addItem(customerId, medicineId);
}

// ✅ SENIOR STANDARD: Identity securely extracted from verified JWT payload
async addItem(req, res, next) {
  try {
    const customerId = req.customer.id; // 🛡️ Tamper-proof from auth middleware
    const { medicineId, quantity } = req.validated.body;
    const result = await cartService.addItem(customerId, { medicineId, quantity });
    return sendCreated(res, result);
  } catch (err) {
    next(err);
  }
}
```

---

### Rule 4: Uniform Response Envelopes (Never Call `res.json()` Manually)
Controllers must never invent custom JSON response shapes on the fly. Always use centralized response helpers (`sendSuccess`, `sendCreated`, `sendPaginated`).

- **Single Record / Action Result:**
  ```json
  { "success": true, "data": { ... } }
  ```
- **Paginated List:**
  ```json
  {
    "success": true,
    "data": [ ... ],
    "meta": { "page": 1, "limit": 10, "total": 45, "totalPages": 5 }
  }
  ```

```javascript
// ❌ BAD: Inconsistent, custom response shape
return res.status(200).json({ status: 'ok', payload: order, timestamp: Date.now() });

// ✅ SENIOR STANDARD: Standardized envelope utility
return sendSuccess(res, order);
```

---

### Rule 5: Strict RESTful HTTP Semantics
Always use the correct HTTP verb and status code that communicates the precise architectural outcome to client apps (Flutter / React):

| HTTP Method | Scenario | Status Code | Helper Function |
| :--- | :--- | :--- | :--- |
| **`GET`** | Resource fetched / listed | `200 OK` | `sendSuccess(res, data)` |
| **`POST`** | New resource created | `201 Created` | `sendCreated(res, data)` |
| **`PATCH` / `PUT`** | Resource updated / state moved | `200 OK` | `sendSuccess(res, data)` |
| **`DELETE`** | Action completed with data | `200 OK` | `sendSuccess(res, data)` |
| **`DELETE`** | Action completed with empty body | `204 No Content` | `sendNoContent(res)` |

> [!WARNING]
> Never return `200 OK` with an error message in the body (`{ success: false, error: 'Failed' }`). If an operation fails, throw an `ApiError` and let the central error handler send the appropriate `4xx` or `5xx` status.

---

### Rule 6: The Global Exception Delegation Rule (`next(err)`)
Controller methods must always wrap their execution in a `try...catch` block, and the `catch` block must do **one thing and one thing only:** pass the error to `next(err)`.

- Never catch an error and swallow it.
- Never write custom `res.status(500).json(...)` inside individual controllers.

```javascript
// ❌ BAD: Leaking internal error details or swallowing errors
catch (err) {
  console.log(err);
  return res.status(500).json({ error: err.message }); // 🚨 May leak DB paths or passwords!
}

// ✅ SENIOR STANDARD: Hands over to Express 5 central error handler
catch (err) {
  next(err); // 🛡️ Central error handler logs requestId, scrubs stack traces in prod, and formats error envelope
}
```

---

### Rule 7: The Actor Pattern for Auditing & RBAC
In enterprise platforms (Admin dashboards, Pharmacy panels), sensitive actions need to record **who** performed the action and in **what role**.

Controllers should extract a standardized `actor` object and pass it down to services:

```javascript
function getActor(req) {
  return {
    type: req.user?.type || 'CUSTOMER',
    userId: req.user?.id || req.customer?.id,
    name: req.user?.name || req.customer?.fullName,
    role: req.user?.role || 'CUSTOMER',
  };
}

// Pass actor cleanly into service:
const result = await orderService.approveOrder(orderId, getActor(req));
```

---

# Standard Production Controller Template

```javascript
/**
 * Order Controller
 * ----------------
 * HTTP transport layer for Order lifecycle management.
 */

import { sendCreated, sendSuccess, sendPaginated } from '../../utils/response.js';
import { parseSort } from '../../utils/pagination.js';
import { orderService } from './orders.service.js';

class OrderController {
  /**
   * POST /api/v1/orders
   * Place an order from active locked cart.
   */
  async createOrder(req, res, next) {
    try {
      const customerId = req.customer.id;
      const { paymentMethod } = req.validated.body;

      const result = await orderService.createOrder(customerId, { paymentMethod });
      return sendCreated(res, result);
    } catch (err) {
      next(err);
    }
  }

  /**
   * GET /api/v1/orders/:id
   * Fetch detailed order record with timeline.
   */
  async getOrderById(req, res, next) {
    try {
      const { id } = req.validated.params;
      const customerId = req.customer.id;

      const result = await orderService.getOrderById(id, customerId);
      return sendSuccess(res, result);
    } catch (err) {
      next(err);
    }
  }

  /**
   * GET /api/v1/orders
   * Paginated list of customer orders.
   */
  async listOrders(req, res, next) {
    try {
      const customerId = req.customer.id;
      const { page, limit, status } = req.validated.query;
      const sort = parseSort(req.validated.query.sort, { createdAt: -1 });

      const { data, meta } = await orderService.listCustomerOrders(
        customerId,
        { status },
        { page, limit, sort }
      );

      return sendPaginated(res, data, meta);
    } catch (err) {
      next(err);
    }
  }

  /**
   * POST /api/v1/orders/:id/cancel
   * Customer cancels an order before packing.
   */
  async cancelOrder(req, res, next) {
    try {
      const { id } = req.validated.params;
      const { reason } = req.validated.body;
      const customerId = req.customer.id;

      const result = await orderService.cancelOrder(id, customerId, reason);
      return sendSuccess(res, result);
    } catch (err) {
      next(err);
    }
  }
}

export const orderController = new OrderController();
```

---

# The 60-Second Pre-Commit Controller Checklist

Before committing a Controller file, audit your code against this checklist:

- [ ] **Zero Database Access:** Are all Mongoose models completely absent from controller imports?
- [ ] **Thin Methods:** Are all business logic checks (`if/else` on domain state) delegated to the service?
- [ ] **Trusted Validation:** Are all inputs read from `req.validated.body`, `req.validated.params`, or `req.validated.query`?
- [ ] **No IDOR Vulnerability:** Is user identity extracted strictly from `req.customer.id` or `req.user.id`?
- [ ] **Standard Envelopes:** Are all responses returned using `sendSuccess()`, `sendCreated()`, or `sendPaginated()`?
- [ ] **Global Error Delegation:** Does every single `catch` block strictly call `next(err)`?
- [ ] **HTTP Status Code Precision:** Does resource creation return `201 Created` via `sendCreated`?
