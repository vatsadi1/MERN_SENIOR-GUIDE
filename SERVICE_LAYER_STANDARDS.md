# ⚙️ Backend Engineering Standards: The Senior Developer's Guide to Service Layer Architecture

> **Author / Reference:** MDP Engineering Architecture  
> **Target Environment:** Node.js, Express, MongoDB (Mongoose), Modular Monolith & Microservices  
> **Purpose:** Principles and architectural laws for writing decoupled, failure-isolated, and high-performance business services.

---

## 📑 Table of Contents
1. [The Role of the Service Layer](#the-role-of-the-service-layer)
2. [The 7 Golden Rules of Service Design](#the-7-golden-rules-of-service-design)
3. [Standard Production Service Template](#standard-production-service-template)
4. [The 60-Second Pre-Commit Service Checklist](#the-60-second-pre-commit-service-checklist)

---

## The Role of the Service Layer

In production backends, we follow **"Thin Controllers, Rich Services"**:
- **Controller:** Pure transport layer (HTTP status codes, reading `req.validated`, calling service, returning `res.json()`). Zero business logic. Zero database queries.
- **Service:** The **Brain** of the application. Owns all business logic, validation rules, state transitions, financial math, and event emissions.

---

# The 7 Golden Rules of Service Design

---

### Rule 1: The Zero-HTTP Decoupling Law (Never Pass `req` or `res`)
A service method must **NEVER** accept Express `req`, `res`, or `next` objects.

> [!IMPORTANT]
> If a service method takes `(req, res)`, it can only ever be called from an HTTP route. You can never call it from a **Cron Job**, a **Queue Worker**, an **Event Subscriber**, a **WebSocket handler**, or a **Unit Test** without mocking the entire HTTP request!

```javascript
// ❌ BAD: Coupled to Express HTTP transport
async cancelOrder(req, res) {
  const orderId = req.params.id;
  const customerId = req.user.id;
  // ...
}

// ✅ SENIOR STANDARD: Decoupled domain arguments (Pure DTOs / Primitives)
async cancelOrder(orderId, customerId, cancellationReason, actor) {
  // Can now be invoked by HTTP controller, CLI script, Cron worker, or Unit test!
}
```

---

### Rule 2: In-Memory Processing over DB Roundtrips (Kill the N+1 Query)
Never execute database queries inside loops. Fetch all dependencies upfront in a single batch, perform business logic and calculations in RAM (CPU speed: $\sim 0.01\text{ms}$), and write back in a single atomic operation.

```javascript
// ❌ BAD: N+1 Database Roundtrips (If cart has 10 items, 10 queries hit the DB!)
for (const item of cart.items) {
  const medicine = await Medicine.findById(item.medicineId); // 🐌 10 network latency roundtrips
  total += medicine.pricePaise * item.quantity;
}

// ✅ SENIOR STANDARD: Single Batch Query + RAM Processing
// Step 1: 1 single DB query fetches all medicines at once
const medicineIds = cart.items.map(i => i.medicine);
const medicines = await Medicine.find({ _id: { $in: medicineIds } }).lean();
const medicineMap = new Map(medicines.map(m => [m._id.toString(), m]));

// Step 2: RAM Loop (Instant microsecond math)
for (const item of cart.items) {
  const medicine = medicineMap.get(item.medicine.toString());
  total += medicine.pricePaise * item.quantity;
}
```

---

### Rule 3: Atomic State Transitions via the FSM Matrix
Never update entity status blindly (e.g., `order.status = 'DELIVERED'`). Always validate current status against the **Finite State Machine Matrix** defined in your `constants.js`.

```javascript
// ✅ SENIOR STANDARD: State transition assertion helper
_assertValidTransition(currentStatus, nextStatus) {
  const allowed = ORDER_STATUS_TRANSITIONS[currentStatus] || [];
  if (!allowed.includes(nextStatus)) {
    throw ApiError.conflict(
      `Cannot transition order from '${currentStatus}' to '${nextStatus}'. Allowed moves: [${allowed.join(', ')}]`
    );
  }
}

// In business method:
async markOrderAsPacking(orderId, actor) {
  const order = await this.findById(orderId);
  this._assertValidTransition(order.status, ORDER_STATUS.PACKING); // 🛡️ Illegal jump blocked!
  
  order.status = ORDER_STATUS.PACKING;
  order.packedAt = new Date();
  await order.save();
}
```

---

### Rule 4: Concurrency Safety & Atomic Conditional Updates
When multiple users compete for limited shared resources (e.g., stock inventory, wallet balances, booking slots), never use **Read-Modify-Save** (`find` $\rightarrow$ `balance -= 10` $\rightarrow$ `save`). A race condition will allow overdrafts.

Always use atomic MongoDB operators (`$inc`, `$set`, `$push`) combined with conditional `$expr` guards.

```javascript
// ❌ BAD: Vulnerable to race conditions (Two users buy the last unit at the same millisecond)
const inventory = await StoreInventory.findById(id);
if (inventory.quantityAvailable >= qty) {
  inventory.quantityReserved += qty;
  await inventory.save(); // Overwritten by concurrent request!
}

// ✅ SENIOR STANDARD: Atomic conditional update
const updated = await StoreInventory.findOneAndUpdate(
  {
    _id: inventoryId,
    $expr: {
      $gte: [{ $subtract: ['$quantityOnHand', '$quantityReserved'] }, qty],
    },
  },
  { $inc: { quantityReserved: qty } },
  { new: true }
);

if (!updated) {
  throw ApiError.conflict('Insufficient stock available to complete reservation');
}
```

---

### Rule 5: The Compensating Transaction Rule (Saga Rollbacks)
In distributed operations involving multiple entities (e.g., Stock Reserve $\rightarrow$ Payment Intent $\rightarrow$ Order Creation), if step 3 fails, the service must execute a **Compensating Transaction** to clean up step 1.

```javascript
async checkout(orderPayload) {
  const reservedBatches = [];
  try {
    // Step 1: Reserve physical inventory
    for (const item of orderPayload.items) {
      await inventoryService.reserveStock(item);
      reservedBatches.push(item);
    }

    // Step 2: Attempt Payment / External Gateway
    const payment = await paymentService.initiate(orderPayload);

    // Step 3: Create Order Record
    return await Order.create({ ...orderPayload, paymentId: payment._id });
  } catch (err) {
    // 🛡️ COMPENSATING TRANSACTION: Roll back reserved stock so it is not locked forever!
    if (reservedBatches.length > 0) {
      await inventoryService.releaseStock(reservedBatches);
    }
    throw err; // Re-throw to inform caller
  }
}
```

---

### Rule 6: Synchronous Core vs Asynchronous Side-Effects (Event Bus Rule)
- **Synchronous (`await`):** The core database write that changes state must be direct and guaranteed.
- **Asynchronous (Event Emission):** Non-critical side-effects (push notifications, SMS, emails, analytics, Slack alerts) must be dispatched via an **Event Bus** on the next event loop tick (`setImmediate`).

> [!WARNING]
> An SMS provider network timeout or third-party email outage must **NEVER** crash the user's checkout or roll back a paid order!

```javascript
// ✅ SENIOR STANDARD: Core saved first, side-effects emitted
async acceptOrder(orderId, actor) {
  // 1. Guaranteed Core Database Write
  const order = await this.findById(orderId);
  this._assertValidTransition(order.status, ORDER_STATUS.STORE_ACCEPTED);
  
  order.status = ORDER_STATUS.STORE_ACCEPTED;
  order.storeAcceptedAt = new Date();
  await order.save();

  // 2. Fire-and-forget Event for Notifications & Analytics
  eventBus.emit(DOMAIN_EVENTS.ORDER.STORE_ACCEPTED, {
    orderId: order._id,
    orderCode: order.orderCode,
    customerId: order.customer,
  });

  return order;
}
```

---

### Rule 7: Idempotency by Design
A well-engineered service handles redundant requests gracefully without throwing HTTP 500 errors or performing redundant database writes.

If a customer clicks "Clear Cart" or "Remove Coupon" when the cart is already empty or has no coupon, return the current state immediately.

```javascript
// ✅ SENIOR STANDARD: Idempotent exit
async removeCoupon(customerId) {
  const cart = await this.getActiveCart(customerId);

  // Idempotent check: If already no coupon, exit early with zero redundant DB writes
  if (!cart.couponCode && cart.discountPaise === 0) {
    return cart;
  }

  cart.couponCode = null;
  cart.discountPaise = 0;
  this.recalculateTotals(cart);
  await cart.save();

  return cart;
}
```

---

# Standard Production Service Template

```javascript
/**
 * Order Service
 * -------------
 * Pure business logic layer for order lifecycle management.
 */

import { BaseService } from '../../shared/BaseService.js';
import { ApiError } from '../../utils/ApiError.js';
import { eventBus } from '../../shared/events/eventBus.js';
import { DOMAIN_EVENTS } from '../../shared/events/events.constants.js';
import { Order } from './orders.model.js';
import {
  ORDER_STATUS,
  ORDER_STATUS_TRANSITIONS,
} from './orders.constants.js';

class OrderService extends BaseService {
  constructor() {
    super(Order, { entityName: 'Order' });
  }

  /**
   * Helper: Guard illegal lifecycle state transitions
   */
  _assertValidTransition(currentStatus, nextStatus) {
    const allowed = ORDER_STATUS_TRANSITIONS[currentStatus] || [];
    if (!allowed.includes(nextStatus)) {
      throw ApiError.conflict(
        `Cannot move order from '${currentStatus}' to '${nextStatus}'. Allowed: [${allowed.join(', ')}]`
      );
    }
  }

  /**
   * Business Method: Cancel an order with audit trail and stock rollback
   */
  async cancelOrder(orderId, customerId, reason, actor) {
    // 1. Fetch domain record
    const order = await this.findById(orderId);

    // 2. Authorization / Ownership Guard
    if (actor.type === 'CUSTOMER' && order.customer.toString() !== customerId.toString()) {
      throw ApiError.forbidden('You do not have permission to cancel this order');
    }

    // 3. FSM State Transition Check
    this._assertValidTransition(order.status, ORDER_STATUS.CANCELLED);

    // 4. Execute Core Business Mutations
    order.status = ORDER_STATUS.CANCELLED;
    order.cancelledAt = new Date();
    order.cancellationReason = reason;
    order.statusHistory.push({
      status: ORDER_STATUS.CANCELLED,
      actorType: actor.type,
      actorId: actor.userId,
      note: reason,
    });

    await order.save();

    // 5. Emit Domain Event for async reactions (Refunds, SMS, Alerts)
    eventBus.emit(DOMAIN_EVENTS.ORDER.CANCELLED, {
      orderId: order._id,
      orderCode: order.orderCode,
      stockReservations: order.stockReservations,
    });

    return order;
  }
}

export const orderService = new OrderService();
```

---

# The 60-Second Pre-Commit Service Checklist

Before committing a new Service file, audit your code against this checklist:

- [ ] **No HTTP Objects:** Are all methods free from Express `req` and `res` parameters?
- [ ] **No N+1 Queries:** Are loops free from `await Model.findById()` calls?
- [ ] **FSM Enforcement:** Does every status change call `_assertValidTransition()`?
- [ ] **Concurrency Safe:** Are inventory/stock/wallet updates using atomic `$inc` or conditional operators?
- [ ] **Compensating Rollbacks:** If a multi-step operation fails midway, are prior steps safely rolled back?
- [ ] **Fire-and-Forget Side Effects:** Are emails, push notifications, and SMS emitted via `eventBus.emit()` rather than blocking HTTP responses?
- [ ] **Idempotent Exits:** Do removal/clearing operations exit cleanly if the entity is already in the target state?
