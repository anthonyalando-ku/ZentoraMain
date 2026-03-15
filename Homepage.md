You are working inside the **Zentora workspace**, which contains two submodules:

* `Backend` → Go API (already implemented)
* `UI` → React frontend (needs API integration)

The backend is **already functional** and exposes all required APIs.

Your task is to **implement the frontend integration incrementally**, starting with the **Homepage**.

You will follow the **Frontend ↔ Backend Integration Plan** provided in the attached architecture document.

Important:

* Do NOT redesign the architecture
* Do NOT modify backend code
* Only modify files inside the `UI` project
* Implement features in small safe steps
* Do NOT remove existing layout or styling
* Replace mock data with backend data progressively

---

# Stage 1 Goal — Homepage Integration

Your goal is to fully integrate the **Homepage** so it loads real data from the backend.

This includes:

1. Category navigation
2. Discovery product feeds
3. Product cards
4. Removing mock data
5. Preserving existing layout

---

# Backend APIs to Use

## Categories

GET `/api/v1/catalog/categories`

Used for:

* header navigation
* shop-by-category section on homepage

---

## Discovery Feeds

GET `/api/v1/discovery/feed`

Parameters:

* `feed_type`
* `limit`

Example:

`/api/v1/discovery/feed?feed_type=trending&limit=8`

Supported feed types:

* trending
* best_sellers
* recommended
* deals
* new_arrivals
* highly_rated
* most_wishlisted
* also_viewed
* featured
* editorial

---

# Frontend Files Relevant to Homepage

You will primarily modify:

UI/src/features/public/home/pages/HomePage.tsx
UI/src/features/products/components/ProductCard.tsx
UI/src/shared/layouts/components/Header.tsx
UI/src/shared/layouts/components/HeaderSearch.tsx
UI/src/shared/layouts/components/MobileMenu.tsx
UI/src/shared/layouts/MainLayout.tsx

---

# Existing Frontend Infrastructure

Already available:

Axios client:
`UI/src/core/api/http.ts`

Response interceptor that unwraps:

```
{ success, message, data }
```

Auth store (Zustand):

`UI/src/features/auth/store/authStore.ts`

React Query is installed but not yet used.

---

# Implementation Requirements

## 1. Create API service modules

Create new service modules for:

catalog API
discovery API

Example location:

`UI/src/core/api/services/catalog.ts`
`UI/src/core/api/services/discovery.ts`

These services should wrap backend calls.

Example functions:

* getCategories()
* getDiscoveryFeed(feedType, limit)

---

## 2. Create React Query hooks

Create hooks for homepage data:

Example location:

`UI/src/features/catalog/hooks/`

Hooks to implement:

* useCategories()
* useDiscoveryFeed(feedType)

React Query must be used for data fetching and caching.

---

## 3. Replace Homepage mock data

Current mock sources:

`UI/src/shared/constants/mockProducts.ts`

These must be removed from homepage.

Instead:

Homepage should fetch feeds such as:

* trending
* best_sellers
* new_arrivals
* deals

Each feed should request:

limit = 8

---

## 4. Homepage Feed Rendering Rules

Each feed section should:

• render a section title
• render product cards in a horizontal grid
• hide the section if the API returns no items

Each section header must include a **Show More** link.

Example link:

```
/products?feed_type=new_arrivals
```

---

## 5. ProductCard Behavior Changes

On homepage, ProductCard must:

Show:

* product image
* name
* price
* discount
* rating
* wishlist button
* WhatsApp CTA

Must NOT show:

* add-to-cart button

Product cards should navigate to:

```
/products/:slug
```

Never use product ID in routes.

---

## 6. Discovery Feed Personalization

Discovery requests may optionally include:

```
session_id
```

If no session system exists yet:

Create a simple persistent session ID stored in:

```
localStorage
```

Example key:

```
zentora_session_id
```

Attach this to discovery feed requests.

---

## 7. UI Constraints

Do NOT change:

* page layout
* styling
* CSS
* component hierarchy

Focus only on:

data wiring and API integration.

---

# Deliverables

After completing Stage 1:

The homepage should:

• load categories from backend
• load discovery feeds from backend
• render product cards using real data
• hide empty feed sections
• link to `/products?feed_type=<type>`
• no longer use mockProducts

---

# Output Expectations

When finished:

1. Show all files created or modified.
2. Explain how the homepage now loads data.
3. Confirm that no mock data remains on the homepage.

---

I will now attach:

1. The **UI repository**
2. The **Integration Architecture Document**

Analyze them before making changes.
