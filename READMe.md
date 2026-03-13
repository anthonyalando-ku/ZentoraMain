Zentora Workspace

Submodules:
- `Backend` → Go API
- `UI` → React frontend

Backend exposes APIs consumed by the frontend.

## Frontend ↔ Backend Integration Plan

This document captures the current integration architecture for the user-facing e-commerce flows only. It intentionally excludes admin, notification, websocket, and auth implementation details.

### Scope

Focus areas:
- product catalog
- discovery feeds
- cart
- wishlist
- orders
- search
- user addresses
- reviews as a product-detail dependency

Hard constraints:
- frontend routes must use `/products/:slug`, never numeric product IDs
- guest users keep cart locally and place orders through `POST /api/v1/orders/guest`
- logged-in users sync cart and wishlist with backend and place orders through authenticated endpoints

## 1. Backend endpoint extraction

Source of truth: `Backend/internal/app/router.go`

### Catalog

Public routes:
- `GET /api/v1/catalog/categories`
  - query support: `active_only`, `parent_id`
- `GET /api/v1/catalog/categories/:id`
- `GET /api/v1/catalog/categories/:id/tree`
- `GET /api/v1/catalog/categories/:id/descendants`
- `GET /api/v1/catalog/brands`
- `GET /api/v1/catalog/brands/:id`
- `GET /api/v1/catalog/tags`
- `GET /api/v1/catalog/tags/:id`
- `GET /api/v1/catalog/attributes`
- `GET /api/v1/catalog/attributes/:id`
- `GET /api/v1/catalog/attributes/:id/values`
- `GET /api/v1/catalog/products`
  - query support from `Backend/internal/handlers/catalog/product.go`:
    - `page`
    - `page_size`
    - `status`
    - `brand_id`
    - `category_id`
    - `is_featured`
    - `q`
- `GET /api/v1/catalog/products/slug/:slug`
- `GET /api/v1/catalog/products/:id`
- `GET /api/v1/catalog/products/:id/images`
- `GET /api/v1/catalog/products/:id/tags`
- `GET /api/v1/catalog/products/:id/categories`
- `GET /api/v1/catalog/products/:id/attribute-values`
- `GET /api/v1/catalog/products/:id/variants`
  - query support: `active_only`
- `GET /api/v1/catalog/products/:id/variants/:variant_id`
- `GET /api/v1/catalog/products/:id/variants/:variant_id/attribute-values`
- `GET /api/v1/catalog/inventory/variants/:variant_id`
- `GET /api/v1/catalog/inventory/variants/:variant_id/stock`
- `GET /api/v1/catalog/inventory/locations`
- `GET /api/v1/catalog/inventory/locations/:id`

Important response notes:
- discovery cards already expose `slug`, image, price, discount, rating, review count, inventory status, brand, and category
- slug detail endpoint returns a product payload suitable for the product page and should remain the canonical frontend detail lookup

### Discovery

Public routes with optional auth:
- `GET /api/v1/discovery/feed`
- `GET /api/v1/discovery/search`
- `GET /api/v1/discovery/suggest`
- `POST /api/v1/discovery/search/events`
- `POST /api/v1/discovery/search/clicks`

Supported feed types from `Backend/internal/domain/discovery/types.go`:
- `trending`
- `best_sellers`
- `recommended`
- `deals`
- `new_arrivals`
- `highly_rated`
- `most_wishlisted`
- `also_viewed`
- `featured`
- `editorial`
- `search`
- `category`

Feed query parameters from `Backend/internal/handlers/discovery/handler.go`:
- `feed_type`
- `limit`
- `session_id`
- `query`
- `category_id`
- `brand_ids`
- `tag_ids`
- `variant_attribute_value_ids`
- `price_min`
- `price_max`
- `min_rating`
- `discount_only`
- `in_stock_only`

Behavioral notes:
- `recommended` requires an authenticated user
- `also_viewed` requires either authenticated user context or `session_id`
- `category` requires `category_id`
- `search` requires `query`

### Cart

Authenticated routes:
- `GET /api/v1/me/cart`
- `POST /api/v1/me/cart/items`
  - body:
    - `product_id`
    - `variant_id`
    - `quantity`
    - `price_at_added`
- `DELETE /api/v1/me/cart/items/:id`

Important cart model details:
- backend cart items are keyed by cart item ID for deletion and variant ID for uniqueness
- variants are required for server-side cart writes

### Wishlist

Authenticated routes:
- `GET /api/v1/me/wishlist`
- `POST /api/v1/me/wishlist/items`
  - body:
    - `product_id`
    - `variant_id`
