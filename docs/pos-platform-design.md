# Event POS Platform Blueprint (Food + Drinks)

## 1) Goals and Non-Functional Requirements

### Primary goals
- **Very easy to use** for staff with low digital literacy.
- **Fast order capture** and payment confirmation in high-pressure event conditions.
- **Concurrent operations** for at least **40+ active service staff** posting orders at the same time.
- **Tight inventory control** for:
  - ready-to-sell beverages (sold directly)
  - recipe-based food items (ingredient consumption tracking)

### Key non-functional requirements
- **P95 action latency** under 1.5s for common actions (create order, update payment status, mark issued).
- **99.9% uptime** during event hours.
- **Auditability** for payments, stock movement, and user actions.
- **Offline resilience** for temporary connectivity issues.
- **Role-based access control** with explicit permissions.

---

## 2) User Roles and Permissions

## Role 1: Service Team (Order Posters)
- Create orders quickly.
- Attach table/customer reference (optional).
- Initiate payment:
  - **M-PESA STK Push**
  - **Card**
  - **Cash**
- See own order/payment status.
- Hand over confirmed order to cashier station for issuing.

## Role 2: Cashier
- View incoming orders from assigned service team(s) or all (configurable).
- Confirm/claim payment against order.
- Resolve payment exceptions (e.g., wrong amount, delayed STK callback).
- Mark order as **Issued/Completed**.

## Role 3: Manager
- Product catalog management.
- Stock management (opening stock, stock adjustments, stock takes).
- Recipe management (BOM for food items).
- Pricing/markup setup.
- Reports and export (sales, payments, stock, variance, staff performance).
- User and role assignment.

---

## 3) Core User Flow (End-to-End)

## A) Service Team Flow
1. Login (PIN + device pairing preferred for speed).
2. Tap **New Order**.
3. Select product(s) using large tiles, categories, search.
4. Add quantity/notes.
5. Choose payment method:
   - M-PESA (enter phone, trigger STK)
   - Card (POS terminal reference input)
   - Cash
6. Submit order.
7. System assigns queue token/order number and status = `Pending Payment` or `Paid` depending on method.
8. Service person sees progress badge and can retry STK, edit, or cancel (within policy window).

## B) Cashier Flow
1. Open **Payment Queue**.
2. Filter by service person / payment type / pending exceptions.
3. Validate payment evidence:
   - STK callback success + amount match.
   - Card receipt/ref match.
   - Cash physically received.
4. Click **Confirm Payment**.
5. Issue order (single click) => status `Issued`.
6. If mismatch, mark `Exception` and route to resolution queue.

## C) Manager Flow
1. Pre-event setup:
   - Upload products and prices.
   - Define recipes for food items.
   - Set opening stock.
   - Assign users to shifts and cashier-service mappings.
2. During event:
   - Monitor dashboards (sales velocity, stock depletion, pending queue).
   - Approve stock adjustments and voids.
3. Post-event:
   - Run stock take.
   - View variance report.
   - Close shift/day and export reconciliations.

---

## 4) Information Architecture (Screen Design)

### Service UI (mobile-first)
- **Home**: New Order, My Active Orders, Help.
- **Order Builder**:
  - Product category tabs (Drinks, Food, Combos)
  - Big buttons and numeric steppers
  - Running total fixed at bottom
- **Payment Panel**:
  - Method cards (M-PESA/Card/Cash)
  - Minimal required fields only
- **Status Screen**:
  - Large color-coded states (Pending, Paid, Exception, Issued)

### Cashier UI (tablet/desktop)
- **Live Queue** with auto-refresh/websocket updates.
- **Payment validation drawer** with amount, payer, timestamp, method metadata.
- **Issue panel** with one-click completion and print/send receipt.

### Manager UI (desktop)
- Dashboard widgets (hourly sales, top sellers, low stock alerts).
- Inventory module (products, recipes, stock movements).
- Reports module (financial + operational).
- Admin module (users, roles, shift assignment, device sessions).

---

## 5) Data Model (High-Level)

### Core entities
- `users` (id, name, role, pin_hash, active, assigned_cashier_id)
- `products` (id, sku, name, type: beverage|food, sale_price, active)
- `recipes` (id, food_product_id)
- `recipe_items` (recipe_id, ingredient_product_id, qty_per_unit)
- `inventory_batches` (product_id, qty_on_hand, unit_cost, source)
- `orders` (id, service_user_id, status, subtotal, total, created_at)
- `order_items` (order_id, product_id, qty, unit_price)
- `payments` (order_id, method, amount, status, external_ref, metadata)
- `payment_events` (payment_id, event_type, raw_payload, created_at)
- `stock_movements` (product_id, type, qty_delta, reason, ref_id)
- `shifts` (id, cashier_user_id, opened_at, closed_at)
- `audit_logs` (actor_user_id, action, entity, entity_id, before, after)

