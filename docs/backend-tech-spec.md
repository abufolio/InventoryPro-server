# InventoryPro — Backend Technical Specification

**Source:** [`message.txt`](./message.txt)  
**Version:** 1.0  
**Purpose:** Standalone implementation reference for backend development

## 1. Purpose and scope

This document consolidates the backend requirements from `message.txt`: business rules, architecture, data model, REST API, security, transaction integrity, tests, and delivery criteria.

Requirements that are missing details or conflict in the source are listed under **Open decisions**. Do not treat an assumption as an approved behavior until the product owner confirms it. Suggested implementation choices are marked **Recommendation**.

InventoryPro is a multi-warehouse Warehouse Management System (WMS). The backend must support:

- Authentication and role-based authorization.
- Product, category, warehouse, and supplier management.
- Current quantity for every product/warehouse pair.
- Stock In, Stock Out, and warehouse Transfer operations.
- Immutable stock movement history and audit logs.
- Low-stock queries, dashboard statistics, and reports.
- Request validation, consistent errors, Swagger/OpenAPI, and tests.

### 1.1. MVP exclusions

- Online payments and e-commerce.
- AI forecasting.
- Accounting integration.
- Barcode scanner hardware integration.
- Mobile application.
- WebSocket real-time notifications.

## 2. Technology and architecture

### 2.1. Required stack

| Technology | Purpose |
|---|---|
| Node.js | Runtime |
| Express.js | HTTP framework |
| TypeScript | Type safety |
| PostgreSQL | Database |
| Prisma | ORM and migrations |
| JWT | Access tokens |
| bcrypt | Password hashing |
| Zod | Request validation |
| Helmet | HTTP security headers |
| CORS | Origin restrictions |
| Multer | Product image uploads |
| Swagger / OpenAPI | API documentation |
| dotenv | Environment configuration |

The source recommends Vitest, Supertest, and Postman for testing. Select the test runner and production file-storage service before implementation.

### 2.2. Request flow and boundaries

```text
HTTP request
  -> Router
  -> Authentication / authorization middleware
  -> Request validation
  -> Controller
  -> Service (use case and business rules)
  -> Prisma / PostgreSQL
  -> Standard response or centralized error handler
```

- **Router:** maps HTTP methods and paths to middleware and controllers.
- **Middleware:** handles cross-cutting authentication, authorization, and validation.
- **Controller:** translates HTTP input/output and calls a service; it contains no business rules.
- **Service:** implements use cases, business rules, and transaction boundaries.
- **Prisma:** performs database access through services/use cases.

### 2.3. Suggested source layout

```text
src/
├── config/
│   ├── env.ts
│   ├── database.ts
│   └── swagger.ts
├── modules/
│   ├── auth/
│   ├── users/
│   ├── products/
│   ├── categories/
│   ├── warehouses/
│   ├── suppliers/
│   ├── inventory/
│   ├── stock-in/
│   ├── stock-out/
│   ├── transfers/
│   ├── movements/
│   ├── dashboard/
│   ├── reports/
│   └── audit-logs/
├── middlewares/
│   ├── authenticate.ts
│   ├── authorize.ts
│   ├── validate.ts
│   ├── error-handler.ts
│   └── not-found.ts
├── utils/
│   ├── api-response.ts
│   ├── app-error.ts
│   ├── pagination.ts
│   └── reference-generator.ts
├── app.ts
└── server.ts
```

Keep router/controller/service/validation/tests within feature modules where practical. The source does not require a separate repository layer. Follow one consistent Prisma access pattern and do not duplicate business rules in controllers.

API base URL:

- Development: `http://localhost:5050/api/v1`
- Production example: `https://api.inventorypro.example.com/api/v1`
- All routes use `/api/v1`. Endpoint paths below omit the prefix; for example, `/auth/login` means `/api/v1/auth/login`.

## 3. Data model

### 3.1. Core models

The source names these database models:

1. `User`
2. `RefreshToken`
3. `Product`
4. `Category`
5. `Warehouse`
6. `Supplier`
7. `Inventory`
8. `StockIn`
9. `StockInItem`
10. `StockOut`
11. `StockOutItem`
12. `StockTransfer`
13. `StockTransferItem`
14. `StockMovement`
15. `AuditLog`