- `DELETE /api/v1/me/wishlist/items`
  - query:
    - `product_id`
    - `variant_id`
- `DELETE /api/v1/me/wishlist`

### Orders

Public:
- `POST /api/v1/orders/guest`

Authenticated:
- `POST /api/v1/orders`
- `GET /api/v1/orders`
- `GET /api/v1/orders/details`

Order payload notes from `Backend/internal/domain/order/dto.go`:
- guest order body:
  - `items[]` with `product_id`, `variant_id`, `quantity`
  - `shipping`
  - optional `delivery_method`
  - optional `payment_method`
- logged-in order body:
  - either `cart_id` or `items[]`
  - optional `address_id` to choose saved address, otherwise backend default address
  - optional `delivery_method`
  - optional `payment_method`
- currently supported payment method constant:
  - `pay_on_delivery`

Order listing query support from `Backend/internal/handlers/order/handler.go`:
- `order_id`
- `order_number`
- `user_id`
- `cart_id`
- `statuses`
- `created_from`
- `created_to`
- `limit`
- `offset`
- `sort_by`
- `sort_desc`

### User / Addresses

Authenticated routes:
- `GET /api/v1/me/addresses`
- `POST /api/v1/me/addresses`
- `GET /api/v1/me/addresses/:id`
- `PUT /api/v1/me/addresses/:id`
- `DELETE /api/v1/me/addresses/:id`
- `PUT /api/v1/me/addresses/:id/default`

Address payload fields from `Backend/internal/domain/user/dto.go`:
- `full_name`
- `phone_number`
- `country`
- `county`
- `city`
- `area`
- `postal_code`
- `address_line_1`
- `address_line_2`
- `is_default`

### Reviews

Authenticated write route:
- `POST /api/v1/me/reviews`

Review creation fields:
- `product_id`
- `order_item_id`
- `rating`
- `comment`

Important dependency note:
- `Backend/internal/handlers/review/handler.go` contains a public `ListProductReviews` handler, but `Backend/internal/app/router.go` does not currently expose a matching GET route. Product-detail review display therefore depends on a backend router addition in a later implementation phase.

### Search

Discovery-backed search endpoints:
- `GET /api/v1/discovery/search`
- `GET /api/v1/discovery/suggest`
- `POST /api/v1/discovery/search/events`
- `POST /api/v1/discovery/search/clicks`

Tracking payloads:
- search event:
  - `query`
  - `session_id`
  - `result_count`
  - `results[]` with `product_id`, `position`, `score`
- click event:
  - `search_event_id`
  - `product_id`
  - `position`
  - `session_id`

## 2. Frontend structure analysis

Primary source files:
- router: `UI/src/app/router/publicRoutes.tsx`
- layout shell: `UI/src/shared/layouts/MainLayout.tsx`
- API client: `UI/src/core/api/http.ts`
- response unwrapping and auth injection: `UI/src/core/api/interceptors.ts`

### Current pages

- Homepage
  - `UI/src/features/public/home/pages/HomePage.tsx`
- Products page
  - `UI/src/features/products/pages/ProductsPage.tsx`
- Product detail page
  - `UI/src/features/products/pages/ProductDetailPage.tsx`
- Cart page
  - `UI/src/features/cart/pages/CartPage.tsx`
- Checkout page
  - `UI/src/features/checkout/pages/CheckoutPage.tsx`
- Account page
  - `UI/src/features/account/pages/AccountDashboardPage.tsx`

### Shared components and layout pieces that affect integration

- product card:
  - `UI/src/features/products/components/ProductCard.tsx`
- desktop search:
  - `UI/src/shared/layouts/components/HeaderSearch.tsx`
- mobile search:
  - `UI/src/shared/layouts/components/MobileMenu.tsx`
- header shell:
  - `UI/src/shared/layouts/components/Header.tsx`

### Existing data/state infrastructure

- axios client already exists and unwraps `{ success, message, data }` responses
- auth persistence already exists in Zustand:
  - `UI/src/features/auth/store/authStore.ts`
- cart persistence currently exists only as a local Zustand store:
  - `UI/src/features/cart/store/cartStore.ts`
- TanStack React Query is available but not yet used for catalog/discovery/cart data

### Mock-data usage that must be replaced

Source:
- `UI/src/shared/constants/mockProducts.ts`

Current mock consumers:
- homepage uses `mockProducts` and `mockCategories`
- products page filters `mockProducts` and `mockCategories`
- product detail page resolves product by slug from `mockProducts`
- account page uses inline `mockOrders`
- cart/checkout rely on cart store items that embed full mock `Product` records