### Inventory logic
- Beverage sale => decrement stock directly on issue.
- Food sale => explode recipe and decrement ingredient stock on issue.
- All decrements are transactional to prevent partial consumption updates.

---

## 6) Payment Architecture

## M-PESA STK Push
- Service triggers STK request via backend.
- Backend saves `payments` row with `pending`.
- M-PESA callback webhook updates payment status (`success`/`failed`) and stores raw payload.
- Idempotency key on callback reference to avoid duplicates.
- Auto-reconciliation job flags amount/reference mismatches.

## Card and Cash
- Card: require terminal reference + amount + optional photo/scan of slip.
- Cash: cashier confirmation is mandatory before issue.

## Controls
- Payment/order status state machine (strict transitions).
- No issue without `payment_confirmed = true` (unless manager override with reason).
- Every override recorded in audit log.

---

## 7) Concurrency and Performance (40+ active users)

- Use **stateless API servers** behind load balancer.
- Real-time queue updates via **WebSocket** or SSE.
- DB: PostgreSQL with:
  - proper indexing (`orders.status`, `payments.status`, `created_at`)
  - row-level locking for critical stock/payment transitions
  - read replicas (optional) for reports
- Task queue (e.g., Redis + workers) for async jobs:
  - STK callbacks processing
  - report generation
  - alerting
- Use optimistic UI updates for speed, server authoritative on final state.

---

## 8) Recommended Tech Stack

### Frontend
- React + TypeScript (or Next.js app router).
- UI kit with large touch targets and accessible contrast.
- PWA support for installable mobile workflow.

### Backend
- Node.js (NestJS/Fastify) or Python (FastAPI) with typed schemas.
- PostgreSQL for transactional data.
- Redis for queues/caching/locks.

### Integrations
- Safaricom M-PESA Daraja APIs (STK + callbacks).
- Card provider integration or manual ref workflow.
- SMS/WhatsApp optional receipt notifications.

### Infrastructure
- Containerized deployment (Docker + managed Kubernetes or ECS).
- Managed Postgres + Redis.
- Centralized logging + metrics + alerting.

---

## 9) UX Guidelines for Low Digital Literacy

- Use icon + text labels (not icon-only).
- Keep max 3-step completion for frequent tasks.
- Use local language labels where needed.
- Large buttons, numeric keypad, and confirmation sounds/vibration.
- Color-coded statuses with text fallback (avoid color-only meaning).
- Undo window for accidental taps.
- In-app guided mode for first-time staff.

---

## 10) Security and Compliance

- PIN + optional device binding per staff account.
- Short session timeout with quick re-auth PIN.
- Least-privilege RBAC.
- Full audit trail for monetary/stock actions.
- Encrypt data in transit (TLS) and at rest.
- PII minimization and secure retention policy.

---

## 11) Reporting (Manager)

- Sales by hour/product/category/staff.
- Payments by method and reconciliation status.
- Open exceptions and aging.
- Stock consumed vs expected (recipe-derived) vs actual stock-take.
- Gross margin by product and category.
- Shift close report (cashier accountability).

---

## 12) Suggested Delivery Plan

## Phase 1 (MVP, 3–5 weeks)
- Roles/auth, product catalog, order capture.
- Cash/card/manual payment confirmation.
- Basic stock decrement.
- Core reports + shift close.

## Phase 2 (2–4 weeks)
- M-PESA STK integration + callback handling.
- Exception queue and reconciliation automation.
- Recipe engine for food ingredients.

## Phase 3 (2–4 weeks)
- Performance hardening, observability.
- Offline-first improvements (queued sync).
- Advanced analytics, forecasts, restock alerts.

---

## 13) Additional Recommendations

- Pre-event stress test with at least **2x expected concurrent load**.
- Create role-specific quick training cards (1 page each).
- Add "panic mode" fallback:
  - continue taking orders locally
  - sync once network recovers
- Keep printed QR/NFC table/customer identifiers for faster order tagging.
- Use hardware strategy:
  - service team on low-cost Android phones
  - cashier on tablets with receipt printer integration

---

## 14) Success Metrics

- Average order capture time < 20 seconds.
- Payment confirmation time < 30 seconds median.
- Order issue accuracy > 99%.
- Stock variance after event < 3%.
- Staff onboarding/training time < 1 hour.
