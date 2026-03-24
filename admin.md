# Zentora Admin Module — Frontend Architecture Proposal

> **Scope:** Frontend-focused architecture document. All backend endpoints, request shapes, and response formats are documented here for direct integration by the UI team. Backend capabilities confirmed from codebase analysis of `internal/app/router.go`, `internal/handlers/catalog/`, `internal/domain/`, and existing middleware.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Role Protection Strategy](#2-role-protection-strategy)
3. [Admin Routing Design](#3-admin-routing-design)
4. [Feature Module Structure](#4-feature-module-structure)
5. [API Service Layer](#5-api-service-layer)
6. [Endpoint Reference — Dashboard](#6-endpoint-reference--dashboard)
7. [Endpoint Reference — Products](#7-endpoint-reference--products)
8. [Endpoint Reference — Inventory](#8-endpoint-reference--inventory)
9. [Endpoint Reference — Orders](#9-endpoint-reference--orders)
10. [Endpoint Reference — Users / Admins](#10-endpoint-reference--users--admins)
11. [Endpoint Reference — Catalog Metadata](#11-endpoint-reference--catalog-metadata)
12. [Shared Types Reference](#12-shared-types-reference)
13. [State Management Design](#13-state-management-design)
14. [Missing Backend Endpoints (Proposed)](#14-missing-backend-endpoints-proposed)

---

## 1. Architecture Overview

### What Already Exists

| Layer | Status | Notes |
|---|---|---|
| `RoleRoute` guard | ✅ Ready | Reads `user.roles[]` from `useAuthStore`; redirects to `/unauthorized` |
| `useAuthStore` (Zustand) | ✅ Ready | Persists `user`, `accessToken`, `roles[]` |
| `UserRole` type | ✅ Ready | `"admin" \| "user"` |
| `http` client + interceptors | ✅ Ready | Bearer token auto-attached; 401 auto-clears auth |
| `/admin/home` stub route | ✅ Ready | Wrapped in `RoleRoute roles={["admin"]}` |
| Backend admin catalog endpoints | ✅ Ready | Full CRUD for products, variants, images, categories, tags, attributes, inventory |
| Backend admin notification endpoints | ✅ Ready | Create / bulk / broadcast |
| Backend super-admin endpoints | ✅ Ready | Create/list/deactivate admins |
| Order listing (user-scoped) | ✅ Ready | `GET /orders` — no admin-scoped order list exists yet |

### Confirmed Backend Admin Namespace

All admin-protected routes use `h.AuthMiddleware.AdminOnly()` or `SuperAdminOnly()` middleware.

```
/api/v1/admin/catalog/...        ← full catalog CRUD
/api/v1/admin/notifications/...  ← notification management
/api/v1/admin/...                ← super-admin: admins management
```

---

## 2. Role Protection Strategy

### Guard Hierarchy

```
PublicRoute        → no guard
ProtectedRoute     → isAuthenticated (any logged-in user)
RoleRoute          → isAuthenticated + role check
```

### Admin Route Wrapping

All `/admin/*` routes must be wrapped with `RoleRoute roles={["admin"]}`. This already works — `RoleRoute` reads `user.roles` from persisted Zustand state, which is populated on login from the backend `AuthUser.roles[]`.

```tsx
// src/app/router/adminRoutes.tsx  (pattern to follow)
{
  path: "/admin",
  element: (
    <RoleRoute roles={["admin"]}>
      <Suspense fallback={<LoaderFallback />}>
        <AdminLayout />
      </Suspense>
    </RoleRoute>
  ),
  children: [
    { index: true, element: <AdminDashboardPage /> },
    { path: "products", element: <AdminProductsPage /> },
    // ...
  ]
}
```

### Layout-Level Guard

Wrap the `AdminLayout` component itself with `RoleRoute`. Child routes inside the layout **do not** need individual guards because the layout already gates access. This prevents flash-of-content on deep navigation.

### Switch to Storefront Link

Inside `AdminLayout`, provide a persistent link to `/` (storefront). Since admins share the same auth session, they land on the storefront as a normal logged-in user with no extra steps.

```tsx
// In AdminSidebar / AdminHeader
<Link to="/" target="_self">← View Storefront</Link>
```

---

## 3. Admin Routing Design

### Route Tree

```
/admin                          → AdminDashboardPage       (overview stats, quick actions)
/admin/products                 → AdminProductsPage        (paginated table, search, filter)
/admin/products/new             → AdminProductNewPage      (create product form)
/admin/products/:id             → AdminProductDetailPage   (view/edit product)
/admin/products/:id/variants    → AdminProductVariantsPage (variant management tab)
/admin/inventory                → AdminInventoryPage       (stock table by variant × location)
/admin/orders                   → AdminOrdersPage          (all orders, status filter)
/admin/orders/:id               → AdminOrderDetailPage     (order detail + status update)
/admin/users                    → AdminUsersPage           (user list — future)
/admin/catalog/categories       → AdminCategoriesPage      (category tree CRUD)
/admin/catalog/brands           → AdminBrandsPage          (brand list CRUD)
/admin/catalog/attributes       → AdminAttributesPage      (attribute + values CRUD)
/admin/catalog/discounts        → AdminDiscountsPage       (discount rules CRUD)
```

### Updated `adminRoutes.tsx`

```tsx
import React, { Suspense } from "react";
import { LoaderFallback } from "@/shared/components/ui";
import { RoleRoute } from "@/core/guards/RoleRoute";

const AdminLayout        = React.lazy(() => import("@/features/admin/shared/layouts/AdminLayout"));
const AdminDashboardPage = React.lazy(() => import("@/features/admin/dashboard/pages/AdminDashboardPage"));
const AdminProductsPage  = React.lazy(() => import("@/features/admin/products/pages/AdminProductsPage"));
const AdminProductNewPage    = React.lazy(() => import("@/features/admin/products/pages/AdminProductNewPage"));
const AdminProductDetailPage = React.lazy(() => import("@/features/admin/products/pages/AdminProductDetailPage"));
const AdminInventoryPage = React.lazy(() => import("@/features/admin/inventory/pages/AdminInventoryPage"));
const AdminOrdersPage    = React.lazy(() => import("@/features/admin/orders/pages/AdminOrdersPage"));
const AdminOrderDetailPage = React.lazy(() => import("@/features/admin/orders/pages/AdminOrderDetailPage"));
const AdminCategoriesPage  = React.lazy(() => import("@/features/admin/catalog/pages/AdminCategoriesPage"));
const AdminBrandsPage      = React.lazy(() => import("@/features/admin/catalog/pages/AdminBrandsPage"));
const AdminAttributesPage  = React.lazy(() => import("@/features/admin/catalog/pages/AdminAttributesPage"));
const AdminDiscountsPage   = React.lazy(() => import("@/features/admin/catalog/pages/AdminDiscountsPage"));

const wrap = (el: React.ReactNode) => (
  <Suspense fallback={<LoaderFallback />}>{el}</Suspense>
);

export const adminRoutes = [
  {
    path: "/admin",
    element: (
      <RoleRoute roles={["admin"]}>
        {wrap(<AdminLayout />)}
      </RoleRoute>
    ),
    children: [
      { index: true,               element: wrap(<AdminDashboardPage />) },
      { path: "products",          element: wrap(<AdminProductsPage />) },
      { path: "products/new",      element: wrap(<AdminProductNewPage />) },
      { path: "products/:id",      element: wrap(<AdminProductDetailPage />) },
      { path: "inventory",         element: wrap(<AdminInventoryPage />) },
      { path: "orders",            element: wrap(<AdminOrdersPage />) },
      { path: "orders/:id",        element: wrap(<AdminOrderDetailPage />) },
      { path: "catalog/categories",element: wrap(<AdminCategoriesPage />) },
      { path: "catalog/brands",    element: wrap(<AdminBrandsPage />) },
      { path: "catalog/attributes",element: wrap(<AdminAttributesPage />) },
      { path: "catalog/discounts", element: wrap(<AdminDiscountsPage />) },
    ],
  },
];
```

> **Note:** `AdminLayout` uses `<Outlet />` from `react-router-dom` to render child pages.

---

## 4. Feature Module Structure

```
src/features/admin/
│
├── shared/
│   ├── layouts/
│   │   └── AdminLayout.tsx          ← sidebar + topbar + <Outlet/>
│   ├── components/
│   │   ├── AdminSidebar.tsx
│   │   ├── AdminTopbar.tsx
│   │   ├── StatCard.tsx
│   │   └── DataTable.tsx            ← reusable paginated table
│   └── hooks/
│       └── useAdminBreadcrumb.ts
│
├── dashboard/
│   ├── pages/
│   │   └── AdminDashboardPage.tsx
│   ├── hooks/
│   │   └── useDashboardStats.ts     ← aggregates from multiple APIs
│   └── components/
│       └── StatsGrid.tsx
│
├── products/
│   ├── pages/
│   │   ├── AdminProductsPage.tsx
│   │   ├── AdminProductNewPage.tsx
│   │   └── AdminProductDetailPage.tsx
│   ├── hooks/
│   │   ├── useAdminProducts.ts      ← list + pagination
│   │   ├── useAdminProduct.ts       ← single product detail
│   │   ├── useCreateProduct.ts
│   │   ├── useUpdateProduct.ts
│   │   ├── useDeleteProduct.ts
│   │   ├── useProductImages.ts
│   │   ├── useProductVariants.ts
│   │   └── useVariantMutations.ts
│   ├── components/
│   │   ├── ProductTable.tsx
│   │   ├── ProductForm.tsx          ← create / edit form
│   │   ├── VariantsTable.tsx
│   │   ├── VariantForm.tsx
│   │   ├── ImageUploader.tsx
│   │   └── ProductStatusBadge.tsx
│   └── services/
│       └── adminProductsApi.ts
│
├── inventory/
│   ├── pages/
│   │   └── AdminInventoryPage.tsx
│   ├── hooks/
│   │   ├── useInventoryItems.ts
│   │   ├── useLocations.ts
│   │   ├── useAdjustStock.ts
│   │   └── useUpsertInventoryItem.ts
│   ├── components/
│   │   ├── InventoryTable.tsx
│   │   ├── StockAdjustModal.tsx
│   │   └── LocationSelector.tsx
│   └── services/
│       └── adminInventoryApi.ts
│
├── orders/
│   ├── pages/
│   │   ├── AdminOrdersPage.tsx
│   │   └── AdminOrderDetailPage.tsx
│   ├── hooks/
│   │   ├── useAdminOrders.ts
│   │   └── useAdminOrderDetail.ts
│   ├── components/
│   │   ├── OrdersTable.tsx
│   │   ├── OrderStatusBadge.tsx
│   │   └── OrderDetailCard.tsx
│   └── services/
│       └── adminOrdersApi.ts
│
└── catalog/
    ├── pages/
    │   ├── AdminCategoriesPage.tsx
    │   ├── AdminBrandsPage.tsx
    │   ├── AdminAttributesPage.tsx
    │   └── AdminDiscountsPage.tsx
    ├── hooks/
    │   ├── useAdminCategories.ts
    │   ├── useAdminBrands.ts
    │   ├── useAdminAttributes.ts
    │   └── useAdminDiscounts.ts
    ├── components/
    │   ├── CategoryTree.tsx
    │   ├── BrandTable.tsx
    │   ├── AttributeValuesManager.tsx
    │   └── DiscountForm.tsx
    └── services/
        └── adminCatalogApi.ts
```

---

## 5. API Service Layer

All admin API services live under `src/features/admin/*/services/` and use the shared `http` client from `@/core/api`. The interceptor automatically attaches the Bearer token and unwraps `{ success, data, message }` envelopes.

### Base URL pattern

```
/api/v1/admin/catalog/...
```

The `http` client base URL already includes `/api/v1` (via `env.apiBaseUrl`). Services call relative paths like `/admin/catalog/products`.

---

## 6. Endpoint Reference — Dashboard

The dashboard aggregates data from existing endpoints. There is no dedicated `/admin/dashboard` endpoint — the UI assembles stats from calls below.

### 6.1 Quick Stats (compose from)

| Stat | Source Endpoint | Notes |
|---|---|---|
| Total products | `GET /catalog/products?page=1&page_size=1` → `total` | |
| Low stock variants | `GET /admin/catalog/inventory/...` | Not yet available — see §14 |
| Pending orders | `GET /orders?statuses=pending` | Currently user-scoped only — see §14 |
| Active categories | `GET /catalog/categories` | Count items |

---

## 7. Endpoint Reference — Products

### 7.1 List Products (Admin)

Uses the **public** list endpoint — it supports all required filters including `status=draft|archived`.

```
GET /api/v1/catalog/products
```

**Query Parameters**

| Param | Type | Default | Description |
|---|---|---|---|
| `page` | number | 1 | Page number |
| `page_size` | number | 20 | Items per page |
| `sort` | string | `new_arrivals` | `new_arrivals`, `trending`, `best_sellers`, `rating` |
| `status` | string | — | `active`, `draft`, `archived` |
| `brand_id` | number | — | Filter by brand |
| `category_id` | number | — | Filter by category |
| `is_featured` | boolean | — | Featured filter |
| `q` | string | — | Name search |
| `in_stock_only` | boolean | — | Stock filter |

**Response**

```ts
{
  items: AdminProductListItem[];
  total: number;
  page: number;
  size: number;
  sort: string;
}

type AdminProductListItem = {
  product_id: number;
  name: string;
  slug: string;
  primary_image: string | null;
  price: number;
  discount: number | null;
  rating: number | null;
  review_count: number | null;
  inventory_status: "in_stock" | "out_of_stock" | "low_stock";
  brand: string | null;
  category: string | null;
};
```

---

### 7.2 Get Product Detail

```
GET /api/v1/catalog/products/:id
```

**Query Parameters**

| Param | Type | Default | Description |
|---|---|---|---|
| `load_related` | boolean | false | Include categories, tags, attribute values |

**Response**

```ts
type AdminProductDetail = {
  id: number;
  name: string;
  slug: string;
  description: { String: string; Valid: boolean } | null;
  short_description: { String: string; Valid: boolean } | null;
  brand_id: { Int64: number; Valid: boolean } | null;
  base_price: number;
  status: "active" | "draft" | "archived";
  is_featured: boolean;
  is_digital: boolean;
  rating: number;
  review_count: number;
  images: ProductImage[];
  categories: { id: number; name: string }[];
  tags: { id: number; name: string }[];
  attribute_values: { id: number; name: string }[];
  variants: { id: number; name: string }[];
  created_at: string;
  updated_at: string;
};

type ProductImage = {
  id: number;
  product_id: number;
  image_url: string;
  is_primary: boolean;
  sort_order: number;
  created_at: string;
};
```

---

### 7.3 Create Product

```
POST /api/v1/admin/catalog/products
Content-Type: multipart/form-data
Authorization: Bearer <token>
```

> ⚠️ **Important:** This endpoint uses `multipart/form-data`. The JSON body must be sent as a **form field** named `data`. Image files are sent as `images[]` file fields.

**Form Fields**

| Field | Type | Required | Description |
|---|---|---|---|
| `data` | string (JSON) | ✅ | Stringified JSON of the product body (see below) |
| `images` | File[] | ✅ | At least one image file |

**`data` JSON Shape**

```ts
type CreateProductRequest = {
  name: string;                        // required, max 255 chars
  description?: string;
  short_description?: string;
  brand_id: number;                    // required, > 0
  base_price: number;                  // required, > 0
  status: "active" | "draft" | "archived";
  is_featured: boolean;
  is_digital: boolean;
  category_ids: number[];              // required, at least 1
  tag_names?: string[];
  attribute_value_ids?: number[];
  variants: VariantInput[];            // required, at least 1
  discount?: {
    discount_id?: number;
    name?: string;
    code?: string;
  };
};

type VariantInput = {
  sku: string;
  price: number;
  weight?: number;
  is_active?: boolean;
  attribute_value_ids?: number[];
  quantity: number;
  location_id?: number;
};
```

**Frontend Usage Example**

```ts
const formData = new FormData();
formData.append("data", JSON.stringify(productPayload));
images.forEach(file => formData.append("images", file));

await http.post("/admin/catalog/products", formData, {
  headers: { "Content-Type": "multipart/form-data" },
});
```

**Response** — `201 Created`

```ts
// Returns created product detail object (same shape as §7.2)
```

---

### 7.4 Update Product

```
PUT /api/v1/admin/catalog/products/:id
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body** (all fields optional)

```ts
type UpdateProductRequest = {
  name?: string;
  description?: string;
  short_description?: string;
  brand_id?: number;
  base_price?: number;
  status?: "active" | "draft" | "archived";
  is_featured?: boolean;
  is_digital?: boolean;
};
```

**Response** — `200 OK`

```ts
// Returns updated product object
```

---

### 7.5 Delete Product

```
DELETE /api/v1/admin/catalog/products/:id
Authorization: Bearer <token>
```

**Response** — `200 OK`

```ts
{ success: true, message: "product deleted", data: null }
```

---

### 7.6 Add Product Image

```
POST /api/v1/admin/catalog/products/:id/images
Content-Type: multipart/form-data
Authorization: Bearer <token>
```

**Form Fields**

| Field | Type | Required | Description |
|---|---|---|---|
| `image` | File | ✅ | Single image file |
| `is_primary` | string | — | `"true"` or `"false"` |
| `sort_order` | string | — | Numeric string, default `"0"` |

**Response** — `201 Created`

```ts
type ProductImage = {
  id: number;
  product_id: number;
  image_url: string;
  is_primary: boolean;
  sort_order: number;
  created_at: string;
};
```

---

### 7.7 Delete Product Image

```
DELETE /api/v1/admin/catalog/products/:id/images/:image_id
Authorization: Bearer <token>
```

**Response** — `200 OK`

---

### 7.8 Set Primary Image

```
PUT /api/v1/admin/catalog/products/:id/images/:image_id/primary
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body**

```ts
{ image_id: number }
```

**Response** — `200 OK`

---

### 7.9 Manage Product Categories

**Add category to product**

```
POST /api/v1/admin/catalog/products/:id/categories
Content-Type: application/json
Authorization: Bearer <token>
```

```ts
// Request body — inferred from handler (CatalogHandler.AddProductCategory)
{ category_id: number }
```

**Remove category from product**

```
DELETE /api/v1/admin/catalog/products/:id/categories/:cat_id
Authorization: Bearer <token>
```

---

### 7.10 Manage Product Tags

**Set all tags (replaces existing)**

```
PUT /api/v1/admin/catalog/products/:id/tags
Content-Type: application/json
Authorization: Bearer <token>
```

```ts
{ tag_ids: number[] }
```

**Add single tag**

```
POST /api/v1/admin/catalog/products/:id/tags
Content-Type: application/json
Authorization: Bearer <token>
```

```ts
{ tag_id: number }
```

**Remove tag**

```
DELETE /api/v1/admin/catalog/products/:id/tags/:tag_id
Authorization: Bearer <token>
```

---

### 7.11 Set Product Attribute Values

```
PUT /api/v1/admin/catalog/products/:id/attribute-values
Content-Type: application/json
Authorization: Bearer <token>
```

```ts
{ attribute_value_ids: number[] }
```

---

### 7.12 Create Variant

```
POST /api/v1/admin/catalog/products/:id/variants
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body**

```ts
type CreateVariantRequest = {
  sku: string;           // max 100 chars
  price: number;         // > 0
  weight?: number;
  is_active?: boolean;
  attribute_value_ids?: number[];
};
```

**Response** — `201 Created`

```ts
type Variant = {
  id: number;
  product_id: number;
  sku: string;
  price: number;
  weight: { Float64: number; Valid: boolean };
  is_active: boolean;
  created_at: string;
};
```

---

### 7.13 Update Variant

```
PUT /api/v1/admin/catalog/products/:id/variants/:variant_id
Content-Type: application/json
Authorization: Bearer <token>
```

```ts
type UpdateVariantRequest = {
  sku?: string;
  price?: number;
  weight?: number;
  is_active?: boolean;
};
```

---

### 7.14 Delete Variant

```
DELETE /api/v1/admin/catalog/products/:id/variants/:variant_id
Authorization: Bearer <token>
```

---

### 7.15 Set Variant Attribute Values

```
PUT /api/v1/admin/catalog/products/:id/variants/:variant_id/attribute-values
Content-Type: application/json
Authorization: Bearer <token>
```

```ts
{ attribute_value_ids: number[] }
```

---

## 8. Endpoint Reference — Inventory

### 8.1 Get Variant Inventory (per location breakdown)

```
GET /api/v1/catalog/inventory/variants/:variant_id
```

**Response**

```ts
type VariantInventoryRow[] = {
  id: number;
  variant_id: number;
  location_id: number;
  available_qty: number;
  reserved_qty: number;
  incoming_qty: number;
  updated_at: string;
  location_name?: string;
  location_code?: { String: string; Valid: boolean };
}[];
```

---

### 8.2 Get Stock Summary (aggregated across all locations)

```
GET /api/v1/catalog/inventory/variants/:variant_id/stock
```

**Response**

```ts
type VariantStockSummary = {
  variant_id: number;
  available_qty: number;
  reserved_qty: number;
  incoming_qty: number;
};
```

---

### 8.3 Upsert Inventory Item

Creates or fully replaces inventory quantities for a variant at a specific location.

```
PUT /api/v1/admin/catalog/inventory/items
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body**

```ts
type UpsertInventoryItemRequest = {
  variant_id: number;
  location_id: number;
  available_qty: number;   // >= 0
  reserved_qty: number;    // >= 0
  incoming_qty: number;    // >= 0
};
```

**Response** — `200 OK`

```ts
// Returns the updated inventory item row
```

---

### 8.4 Adjust Available Stock (delta)

```
PUT /api/v1/admin/catalog/inventory/variants/:variant_id/locations/:location_id/adjust
Content-Type: application/json
Authorization: Bearer <token>
```

**Request Body**

```ts
type AdjustQtyRequest = {
  delta: number;   // positive to add, negative to subtract
};
```

**Response** — `200 OK`

---

### 8.5 Reserve Stock

```
PUT /api/v1/admin/catalog/inventory/variants/:variant_id/locations/:location_id/reserve
Authorization: Bearer <token>
```

**Query Parameters**

| Param | Type | Required |
|---|---|---|
| `qty` | number | ✅ (> 0) |

---

### 8.6 Release Stock

```
PUT /api/v1/admin/catalog/inventory/variants/:variant_id/locations/:location_id/release
Authorization: Bearer <token>
```

**Query Parameters**

| Param | Type | Required |
|---|---|---|
| `qty` | number | ✅ (> 0) |

---

### 8.7 Delete Inventory Item

```
DELETE /api/v1/admin/catalog/inventory/variants/:variant_id/locations/:location_id
Authorization: Bearer <token>
```

---

### 8.8 List Inventory Locations

```
GET /api/v1/catalog/inventory/locations
```

**Query Parameters**

| Param | Type | Description |
|---|---|---|
| `active_only` | boolean | Filter to active locations only |

**Response**

```ts
type Location[] = {
  id: number;
  name: string;
  location_code: { String: string; Valid: boolean };
  is_active: boolean;
  created_at: string;
  updated_at: string;
}[];
```

---

### 8.9 Create Location

```
POST /api/v1/admin/catalog/inventory/locations
Content-Type: application/json
Authorization: Bearer <token>
```

```ts
type CreateLocationRequest = {
  name: string;            // max 150 chars
  location_code?: string;  // max 50 chars
  is_active?: boolean;
};
```

---

### 8.10 Update Location

```
PUT /api/v1/admin/catalog/inventory/locations/:id
Content-Type: application/json
Authorization: Bearer <token>
```

```ts
type UpdateLocationRequest = {
  name?: string;
  location_code?: string;
  is_active?: boolean;
};
```

---

### 8.11 Delete Location

```
DELETE /api/v1/admin/catalog/inventory/locations/:id
Authorization: Bearer <token>
```

---

## 9. Endpoint Reference — Orders

> ⚠️ **Current state:** The backend `GET /orders` endpoint is **user-scoped** — it returns only the authenticated user's orders (filtered by `identity_id`). A true admin-scoped `GET /admin/orders` that returns **all users' orders** does not yet exist. See §14.
>
> **Workaround for Phase 1:** Admin can use `GET /orders?user_id=<id>` if the handler supports that filter, or the backend team adds the admin orders endpoint (proposed in §14).

### 9.1 List Orders (current user-scoped endpoint)

```
GET /api/v1/orders
Authorization: Bearer <token>
```

**Query Parameters**

| Param | Type | Description |
|---|---|---|
| `limit` | number | Page size |
| `offset` | number | Pagination offset |
| `sort_by` | string | Sort field |
| `sort_desc` | boolean | Descending sort |
| `statuses` | string | Comma-separated statuses e.g. `pending,processing` |
| `created_from` | string | RFC3339 date |
| `created_to` | string | RFC3339 date |
| `order_number` | string | Filter by order number |

**Response**

```ts
type OrdersListData = {
  offset: number;
  orders: OrderListRow[];
};

type OrderListRow = {
  ID: number;
  UserID: number;
  CartID: number | null;
  OrderNumber: string;
  Status: string;
  Subtotal: number;
  DiscountAmount: number;
  TaxAmount: number;
  ShippingFee: number;
  TotalAmount: number;
  Currency: string;
  Shipping: OrderShipping;
  CreatedAt: string;
  UpdatedAt: string;
  Items: OrderItem[] | null;
};

type OrderShipping = {
  full_name: string;
  phone: string;
  country: string;
  county?: string;
  city: string;
  area?: string;
  postal_code?: string;
  address_line_1: string;
  address_line_2?: string;
};

type OrderItem = {
  ID: number;
  OrderID: number;
  ProductID: number;
  VariantID: number;
  ProductName: string;
  ProductSlug: string;
  VariantSKU: string | null;
  VariantName: string | null;
  ImageURL: string | null;
  UnitPrice: number;
  Quantity: number;
  DiscountAmount: number;
  TaxRate: number;
  TotalPrice: number;
  Currency: string;
};
```

---

### 9.2 Get Order Detail

```
GET /api/v1/orders/details?id=<order_id>
Authorization: Bearer <token>
```

**Response**

```ts
{ order: OrderListRow }
```

---

## 10. Endpoint Reference — Users / Admins

### 10.1 Create Admin (Super-Admin Only)

```
POST /api/v1/admin/admins
Content-Type: application/json
Authorization: Bearer <token>   ← must have super_admin role
```

```ts
// Request body — matches backend CreateAdmin handler
{
  email: string;
  full_name: string;
  password: string;
}
```

---

### 10.2 List Admins (Super-Admin Only)

```
GET /api/v1/admin/admins
Authorization: Bearer <token>
```

**Response** — array of admin user objects.

---

### 10.3 Deactivate Admin (Super-Admin Only)

```
DELETE /api/v1/admin/admins/:id
Authorization: Bearer <token>
```

---

## 11. Endpoint Reference — Catalog Metadata

### 11.1 Categories

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/catalog/categories` | Public | List all (`active_only` param) |
| `GET` | `/catalog/categories/:id` | Public | Single category |
| `GET` | `/catalog/categories/:id/tree` | Public | Full tree from node |
| `GET` | `/catalog/categories/:id/descendants` | Public | All descendants |
| `POST` | `/admin/catalog/categories` | Admin | Create category |
| `PUT` | `/admin/catalog/categories/:id` | Admin | Update category |
| `DELETE` | `/admin/catalog/categories/:id` | Admin | Delete category |

**Create/Update Request**

```ts
type CategoryRequest = {
  name: string;
  slug?: string;
  image_url?: string;
  parent_id?: number;
  is_active?: boolean;
};
```

---

### 11.2 Brands

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/catalog/brands` | Public | List all |
| `GET` | `/catalog/brands/:id` | Public | Single brand |
| `POST` | `/admin/catalog/brands` | Admin | Create |
| `PUT` | `/admin/catalog/brands/:id` | Admin | Update |
| `DELETE` | `/admin/catalog/brands/:id` | Admin | Delete |

**Create/Update Request**

```ts
type BrandRequest = {
  name: string;
  slug?: string;
  image_url?: string;
  is_active?: boolean;
};
```

---

### 11.3 Attributes & Values

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/catalog/attributes` | Public | List attributes |
| `GET` | `/catalog/attributes/:id` | Public | Single attribute |
| `GET` | `/catalog/attributes/:id/values` | Public | List values for attribute |
| `POST` | `/admin/catalog/attributes` | Admin | Create attribute |
| `PUT` | `/admin/catalog/attributes/:id` | Admin | Update attribute |
| `DELETE` | `/admin/catalog/attributes/:id` | Admin | Delete attribute |
| `POST` | `/admin/catalog/attributes/:id/values` | Admin | Add attribute value |
| `DELETE` | `/admin/catalog/attributes/:id/values/:value_id` | Admin | Delete attribute value |

**Attribute Request**

```ts
type AttributeRequest = {
  name: string;
  slug?: string;
};
```

**Attribute Value Request**

```ts
type AttributeValueRequest = {
  value: string;
};
```

---

### 11.4 Tags

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/catalog/tags` | Public | List all tags |
| `GET` | `/catalog/tags/:id` | Public | Single tag |

> Tags are created implicitly via `tag_names[]` on product create. No standalone admin create-tag endpoint exists.

---

### 11.5 Discounts

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/admin/catalog/discounts` | Admin | List all discounts |
| `GET` | `/admin/catalog/discounts/:id` | Admin | Single discount |
| `POST` | `/admin/catalog/discounts` | Admin | Create discount |
| `PUT` | `/admin/catalog/discounts/:id` | Admin | Update discount |
| `DELETE` | `/admin/catalog/discounts/:id` | Admin | Delete discount |
| `PUT` | `/admin/catalog/discounts/:id/targets` | Admin | Set discount targets (products/categories) |

---

## 12. Shared Types Reference

```ts
// src/features/admin/shared/types/index.ts

export type AdminApiEnvelope<T> = {
  success: boolean;
  message: string;
  data: T;
};

export type PaginatedResponse<T> = {
  items: T[];
  total: number;
  page: number;
  size: number;
};

export type ProductStatus = "active" | "draft" | "archived";

export type InventoryStatus = "in_stock" | "out_of_stock" | "low_stock";

export type OrderStatus =
  | "pending"
  | "processing"
  | "shipped"
  | "delivered"
  | "cancelled"
  | "refunded";
```

---

## 13. State Management Design

### React Query Keys (suggested)

```ts
// src/features/admin/shared/queryKeys.ts

export const adminKeys = {
  products: {
    all:    ["admin", "products"] as const,
    list:   (params: object) => ["admin", "products", "list", params] as const,
    detail: (id: number)     => ["admin", "products", id] as const,
  },
  inventory: {
    all:      ["admin", "inventory"] as const,
    variant:  (id: number) => ["admin", "inventory", "variant", id] as const,
    locations:             ["admin", "inventory", "locations"] as const,
  },
  orders: {
    all:    ["admin", "orders"] as const,
    list:   (params: object) => ["admin", "orders", "list", params] as const,
    detail: (id: number)     => ["admin", "orders", id] as const,
  },
  catalog: {
    categories: ["admin", "catalog", "categories"] as const,
    brands:     ["admin", "catalog", "brands"] as const,
    attributes: ["admin", "catalog", "attributes"] as const,
    discounts:  ["admin", "catalog", "discounts"] as const,
  },
};
```

### Zustand Store — Admin UI State

```ts
// src/features/admin/shared/store/adminUiStore.ts
// Light store for UI-only state (sidebar collapse, selected rows, etc.)

type AdminUiState = {
  sidebarCollapsed: boolean;
  toggleSidebar: () => void;
};
```

> All server state (products, orders, inventory) should live in **React Query**, not Zustand.

---

## 14. Missing Backend Endpoints (Proposed)

These endpoints are needed for a complete admin experience. They should be flagged to the backend team but are **not blockers** for Phase 1 — the UI can be scaffolded with loading/empty states until they land.

### 14.1 Admin-Scoped Order Management

```
GET  /api/v1/admin/orders                 ← all users' orders with filters
GET  /api/v1/admin/orders/:id             ← order detail by path param
PUT  /api/v1/admin/orders/:id/status      ← update order status
```

**Proposed Request (status update)**

```ts
{ status: OrderStatus; note?: string }
```

### 14.2 Inventory Overview / Low-Stock Report

```
GET /api/v1/admin/catalog/inventory/overview
  → { total_variants, low_stock_count, out_of_stock_count }

GET /api/v1/admin/catalog/inventory/low-stock?threshold=5
  → VariantInventoryRow[]  // variants where available_qty <= threshold
```

### 14.3 User Listing (Admin)

```
GET /api/v1/admin/users               ← paginated user list
GET /api/v1/admin/users/:id           ← single user with orders
PUT /api/v1/admin/users/:id/status    ← activate / deactivate user
```

### 14.4 Dashboard Stats Aggregate

```
GET /api/v1/admin/analytics/summary
  → {
      total_products: number;
      active_products: number;
      total_orders: number;
      pending_orders: number;
      revenue_today: number;
      revenue_this_month: number;
    }
```

### 14.5 Schema Improvements (proposed, non-breaking)

| Improvement | Rationale |
|---|---|
| `UNIQUE` index on `variants.sku` | Prevent duplicate SKUs at DB level |
| `inventory_audit_logs` table | Track stock adjustments with actor, delta, reason, timestamp |
| Index on `orders.status` | Speed up admin order filtering by status |
| Index on `orders.created_at` | Speed up date-range queries |
| `products.archived_at` timestamp | Soft-delete with timestamp for audit trail |

---

*End of Zentora Admin Architecture Proposal — Frontend Edition*
*Generated from codebase analysis: `internal/app/router.go`, `internal/handlers/catalog/`, `internal/domain/`, `src/app/router/`, `src/core/`, `src/features/auth/`*


update the documetation for orders (its actually not missing ) note that the endpoint for orders (shared by user can also be used to get orders on admin side)
this were the handlers (and filters),


func (h *Handler) GetOrderByID(c *gin.Context) {
	id := c.Query("id")
	if id == "" {
		response.Error(c, http.StatusBadRequest, "invalid order id", nil)
		return
	}
	idInt64, err := strconv.ParseInt(id, 10, 64)
	if err != nil {
		response.Error(c, http.StatusBadRequest, "invalid order id", err)
		return
	}
	o, err := h.orders.GetOrderByID(c.Request.Context(), idInt64)
	if err != nil {
		handleErr(c, err)
		return
	}
	response.Success(c, http.StatusOK, "order details", gin.H{"order": o})
}

func (h *Handler) ListOrders(c *gin.Context) {
	var filter orderdomain.ListFilter

	// OrderID
	if v := c.Query("order_id"); v != "" {
		id, err := strconv.ParseInt(v, 10, 64)
		if err != nil {
			response.Error(c, http.StatusBadRequest, "invalid order_id", err)
			return
		}
		filter.OrderID = &id
	}

	// OrderNumber
	if v := c.Query("order_number"); v != "" {
		filter.OrderNumber = &v
	}

	// UserID
	if v := c.Query("user_id"); v != "" {
		id, err := strconv.ParseInt(v, 10, 64)
		if err != nil {
			response.Error(c, http.StatusBadRequest, "invalid user_id", err)
			return
		}
		filter.UserID = &id
	}

	// CartID
	if v := c.Query("cart_id"); v != "" {
		id, err := strconv.ParseInt(v, 10, 64)
		if err != nil {
			response.Error(c, http.StatusBadRequest, "invalid cart_id", err)
			return
		}
		filter.CartID = &id
	}

	// Statuses (comma separated)
	if v := c.Query("statuses"); v != "" {
		statuses := strings.Split(v, ",")
		for _, s := range statuses {
			filter.Statuses = append(filter.Statuses, orderdomain.OrderStatus(strings.TrimSpace(s)))
		}
	}

	// CreatedFrom
	if v := c.Query("created_from"); v != "" {
		t, err := time.Parse(time.RFC3339, v)
		if err != nil {
			response.Error(c, http.StatusBadRequest, "invalid created_from", err)
			return
		}
		filter.CreatedFrom = &t
	}

	// CreatedTo
	if v := c.Query("created_to"); v != "" {
		t, err := time.Parse(time.RFC3339, v)
		if err != nil {
			response.Error(c, http.StatusBadRequest, "invalid created_to", err)
			return
		}
		filter.CreatedTo = &t
	}

	// Limit
	if v := c.DefaultQuery("limit", "20"); v != "" {
		limit, err := strconv.Atoi(v)
		if err != nil {
			response.Error(c, http.StatusBadRequest, "invalid limit", err)
			return
		}
		filter.Limit = limit
	}

	// Offset
	if v := c.DefaultQuery("offset", "0"); v != "" {
		offset, err := strconv.Atoi(v)
		if err != nil {
			response.Error(c, http.StatusBadRequest, "invalid offset", err)
			return
		}
		filter.Offset = offset
	}

	// SortBy
	filter.SortBy = c.DefaultQuery("sort_by", "created_at")

	// SortDesc
	if v := c.DefaultQuery("sort_desc", "true"); v != "" {
		desc, err := strconv.ParseBool(v)
		if err != nil {
			response.Error(c, http.StatusBadRequest, "invalid sort_desc", err)
			return
		}
		filter.SortDesc = desc
	}

	orders, offset, err := h.orders.ListOrders(c.Request.Context(), filter)
	if err != nil {
		response.Error(c, http.StatusInternalServerError, "failed to list orders", err)
		return
	}

	response.Success(c, http.StatusOK, "orders listed", gin.H{"orders": orders, "offset": offset})
}


```go
		ordersProtected.GET("", h.OrderHandler.ListOrders) // list user's orders with filters + pagination
		ordersProtected.GET("/details", h.OrderHandler.GetOrderByID) // order details by ID (with items)
```


9. Endpoint Reference — Orders (Updated)
Overview

The Zentora backend uses a shared orders endpoint for both users and administrators.

GET /api/v1/orders

The endpoint supports rich filtering, allowing administrators to query orders across the system by specifying filters such as:

user_id
order_id
order_number
statuses
created_from
created_to

Because of this flexible filtering design, a separate /admin/orders endpoint is not required.

Administrators can retrieve any order by applying filters.

9.1 List Orders
GET /api/v1/orders
Authorization: Bearer <token>
Query Parameters
Param	Type	Description
order_id	number	Retrieve specific order by ID
order_number	string	Lookup order by number
user_id	number	Filter orders belonging to a specific user
cart_id	number	Filter by cart
statuses	string	Comma-separated list (pending,processing,shipped)
created_from	string	RFC3339 date
created_to	string	RFC3339 date
limit	number	Pagination limit (default 20)
offset	number	Pagination offset
sort_by	string	Sort field (default created_at)
sort_desc	boolean	Descending sort (default true)
Example — Admin retrieving all pending orders
GET /api/v1/orders?statuses=pending&limit=50
Example — Admin retrieving a user's orders
GET /api/v1/orders?user_id=42
Response
{
  offset: number;
  orders: OrderListRow[];
}
Order Object
type OrderListRow = {
  ID: number;
  UserID: number;
  CartID: number | null;
  OrderNumber: string;
  Status: string;

  Subtotal: number;
  DiscountAmount: number;
  TaxAmount: number;
  ShippingFee: number;
  TotalAmount: number;
  Currency: string;

  Shipping: OrderShipping;

  CreatedAt: string;
  UpdatedAt: string;

  Items: OrderItem[] | null;
};
9.2 Get Order Detail
GET /api/v1/orders/details?id=<order_id>
Authorization: Bearer <token>
Response
{
  order: OrderListRow
}
Implementation Notes

The backend handler internally supports all filters through:

orderdomain.ListFilter

Which includes:

OrderID
OrderNumber
UserID
CartID
Statuses
CreatedFrom
CreatedTo
Limit
Offset
SortBy
SortDesc

This design allows a single endpoint to serve both user and admin use cases.