### Current frontend gaps relevant to integration

- no catalog/discovery API modules yet
- no page-specific React Query hooks for categories, feeds, products, cart, wishlist, orders, or addresses
- header and mobile search inputs are presentational only
- homepage product cards still expose add-to-cart, which conflicts with the requirement to defer cart actions until variant selection on product detail
- cart store is product-ID based, while real backend cart operations are variant-based
- account routing currently exposes only `/account`; deeper account sections in the UI are placeholders

## 3. Page-to-API mapping

Each page section below is organized the same way:
- `Components/pages to wire` identifies the current frontend files that will carry the integration work
- `APIs` identifies the backend endpoints that should drive that surface
- `UI mapping` explains the user-visible behavior and constraints that the implementation must preserve

### Homepage

Components/pages to wire:
- `HomePage.tsx`
- `ProductCard.tsx`
- `MainLayout.tsx`
- `HeaderSearch.tsx`
- `MobileMenu.tsx`

APIs:
- `GET /api/v1/catalog/categories`
  - powers navigation category menu
  - powers shop-by-category grid
- `GET /api/v1/discovery/feed?feed_type=<type>&limit=8`
  - powers homepage discovery sections
  - use feed types such as `trending`, `best_sellers`, `recommended`, `deals`, `new_arrivals`, `highly_rated`, `most_wishlisted`, `also_viewed`, `featured`, `editorial`

UI mapping:
- keep hero/banner placeholder content for now
- hide any discovery section whose `items` array is empty
- render 4-5 cards per row depending on breakpoint
- each section header needs a title and “show more” link to `/products?feed_type=<type>`
- homepage cards should show image, name, price, discount, wishlist toggle, and WhatsApp CTA
- homepage cards should not show add-to-cart

### Products Page

Components/pages to wire:
- `ProductsPage.tsx`
- `ProductCard.tsx`

Primary APIs:
- `GET /api/v1/discovery/feed`
  - preferred when user is browsing a feed-based landing page, deals page, category discovery section, or filtered search-like listing
- `GET /api/v1/catalog/products`
  - fallback/general catalog browsing endpoint for straightforward catalog listings
- filter metadata helpers:
  - `GET /api/v1/catalog/categories`
  - `GET /api/v1/catalog/brands`
  - `GET /api/v1/catalog/attributes`
  - `GET /api/v1/catalog/attributes/:id/values`

Recommended mapping:
- use discovery for feed-driven and richer filtered experiences
- use catalog products for simple all-products browsing if discovery feed type is absent

UI mapping:
- current left-column filters should be rebuilt from backend-supported filters, not hardcoded category unions
- current pagination controls should map to either:
  - discovery `limit` plus future cursor/page adapter in frontend state, or
  - catalog `page` / `page_size`
- cards on this page must link to detail only and must not add directly to cart

### Product Detail Page

Components/pages to wire:
- `ProductDetailPage.tsx`
- `ProductCard.tsx` for related products only if related data is still surfaced there

APIs:
- `GET /api/v1/catalog/products/slug/:slug`
- `GET /api/v1/catalog/products/:id/variants`
  - or consume variants if slug detail already includes enough embedded variant data
- `GET /api/v1/catalog/inventory/variants/:variant_id/stock`
- optional enrichments:
  - `GET /api/v1/catalog/products/:id/categories`
  - `GET /api/v1/catalog/products/:id/tags`
  - `GET /api/v1/catalog/products/:id/images`
  - `GET /api/v1/discovery/feed?feed_type=category&category_id=<id>&limit=4` for related products if needed

Cart/Wishlist/Review actions:
- guest cart: local cart store only
- logged-in cart: `POST /api/v1/me/cart/items` and `GET /api/v1/me/cart`
- logged-in wishlist: `POST /api/v1/me/wishlist/items`, `DELETE /api/v1/me/wishlist/items`
- logged-in reviews later: `POST /api/v1/me/reviews`

UI mapping:
- route stays `/products/:slug`
- select the first available variant by default
- add-to-cart and buy-now both require a selected variant
- if the selected variant already exists in cart, replace add-to-cart with quantity controls
- add WhatsApp purchase CTA alongside buy-now

### Cart Page

Components/pages to wire:
- `CartPage.tsx`
- `cartStore.ts`

APIs:
- guest:
  - no backend cart endpoints; use local persisted cart
- logged-in:
  - `GET /api/v1/me/cart`
  - `POST /api/v1/me/cart/items`
  - `DELETE /api/v1/me/cart/items/:id`

