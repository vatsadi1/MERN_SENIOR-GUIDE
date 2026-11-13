# 🛣️ Backend Engineering Standards: The Senior Developer's Guide to Route Layer Architecture

> **Author / Reference:** MDP Engineering Architecture  
> **Target Environment:** Express.js 5, RESTful APIs, Modular Monolith  
> **Purpose:** Architectural laws for designing clean, collision-free, secure, and declarative API routing pipelines.

---

## 📑 Table of Contents
1. [The Role of the Route Layer](#the-role-of-the-route-layer)
2. [The 7 Golden Rules of Route Design](#the-7-golden-rules-of-route-design)
3. [Standard Production Route Template](#standard-production-route-template)
4. [The 60-Second Pre-Commit Route Checklist](#the-60-second-pre-commit-route-checklist)

---

## The Role of the Route Layer

In enterprise backends, Route files are the **Traffic Gateways**:
- **What it DOES:** Acts as a declarative pipeline binding:
  $$\text{HTTP Method + Path} \longrightarrow \text{Auth Guard} \longrightarrow \text{RBAC Permission} \longrightarrow \text{Input Validation} \longrightarrow \text{Controller}$$
- **What it NEVER DOES:** It contains **ZERO inline handler functions `(req, res) => {}`**, **ZERO database access**, and **ZERO business checks**.

---

# The 7 Golden Rules of Route Design

---

### Rule 1: The Pipeline Principle (Zero Anonymous Inline Callbacks)
Route files must be 100% declarative. Every endpoint definition must be a clean chain of middleware references terminating at a controller method.

> [!WARNING]
> Never write inline `async (req, res) => { ... }` functions inside route files. Anonymous inline callbacks cannot be unit-tested in isolation, clutter the router, and violate the Single Responsibility Principle.

```javascript
// ❌ BAD: Cluttered inline callback logic in route file
orderRouter.post('/cancel', async (req, res) => {
  const { id } = req.body;
  const order = await Order.findById(id); // 🚨 DB access inside route file!
  res.json({ success: true });
});

// ✅ SENIOR STANDARD: Declarative Middleware Pipeline
orderRouter.post(
  '/:id/cancel',
  authenticateCustomer,                        // 1. Identity Guard
  validate({
    params: orderIdParamSchema,               // 2. Input Validation (URL)
    body: cancelOrderSchema,                  // 3. Input Validation (Body)
  }),
  orderController.cancelOrder,                // 4. Controller Adapter
);
```

---

### Rule 2: The Route Collision Law (Static Routes BEFORE Dynamic Params)
Express evaluates routes **top-to-bottom in the order they are defined**.
- Always declare **Static Paths** (`/export`, `/bulk-upload`, `/statistics`, `/me`) **BEFORE Dynamic Parameter Paths** (`/:id`).

> [!IMPORTANT]
> If `GET /:id` is placed above `GET /export`, Express will match `/export` against `/:id` and treat `'export'` as the MongoDB ID! This leads to cryptic `400 INVALID_ID` errors in production.

```javascript
// ❌ BAD: Dynamic param intercepts the static route
orderRouter.get('/:id', orderController.getById);
orderRouter.get('/export', orderController.exportOrders); // 💥 Never reached! 'export' matched as :id!

// ✅ SENIOR STANDARD: Static routes declared first
orderRouter.get('/export', orderController.exportOrders); // 🛡️ Evaluated first
orderRouter.get('/:id', orderController.getById);        // Evaluated only if no static match
```

---

### Rule 3: The Top-Level Router Guard Law (`router.use(auth)`)
If all (or nearly all) endpoints in a module require the same security authentication (e.g. `/cart` or `/orders`), do **NOT** duplicate `authenticateCustomer` on every single route definition.

Mount the authentication middleware once at the top of the router.

```javascript
// ❌ BAD: Duplicating auth on 15 routes (Easy to forget on 1 route, causing security leaks)
orderRouter.get('/', authenticateCustomer, orderController.list);
orderRouter.post('/', authenticateCustomer, orderController.create);
orderRouter.get('/:id', authenticateCustomer, orderController.getById);

// ✅ SENIOR STANDARD: Module-level blanket protection
export const orderRouter = Router();

// Protect every route declared below with zero boilerplate
orderRouter.use(authenticateCustomer);

orderRouter.get('/', orderController.list);
orderRouter.post('/', orderController.create);
orderRouter.get('/:id', orderController.getById);
```

---

### Rule 4: The Edge Validation Guard (No Route Without Validation)
Every endpoint that accepts input (URL params, request body, or query strings) **MUST** pass through a validation middleware (e.g. Zod) before reaching the controller.

- If an ID is passed in URL: Validate `params: idParamSchema`.
- If pagination is passed: Validate `query: paginationQuerySchema`.
- If JSON payload is passed: Validate `body: payloadSchema`.

> [!TIP]
> **Fail Fast at the Network Edge:** Rejecting a bad request at the route layer takes $\approx 0.1\text{ms}$ of CPU time without touching controllers, services, or the database.

---

### Rule 5: RESTful Resource Naming Conventions
Follow enterprise URL design standards:

1. **Plural Nouns for Resources:**
   - `/api/v1/orders` (not `/api/v1/order`)
   - `/api/v1/medicines` (not `/api/v1/medicine`)
2. **Kebab-Case for Multi-Word Paths:**
   - `/api/v1/stores/:id/price-list` (not `/priceList` or `/price_list`)
   - `/api/v1/stores/bulk-upload`
3. **Sub-Resources Represent Relationships:**
   - `/api/v1/stores/:storeId/inventory` (Inventory belonging to a store)
   - `/api/v1/cart/items/:itemId` (Item belonging to active cart)
4. **Verbs ONLY for Non-CRUD State Transitions:**
   - State moves and actions are represented as action sub-paths:  
     `POST /api/v1/orders/:id/cancel`  
     `POST /api/v1/orders/:id/pack`  
     `POST /api/v1/cart/checkout`

---

### Rule 6: Child Router Isolation via `mergeParams: true`
When building nested resource routers (e.g. mounting inventory under stores: `/api/v1/stores/:storeId/inventory`), the child router **CANNOT** access `:storeId` by default unless configured.

Always declare nested child routers with `{ mergeParams: true }`.

```javascript
// src/routes/index.js
apiRouter.use('/stores/:storeId/inventory', inventoryRouter);

// src/modules/inventory/inventory.routes.js
// ❌ BAD: req.params.storeId will be undefined!
export const inventoryRouter = Router();

// ✅ SENIOR STANDARD: Parent URL params merged into child router
export const inventoryRouter = Router({ mergeParams: true });
```

---

### Rule 7: Clean Central API Gateway Mounting
Modules must never mount themselves directly to `app.js`. All feature routers must export a single named router and be mounted in a central versioned gateway (`src/routes/index.js` under `/api/v1`).

```javascript
// src/routes/index.js (Single Gateway)
import { Router } from 'express';
import { storeRouter } from '../modules/stores/stores.routes.js';
import { medicineRouter } from '../modules/medicines/medicines.routes.js';
import { cartRouter } from '../modules/cart/cart.routes.js';
import { orderRouter } from '../modules/orders/orders.routes.js';

export const apiRouter = Router();

apiRouter.use('/stores', storeRouter);
apiRouter.use('/medicines', medicineRouter);
apiRouter.use('/cart', cartRouter);
apiRouter.use('/orders', orderRouter);
```

---

# Standard Production Route Template

```javascript
/**
 * Order Routes
 * ------------
 * HTTP route definitions for Order lifecycle management.
 * Base Path: /api/v1/orders
 */

import { Router } from 'express';
import { authenticateCustomer } from '../../middlewares/auth.js';
import { validate } from '../../middlewares/validate.js';
import { orderController } from './orders.controller.js';
import {
  cancelOrderBodySchema,
  createOrderBodySchema,
  listOrdersQuerySchema,
  orderIdParamSchema,
} from './orders.validation.js';

export const orderRouter = Router();

// 1. Blanket Authentication Guard
orderRouter.use(authenticateCustomer);

// ---------------------------------------------------------------------------------------------
// Customer Order Operations
// ---------------------------------------------------------------------------------------------

/**
 * POST /api/v1/orders
 * Place new order from active locked cart.
 */
orderRouter.post(
  '/',
  validate({ body: createOrderBodySchema }),
  orderController.createOrder,
);

/**
 * GET /api/v1/orders
 * List paginated customer order history.
 */
orderRouter.get(
  '/',
  validate({ query: listOrdersQuerySchema }),
  orderController.listOrders,
);

// ---------------------------------------------------------------------------------------------
// Parameterized Resource Routes (:id)
// Declared AFTER static root routes to prevent collision
// ---------------------------------------------------------------------------------------------

/**
 * GET /api/v1/orders/:id
 * Fetch detailed order summary and live delivery tracking timeline.
 */
orderRouter.get(
  '/:id',
  validate({ params: orderIdParamSchema }),
  orderController.getOrderById,
);

/**
 * POST /api/v1/orders/:id/cancel
 * Customer cancels order before packing.
 */
orderRouter.post(
  '/:id/cancel',
  validate({
    params: orderIdParamSchema,
    body: cancelOrderBodySchema,
  }),
  orderController.cancelOrder,
);
```

---

# The 60-Second Pre-Commit Route Checklist

Before committing a Route file, audit your code against this checklist:

- [ ] **No Inline Logic:** Are all route handlers delegating to controller methods (zero `(req, res) => {}` logic)?
- [ ] **Collision-Proof:** Are all static endpoints (`/export`, `/bulk-upload`, `/me`) placed **ABOVE** dynamic `/:id` paths?
- [ ] **Validation Guarded:** Does every single route have `validate(...)` chained before the controller?
- [ ] **Top-Level Security:** Is `router.use(authenticate...)` used for blanket protection instead of duplicating auth on every line?
- [ ] **RESTful Standards:** Are paths plural nouns (`/orders`, `/cart/items`), multi-words in kebab-case (`/price-list`), and verbs reserved only for actions (`/cancel`, `/checkout`)?
- [ ] **Nested Params:** If the router is nested under a parent path with params, is `{ mergeParams: true }` enabled?
- [ ] **Clean Gateway Mount:** Is the router cleanly exported as a named object and mounted under `/api/v1` in `routes/index.js`?