### 3.2. Fields and invariants

**User**

- `id`: UUID; `fullName`: required.
- `email`: required and unique.
- `passwordHash`: bcrypt hash; never store a plain-text password.
- `role`: `ADMIN`, `MANAGER`, or `OPERATOR`; default `OPERATOR`.
- `isActive`: default `true`; include `createdAt` and `updatedAt`.
- Inactive users cannot log in or refresh tokens.
- Prevent an Admin from accidentally deactivating their own account.

**Product**

- `id`, `name`, `sku`, `description`, `categoryId`, `supplierId`.
- `purchasePrice` and `salePrice`: non-negative `Decimal(12,2)`.
- `unit`: `PIECE`, `KG`, `LITER`, `METER`, or `BOX`.
- `minStockLevel`: default 0; optional `imageUrl`; `isActive`: default `true`.
- Include `createdAt` and `updatedAt`.
- SKU is unique; category must exist; validate supplier when provided.
- Do not hard-delete a product referenced by history; deactivate it instead.
- A product's unit cannot be changed after creation.

**Category**

- `id`, `name`, `description`, optional `parentId`, timestamps.
- `name` is unique.
- A category cannot be its own parent or the parent of one of its descendants.
- Do not hard-delete a category referenced by products.
- Detect hierarchy cycles in service logic; a foreign key alone does not prevent them.

**Warehouse**

- `id`, `name`, unique `code`, `address`, `description`, `isActive`, timestamps.
- Name is required.
- Do not hard-delete a warehouse with inventory or historical operations.
- Inactive warehouses cannot be used for Stock In, Stock Out, or Transfer.

**Supplier**

- `id`, `name`, `companyName`, optional `email`, `phone`, `address`, `description`, `isActive`, timestamps.
- Name is required. Validate email format when supplied.
- Preserve historical Stock In records; deactivate rather than hard-delete.

**Inventory**

- `id`, `productId`, `warehouseId`, `quantity`, `averageUnitCost`, `updatedAt`.
- `quantity`: integer, default 0, never negative.
- `averageUnitCost`: `Decimal(12,2)`, default 0.
- Unique constraint on `(productId, warehouseId)`.
- Add indexes for common warehouse, product, and reporting queries.

**StockIn / StockOut / StockTransfer and items**

Headers need an ID, reference number, creator, note, timestamp, and relevant warehouse/supplier/reason fields. Add a status when the confirmed workflow requires one. Items need their header relation, product, quantity, and applicable unit cost.

The source does not provide complete Prisma fields, relations, or state models for these entities. Finalize their schema before writing migrations. Historical items and movements must not be edited or deleted.

**StockMovement**

- `id`, `productId`, `warehouseId`.
- `type`: `STOCK_IN`, `STOCK_OUT`, `TRANSFER_IN`, `TRANSFER_OUT`, `ADJUSTMENT`, or `REVERSAL`.
- `quantity`: positive integer; the movement type indicates direction.
- Optional `unitCost`: `Decimal(12,2)`.
- `referenceType`, `referenceId`, `createdById`, `createdAt`.
- Index `(productId, warehouseId)` and `createdAt`.
- Movements are immutable; expose no update/delete endpoint.

**RefreshToken**

Store a token hash, user relation, expiry, and revocation state/time. Decide whether session/device data and token-rotation lineage are needed. Never store the raw refresh token.

**AuditLog**

- User ID, action, entity, entity ID.
- Old/new values, IP address, timestamp.
- Only Admin may read audit logs.
- Never include passwords, token values, or secrets in audit payloads.

### 3.3. Database constraints and deletion

Enforce at the database layer:

- Unique user email, product SKU, warehouse code, and category name.
- Unique `(productId, warehouseId)` inventory pair.
- Foreign-key integrity.
- CHECK constraints for non-negative quantity, minimum stock, prices, and costs.
- Positive movement quantity.
- Enum constraints and deletion rules that preserve historical data.

Do not cascade-delete historical movements or operations. Deactivate referenced master records instead. Confirm whether uniqueness is case-sensitive before selecting the database constraint.

### 3.4. Quantity precision and valuation

