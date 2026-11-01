# CanteenFlow

CanteenFlow is a small college-level canteen ordering application. Students can browse available food and submit orders; staff can maintain the menu and manage orders from a dashboard. The project uses plain HTML, CSS, and browser JavaScript with an Express API, MongoDB, and optional Redis caching.

This is a **single-user academic demonstration**. It has no authentication or authorization, so do not expose the staff dashboard or API to a public network. There are no online payments or delivery features.

## What the databases do

- **MongoDB is the permanent source of truth.** Menu items and orders are stored there, and orders keep item-name and price snapshots so history remains readable if the menu changes.
- **Redis is an optional performance and demonstration feature.** It caches menu listings, individual menu reads, and dashboard statistics for 60 seconds. It also counts category filters in a sorted set so the dashboard can show the three most-viewed categories. Redis data is not permanent application data.
- If Redis is missing or unavailable, menu and order CRUD continue against MongoDB. Cache reads fall back to MongoDB, and the category ranking reports that Redis is unavailable.

## Requirements

- Node.js 18 or newer and npm.
- MongoDB Community Server running locally, or a hosted MongoDB connection string.
- Redis server running locally, or a hosted Redis connection URL. Redis is optional.

Installing the `redis` npm package only installs the Node.js client library; it does **not** install or start a Redis server. Likewise, installing Mongoose does not install or start MongoDB. Install/start those database services separately, or configure hosted services.

## Setup and run

From the project directory:

```sh
npm install
```

Copy `.env.example` to `.env`, then set the MongoDB URI. In PowerShell:

```powershell
Copy-Item .env.example .env
```

On macOS or Linux, use `cp .env.example .env`. Then edit `.env`:

```text
MONGODB_URI=mongodb://127.0.0.1:27017/canteenflow
REDIS_URL=redis://localhost:6379
PORT=3000
```

For hosted MongoDB, replace the local URI with the connection string supplied by your provider and allow your machine's network address in that provider's network settings. For hosted Redis, set `REDIS_URL` to its supplied Redis URL. Keep credentials private and do not commit `.env`. Leave `REDIS_URL` blank to run without Redis.

Start MongoDB and, if using it, Redis. To add the optional sample menu entries, run:

```sh
npm run seed
```

The seed script adds missing examples only; it does not delete or overwrite existing menu items. The application also runs with an empty database.

Start the application:

```sh
npm start
```

Open the student menu at <http://localhost:3000/menu.html>, the staff dashboard at <http://localhost:3000/admin.html>, or the home page at <http://localhost:3000>. Set `PORT` in `.env` to use a different port.

## Features

- Search available food by name and filter by category.
- Add items to a session-persisted cart, change quantities, and remove items.
- Submit an order with a student name and optional contact information.
- Create, edit, and delete menu items; toggle their availability.
- View orders and their item snapshots, filter by status, edit student/contact/status details, and delete orders.
- Dashboard counts for menu items, orders, pending orders, and completed orders.
- Redis category-filter ranking, cache TTLs, and cache invalidation.
- Responsive layouts, accessible labels and focus styles, and safe text rendering for user/database values.

## API

All API responses are JSON. IDs must be MongoDB ObjectIds. Order status values are `Pending`, `Preparing`, `Ready`, and `Completed`.

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/menu` | List available menu items; optional `?category=Drinks` filter |
| GET | `/api/menu/all` | List all menu items for staff |
| GET | `/api/menu/:id` | Get a menu item |
| POST | `/api/menu` | Create a menu item |
| PUT | `/api/menu/:id` | Partially update a menu item |
| DELETE | `/api/menu/:id` | Delete a menu item |
| GET | `/api/orders` | List orders; optional `?status=Pending` filter |
| GET | `/api/orders/:id` | Get order details |
| POST | `/api/orders` | Create an order; server looks up current prices |
| PUT | `/api/orders/:id` | Update `studentName`, `contact`, and/or `status` |
| DELETE | `/api/orders/:id` | Delete an order |
| GET | `/api/dashboard/stats` | Get dashboard counts |
| GET | `/api/dashboard/popular-categories` | Get the Redis top-three category ranking |
| GET | `/api/health` | Check MongoDB and Redis status without returning connection details |

Example request bodies:

```json
{
  "name": "Masala Dosa",
  "description": "Dosa with spiced potato.",
  "category": "South Indian",
  "price": 65,
  "available": true
}
```

```json
{
  "studentName": "Asha",
  "contact": "",
  "items": [{ "itemId": "0123456789abcdef01234567", "quantity": 2 }]
}
```

The order example's ID is illustrative; use an actual ID returned by `/api/menu`. Submitted prices and totals are not accepted. The server rejects empty orders, invalid quantities/IDs, duplicate item IDs, missing or unavailable menu items, and invalid fields.

## Checking CRUD and caching

1. Open the student menu and add a menu item from the staff dashboard. Confirm it appears on the student menu. Edit it, mark it unavailable, and confirm it is no longer listed; delete it when finished.
2. Place an order from the student page. Its confirmation shows the server-returned order ID and total. On the dashboard, inspect its item snapshots, filter by status, update its status or student details, then delete it if desired.
3. Use the dashboard Add/Edit/Delete controls to exercise menu CRUD. The corresponding API routes can also be called with a REST client; `GET /api/menu/all` lists all records.
4. With Redis running, request a menu listing or dashboard stats twice and observe cache keys and their approximately 60-second expiration in your Redis client. Create/edit/delete a menu item and request the listing again to see invalidation. Create/update/delete an order and request dashboard stats again to see its invalidation.
5. Select category filters on the student menu, then refresh Popular categories on the staff dashboard. The sorted set ranks category-filter use, not item popularity or sales.
6. Stop the Redis service and use the menu and staff CRUD again. MongoDB operations should still work; `/api/health` reports Redis as unavailable and the category ranking explains why it cannot be shown.
7. Restart the Node server and query `/api/orders` and `/api/menu/all` to verify that records persist in MongoDB.

## Common setup issues

- **MongoDB connection failed:** confirm the MongoDB service is running, the URI is correct, the database host is reachable, and hosted-service network access is configured. MongoDB is required for server startup.
- **Redis unavailable warning:** confirm the Redis service and `REDIS_URL`; otherwise leave Redis disabled and continue with MongoDB.
- **Port already in use:** set `PORT` to an unused port, then open the matching localhost URL.
- **API returns 400:** check the documented JSON fields, allowed order statuses, positive prices, integer quantities, and actual ObjectIds.

## Viva summary

MongoDB stores the actual menu and orders permanently. Redis temporarily speeds up repeated menu/dashboard reads and uses a sorted set for category-filter counts. Redis can be cleared or unavailable without losing menu or order data.