UI mapping:
- merge strategy is required when a guest signs in and already has local cart items
- quantity updates for authenticated carts should write the full desired quantity through `POST /me/cart/items`
- delete actions must use backend cart item ID, not product ID
- expose both checkout and WhatsApp order actions

### Checkout Page

Components/pages to wire:
- `CheckoutPage.tsx`
- `cartStore.ts`
- future address management subcomponents inside checkout

APIs:
- guest:
  - `POST /api/v1/orders/guest`
- logged-in:
  - `GET /api/v1/me/addresses`
  - `POST /api/v1/me/addresses`
  - `PUT /api/v1/me/addresses/:id`
  - `POST /api/v1/orders`

UI mapping:
- guest checkout keeps shipping form fields visible and posts `shipping`
- logged-in checkout should show saved addresses first, with add/edit address actions
- payment choices:
  - enable pay on delivery
  - show M-Pesa as disabled “coming soon”
- place-order payload should submit either `cart_id` or explicit `items[]`, depending on cart ownership flow

### Account Page

Components/pages to wire:
- `AccountDashboardPage.tsx`
- future child account sections for orders, wishlist, and addresses

APIs:
- profile summary:
  - existing auth/me data can remain source for basic identity header
- orders:
  - `GET /api/v1/orders`
  - `GET /api/v1/orders/details?id=<orderId>`
- wishlist:
  - `GET /api/v1/me/wishlist`
  - `POST /api/v1/me/wishlist/items`
  - `DELETE /api/v1/me/wishlist/items`
- addresses:
  - `GET /api/v1/me/addresses`
  - `POST /api/v1/me/addresses`
  - `PUT /api/v1/me/addresses/:id`
  - `DELETE /api/v1/me/addresses/:id`
  - `PUT /api/v1/me/addresses/:id/default`

UI mapping:
- replace inline mock order stats and recent orders with live orders data
- add saved-addresses and wishlist panels or dedicated account subroutes
- order detail actions should navigate using order number/ID while keeping product links slug-based

### Search Bar

Components/pages to wire:
- `HeaderSearch.tsx`
- `MobileMenu.tsx`
- `ProductsPage.tsx` as search-results destination

APIs:
- suggestions:
  - `GET /api/v1/discovery/suggest?prefix=<term>&limit=<n>`
- results:
  - `GET /api/v1/discovery/search?query=<term>&limit=<n>&...filters`
- analytics:
  - `POST /api/v1/discovery/search/events`
  - `POST /api/v1/discovery/search/clicks`

UI mapping:
- show live suggestion dropdown while typing
- on submit, navigate to `/products?query=<term>` and resolve results from discovery search
- fire search tracking after results load
- fire click tracking when a suggestion or result is selected

## 4. Step-by-step implementation roadmap

### Step 1 — API client layer

Components/files to modify:
- `UI/src/core/api/http.ts`
- `UI/src/core/api/interceptors.ts`
- new user-side API modules under feature folders or `UI/src/core/api`

APIs to integrate:
- catalog
- discovery
- cart
- wishlist
- orders
- addresses

Work items:
- add typed service modules for each endpoint group
- normalize backend response shapes into frontend view models
- centralize optional-auth request patterns for discovery endpoints
- keep slug as the only product route key exposed to components

State requirements:
- React Query for server data
- Zustand retained for local guest cart and auth persistence

### Step 2 — Global data hooks

Components/files to modify:
- add feature hooks for:
  - categories
  - discovery feeds
  - product detail by slug
  - cart
  - wishlist
  - orders
  - addresses
  - search suggestions/results

APIs to integrate:
- all endpoint groups above

Work items:
- create query keys and cache policies
- create auth-aware hooks that switch between guest and user flows
- add mapper helpers for product cards, cart lines, order summaries, and addresses

State requirements:
- guest/user cart abstraction layer
- session ID persistence for anonymous discovery personalization and search analytics

### Step 3 — Homepage

Components/files to modify:
- `HomePage.tsx`
- `ProductCard.tsx`
- `MainLayout.tsx`
- `Header.tsx`
- category navigation pieces

APIs to integrate:
- `GET /catalog/categories`
- `GET /discovery/feed`

UI changes:
- replace category mock grid with live categories
- replace featured mock section with multiple discovery sections
- add show-more links into products page feed views
- remove add-to-cart from homepage cards
- add wishlist and WhatsApp actions to cards

State requirements:
- concurrent feed queries
- conditional rendering for empty sections
- optional auth/session-aware discovery requests

### Step 4 — Products page

Components/files to modify:
- `ProductsPage.tsx`
- `ProductCard.tsx`