The source examples use integer inventory quantities, but units such as `KG`, `LITER`, and `METER` may require fractions. Confirm decimal quantity requirements before finalizing the schema.

Weighted-average cost:

```text
new average cost =
  (old quantity * old average cost + incoming quantity * unit cost)
  / new total quantity
```

Stock In updates average cost. Transfer carries the cost and does not change total inventory value. Confirm rounding precision and whether average cost is retained when quantity reaches zero.

## 4. Business rules

### 4.1. Master data

- Product name is required; SKU is unique; purchase/sale prices are non-negative.
- Product category must exist; validate supplier when provided.
- Category hierarchy must be valid and cycle-free.
- Warehouse code is unique; inactive warehouses cannot accept stock operations.
- Supplier name is required; validate a supplied email.
- Deactivate referenced master records instead of hard-deleting them.
- Product unit is immutable after creation.

### 4.2. Inventory status

Inventory stores the current quantity for each product/warehouse pair. Quantity changes only through Stock In, Stock Out, Transfer, or an approved Adjustment. The frontend cannot directly change quantity.

```text
Available Stock = Total Stock In - Total Stock Out + Transfer In - Transfer Out
```

- `OUT_OF_STOCK`: quantity is 0.
- `LOW_STOCK`: quantity is greater than 0 and at or below the minimum threshold.
- `IN_STOCK`: all other quantities.

The source defines `minStockLevel` on Product but does not say whether the threshold is global or warehouse-specific. This affects both schema and API behavior.

### 4.3. Stock In

1. Accept warehouse, supplier, products, quantities, unit costs, and optional note.
2. Validate user permissions and the existence/active state of referenced records.
3. Require positive quantities and non-negative unit costs.
4. Decide whether duplicate product rows are rejected or combined.
5. Calculate totals on the backend; do not trust a client-supplied total.
6. In one transaction write header/items, inventory, weighted-average cost, `STOCK_IN` movements, and audit log.
7. Roll back the entire transaction if any step fails.
8. Do not edit/delete confirmed operations; correct them with reversal/adjustment movements.

### 4.4. Stock Out

- Reasons: `SALE`, `DAMAGE`, `INTERNAL_USE`, `RETURN_TO_SUPPLIER`, `OTHER`.
- Quantity must be positive and no greater than available stock.
- Reject inactive warehouses or products.
- Prevent negative inventory under concurrent requests.
- In one transaction write header/items, inventory decrement, `STOCK_OUT` movement, and audit log.
- Do not silently edit a completed Stock Out.

### 4.5. Transfer

- States: `PENDING`, `COMPLETED`, `CANCELLED`.
- Source and destination must be different, valid, active warehouses.
- Source inventory must cover every requested product quantity.
- A product may appear only once in a transfer.
- Confirmation decrements source, increments destination, creates `TRANSFER_OUT` and `TRANSFER_IN` movements, and updates status in one transaction.
- If any source item is insufficient, roll back the entire transfer.
- Transfer does not change total quantity or total inventory value.

**Recommendation:** `POST /transfers` creates a `PENDING` header/items without changing inventory. `/confirm` performs the transfer and sets `COMPLETED`. Cancellation is allowed only while `PENDING`. The source does not fully define when transfer inventory changes.

### 4.6. Reversal and adjustment

- Movement history is immutable.
- Correct confirmed Stock In errors with a separate correction operation.
- The source lists `ADJUSTMENT` and `REVERSAL` but does not define permissions, request shapes, or source-movement relations.
- Define endpoints, allowed roles, required reason, maximum reversal quantity, and source linkage before production.
- Recommended: link a reversal to its source movement and prevent duplicate reversal.

## 5. Roles and authorization

Enforce authorization on every protected endpoint. Frontend permission checks control presentation only.

| Module | ADMIN | MANAGER | OPERATOR |
|---|---|---|---|
| Dashboard | Full | View | View |
| Products / Categories | Full | Full | View |
| Warehouses / Inventory | Full | View | View |
| Suppliers | Full | Full | View |
| Stock In / Out / Transfer | Full | Full | Create |
| Reports | Full | View | No access |
| Users / Audit Logs | Full | No access | No access |

Additional rules:

- Manager cannot manage users or roles.
- Operator cannot delete products, manage suppliers/users/roles, or export reports.
- Operators should see their own operation history; whether movement list endpoints require an owner filter is not clear in the source.
- Do not allow mass assignment of role/status fields.
- The source does not define whether Managers/Operators are restricted to assigned warehouses.

## 6. REST API

All paths use `/api/v1`. Protected routes require `Authorization: Bearer <accessToken>`.

| Module | Method and path |
|---|---|
| Auth | `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me` |
| Users | `GET /users`, `GET /users/:id`, `POST /users`, `PATCH /users/:id`, `PATCH /users/:id/role`, `PATCH /users/:id/status` |
| Products | `GET /products`, `GET /products/:id`, `POST /products`, `PATCH /products/:id`, `PATCH /products/:id/status`, `GET /products/:id/movements` |
| Categories | `GET /categories`, `GET /categories/:id`, `POST /categories`, `PATCH /categories/:id`, `PATCH /categories/:id/status` |
| Warehouses | `GET /warehouses`, `GET /warehouses/:id`, `POST /warehouses`, `PATCH /warehouses/:id`, `PATCH /warehouses/:id/status`, `GET /warehouses/:id/inventory`, `GET /warehouses/:id/movements` |
| Suppliers | `GET /suppliers`, `GET /suppliers/:id`, `POST /suppliers`, `PATCH /suppliers/:id`, `PATCH /suppliers/:id/status` |
| Inventory | `GET /inventory`, `GET /inventory/:id`, `GET /inventory/summary`, `GET /inventory/low-stock`, `GET /inventory/out-of-stock`, `GET /inventory/by-warehouse/:warehouseId` |
| Stock In | `GET /stock-in`, `GET /stock-in/:id`, `POST /stock-in`, `POST /stock-in/:id/confirm` |
| Stock Out | `GET /stock-out`, `GET /stock-out/:id`, `POST /stock-out`, `POST /stock-out/:id/confirm` |
| Transfers | `GET /transfers`, `GET /transfers/:id`, `POST /transfers`, `POST /transfers/:id/confirm`, `POST /transfers/:id/cancel` |
| Movements | `GET /movements`, `GET /movements/:id` |
| Dashboard | `GET /dashboard/summary`, `GET /dashboard/stock-trends`, `GET /dashboard/category-distribution`, `GET /dashboard/warehouse-distribution`, `GET /dashboard/recent-movements` |
| Reports | `GET /reports/inventory`, `GET /reports/stock-in`, `GET /reports/stock-out`, `GET /reports/warehouse`, `GET /reports/low-stock`, `GET /reports/export` |
| Audit | `GET /audit-logs`, `GET /audit-logs/:id` (Admin only) |

### 6.1. Query parameters and request rules

- Product list: `page`, `limit`, `search`, `categoryId`, `supplierId`, `isActive`, `sortBy`, `sortOrder`.
- Inventory list: `page`, `limit`, `search`, `warehouseId`, `categoryId`, `status`.
- Movement list: `productId`, `warehouseId`, `type`, `userId`, `startDate`, `endDate`, `page`, `limit`.
- Reports: `startDate`, `endDate`, `warehouseId`; export also accepts `format=csv`.
- Validate every query with Zod; restrict `sortBy` to an allowlist.
- `page` is a positive integer starting at 1. `limit` has a server-side maximum.

Stock In request:

```json
{
  "warehouseId": "uuid",
  "supplierId": "uuid",
  "note": "Monthly stock purchase",
  "items": [
    {
      "productId": "uuid",
      "quantity": 10,
      "unitCost": 650
    }
  ]
}
```

Stock Out request:

```json
{
  "warehouseId": "uuid",
  "reason": "SALE",
  "note": "Customer order",
  "items": [
    {
      "productId": "uuid",
      "quantity": 3
    }
  ]
}
```

Transfer request:

```json
{
  "sourceWarehouseId": "uuid",
  "destinationWarehouseId": "uuid",
  "note": "Warehouse redistribution",
  "items": [
    {
      "productId": "uuid",
      "quantity": 5
    }
  ]
}
```

## 7. Responses and errors

Success response:

```json
{
  "success": true,
  "message": "Products retrieved successfully",
  "data": []
}
```

Paginated response:

```json
{
  "success": true,
  "data": [],
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "totalPages": 10
  }
}
```

Error response:

```json
{
  "success": false,
  "message": "Validation failed",
  "errors": [
    {
      "field": "email",
      "message": "Invalid email address"
    }
  ]
}
```

HTTP status: `200` success, `201` created, `204` empty success, `400` invalid request, `401` unauthenticated, `403` forbidden, `404` not found, `409` conflict, `422` validation error, `429` rate limit, `500` internal error.

Never return stack traces, SQL, tokens, or secrets in production errors. A centralized error handler maps Zod, Prisma, domain, and unknown errors to the standard response. Include a request/correlation ID in server logs.

## 8. Transactions and concurrency

### 8.1. Transaction boundary

Write operation header/items, inventory update, movements, Stock In average cost, transfer status, and audit log in one database transaction. Return success only after commit. Roll back all writes on any failure. Do not make external HTTP or file-storage calls inside the transaction.

### 8.2. Prevent negative inventory

A Prisma transaction alone does not prevent concurrent Stock Out requests from overselling. Choose and test a PostgreSQL concurrency strategy.

1. Add a database CHECK constraint `quantity >= 0`.
2. Make validation and decrement atomic, for example a conditional update requiring `quantity >= requested` and checking affected rows, or an appropriate row-lock/isolation strategy.
3. Lock/update Transfer source and destination inventory rows in a stable order.
4. If retrying deadlocks/serialization failures, bound retries and prevent duplicate movements.
5. Add integration tests for concurrent requests.

### 8.3. Idempotency

The source does not require idempotency keys. Network retries can create duplicate stock operations. **Recommendation:** support `Idempotency-Key` for Stock In/Out/Transfer writes and return the prior result for a repeated request. Confirm key retention and whether it is required.

## 9. Authentication and security

- Hash passwords with bcrypt.
- Access JWT lifetime: 15 minutes. Refresh token lifetime: 7 days.
- Send refresh tokens in `HttpOnly`, `Secure`, `SameSite` cookies; store only their hashes.
- Send access tokens in the `Authorization: Bearer` header.
- Revoke refresh tokens server-side on logout.
- Use separate environment secrets for `JWT_ACCESS_SECRET` and `JWT_REFRESH_SECRET`.
- Return a generic login error that does not reveal whether an account exists.
- Inactive users cannot log in or refresh.
- Require HTTPS in production.
- Configure a CORS origin allowlist and decide CSRF protection for cookie-based refresh.
- Never combine wildcard `Access-Control-Allow-Origin` with credentials.
- Rate-limit login, refresh, and password-related endpoints.
- Use Prisma parameterized queries; never build raw SQL from user input.
- Validate all body, route, and query input with Zod.
- Use an allowlist for writable fields to prevent mass assignment.
- Do not expose internal error details in production.

### 9.1. Product image upload

Allowed formats: JPEG, PNG, WEBP. Maximum size: 5 MB. Check actual file format in addition to MIME type, and generate filenames on the server. The source does not define production storage or an upload route. Decide these before implementation; do not store binary files in the database.

### 9.2. Audit and logs

Audit actor, action, entity, entity ID, old/new values, IP, and timestamp. Never log passwords, token values, or secrets. Store the operation, movement, and audit record in the same transaction. Trust proxy headers only when trusted-proxy configuration is set.

## 10. Dashboard, reports, and deployment

Dashboard metrics:

- Total products and warehouses.
- Total stock quantity and inventory value.
- Low-stock and out-of-stock counts.
- Stock In vs Stock Out trends.
- Category and warehouse distributions.
- Recent movements.

```text
Inventory Value = SUM(Available Quantity * Average Cost)
```

Use weighted-average cost for MVP. Stock In updates average cost; Transfer does not change total value. Reports support date range and warehouse filters. Limit CSV export size. Confirm timezone, date boundaries, currency, and rounding before implementation.

Environment example:

```env
NODE_ENV=development
PORT=5050
DATABASE_URL=postgresql://...
JWT_ACCESS_SECRET=...
JWT_REFRESH_SECRET=...
FRONTEND_URL=http://localhost:5173
```