APIs to integrate:
- `GET /discovery/feed`
- `GET /catalog/products`
- filter helper endpoints

UI changes:
- derive filters from backend-supported categories/brands/attributes
- keep pagination footer
- convert cards to detail-navigation only

State requirements:
- URL-synced filters and pagination
- page reset on filter changes
- feed-type vs catalog-mode query branching

### Step 5 — Product detail

Components/files to modify:
- `ProductDetailPage.tsx`
- cart/wishlist helper hooks

APIs to integrate:
- `GET /catalog/products/slug/:slug`
- `GET /catalog/products/:id/variants`
- `GET /catalog/inventory/variants/:variant_id/stock`
- cart and wishlist mutation endpoints

UI changes:
- support variant picker with default selection
- replace static quantity/add flow with cart-aware controls
- add buy-now and WhatsApp CTA
- keep related products only if backed by live discovery/category data

State requirements:
- selected variant state
- optimistic cart/wishlist updates where safe
- guest/user cart branching

### Step 6 — Cart

Components/files to modify:
- `CartPage.tsx`
- `cartStore.ts`
- auth transition logic

APIs to integrate:
- `GET /me/cart`
- `POST /me/cart/items`
- `DELETE /me/cart/items/:id`

UI changes:
- preserve empty-state UX
- show server-backed quantities and totals for authenticated users
- add WhatsApp checkout shortcut

State requirements:
- local guest cart schema must evolve from whole-product records to product + variant snapshots
- signed-in cart cache invalidation after mutations
- guest-to-user cart merge plan

### Step 7 — Checkout

Components/files to modify:
- `CheckoutPage.tsx`
- address form/select components

APIs to integrate:
- `GET /me/addresses`
- `POST /me/addresses`
- `PUT /me/addresses/:id`
- `POST /orders`
- `POST /orders/guest`

UI changes:
- split guest shipping form and logged-in address selection
- keep pay-on-delivery enabled
- show disabled M-Pesa option
- handle success state from actual order responses

State requirements:
- order submission mutation state
- address selection and default-address fallback
- checkout source resolution from guest cart vs user cart

### Step 8 — Account pages

Components/files to modify:
- `AccountDashboardPage.tsx`
- new or expanded account subviews for:
  - orders
  - wishlist
  - addresses

APIs to integrate:
- `GET /orders`
- `GET /orders/details`
- `GET /me/wishlist`
- `POST /me/wishlist/items`
- `DELETE /me/wishlist/items`
- address endpoints

UI changes:
- replace mock orders with real order data
- surface wishlist management
- surface saved addresses and default-address actions

State requirements:
- paginated order queries
- address mutation invalidation
- wishlist cache shared with product cards

### Step 9 — Search system

Components/files to modify:
- `HeaderSearch.tsx`
- `MobileMenu.tsx`
- `ProductsPage.tsx`

APIs to integrate:
- `GET /discovery/suggest`
- `GET /discovery/search`
- `POST /discovery/search/events`
- `POST /discovery/search/clicks`

UI changes:
- autocomplete dropdown
- keyboard navigation for suggestions
- search result routing into products page

State requirements:
- debounced suggestion queries
- persisted anonymous `session_id`
- analytics event lifecycle for result impressions and clicks

## 5. Recommended implementation order

1. Build typed API modules and React Query hooks
2. Convert shared product/category/search data sources
3. Integrate homepage categories and discovery sections
4. Integrate products listing and filters
5. Integrate product detail with variants, wishlist, and cart branching
6. Rework cart model around variants and authenticated sync
7. Integrate checkout guest/user flows and addresses
8. Replace account mocks with orders, wishlist, and address data
9. Complete live search suggestions, results, and tracking

## 6. Known blockers / contract mismatches

- Product detail requires frontend access to variants alongside slug detail. The current backend router exposes both `GET /catalog/products/slug/:slug` and `GET /catalog/products/:id/variants`, and the safest implementation plan should assume the frontend will chain the variant list request after slug lookup unless the real slug-detail payload is confirmed to already contain the exact variant data needed by the UI.
- Homepage and products cards need wishlist state, but wishlist writes require both `product_id` and `variant_id`; the recommended implementation path is to resolve a deterministic default variant from product detail or variant-list data before enabling wishlist writes, while treating a product-level wishlist endpoint as a future backend enhancement rather than a frontend dependency.
- The backend contains review list logic but does not currently register a public GET reviews route in the router, so rendering review lists on product detail is blocked until that route is exposed.
- Logged-in cart deletion is item-ID based, while the current frontend cart store keys by product ID; the frontend cart model must change before API integration.