- Never commit real secrets. Validate required configuration during startup.
- Build/start commands: `pnpm build`, `pnpm start`.
- Use Prisma migrations in production; do not manage production schema with `db push`.
- Make seed scripts safe to rerun.
- Plan backups, restore checks, and restricted database access.
- Manage Prisma connection startup/shutdown correctly.
- Decide whether deployment needs `/health` or readiness endpoints.

## 11. Testing and acceptance

### 11.1. Required tests

**Authentication:** valid/invalid login, invalid/expired access token, refresh, logout revocation, inactive account.

**Master data and permissions:** product creation, duplicate SKU, invalid category, negative price, category cycles, protected deletes, Admin/Manager/Operator permissions, 401 and 403 cases.

**Inventory:** Stock In, Stock Out, insufficient stock, Transfer, inactive warehouse/product, transaction rollback, immutable movements, audit records, concurrent Stock Out.

**API and reporting:** validation, pagination, search, filters, sorting, dashboard/report aggregation, CSV date range/size limit, OpenAPI consistency.

### 11.2. Critical end-to-end scenario

Initial quantity: Main Warehouse has 20 Laptop; Samarkand Warehouse has 5 Laptop. Add 10 to Main, remove 5 from Main, then transfer 8 from Main to Samarkand.

| Warehouse | Initial | In | Out | Transfer | Final |
|---|---:|---:|---:|---:|---:|
| Main | 20 | +10 | -5 | -8 | 17 |
| Samarkand | 5 | 0 | 0 | +8 | 13 |

Total quantity is `17 + 13 = 30`. Transfer does not change total quantity.

### 11.3. Backend completion checklist

- [ ] Login, refresh, logout, and current-user endpoints work.
- [ ] Passwords and refresh tokens are stored securely.
- [ ] Every protected endpoint enforces authorization.
- [ ] Master-data endpoints and database constraints work.
- [ ] Inventory quantity changes only through stock operations.
- [ ] Stock operations are atomic and cannot produce negative quantity.
- [ ] Movement history is immutable; required audit is written transactionally.
- [ ] Low-stock, dashboard, and report calculations are tested.
- [ ] List APIs provide server-side pagination, filtering, and search.
- [ ] Swagger, validation, error handling, and tests are complete.
- [ ] Migrations, seed, `.env.example`, README, and deployment are ready.

## 12. Open decisions

Confirm these with the product owner before finalizing the schema/API:

1. **Stock In/Out lifecycle:** Does `POST` confirm immediately, or create a draft that changes inventory only at `/confirm`?
2. **Transfer lifecycle:** Should `POST` create `PENDING` and confirmation atomically move stock? Is partial shipment needed?
3. **Fractional quantities:** Are decimal quantities required for `KG`, `LITER`, and `METER`?
4. **Low-stock threshold:** Is `minStockLevel` global per product or separate per product/warehouse?
5. **Costing:** Is weighted average global or per warehouse? What are rounding and zero-stock rules?
6. **Operator history:** Can Operators see only their own operations/movements?
7. **Reports:** Can Managers export CSV?
8. **Idempotency:** Are idempotency keys required for stock writes?
9. **Adjustment/Reversal:** Which endpoints, roles, reasons, limits, and source links are allowed?
10. **Product images:** Which storage service, upload API, and cleanup policy will be used?
11. **User provisioning:** How is the initial password set? Is password reset/change in MVP?
12. **Warehouse access:** Are Managers/Operators restricted to assigned warehouses?
13. **Uniqueness:** Are email, SKU, category name, and warehouse code case-insensitive?
14. **Reports:** Which timezone, date boundaries, currency, and CSV size limit apply?
15. **Reference numbers:** What format and uniqueness scope are required?
16. **Audit retention:** How long are logs retained, and what is the archival policy?
17. **Error contract:** What is the 400/422 boundary, and which endpoints use 204?

## 13. Source analysis summary

This specification consolidates the source's backend requirements and backend-relevant full-stack requirements: MVP scope, business rules, role matrix, endpoints, model list, response contract, security, transactions, testing, deployment, and acceptance criteria.

The source includes Prisma examples for `User`, `Product`, `Inventory`, and `StockMovement`, but not complete schemas for the other models. Stock In/Out/Transfer and AuditLog relations must be finalized before implementation; this document does not present invented schemas as approved designs.
