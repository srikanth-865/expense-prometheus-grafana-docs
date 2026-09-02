# Expense Backend v1 — API Documentation

## Overview

**Base URL:** `http://<host>:8080` (or same-origin `/api/...` behind the frontend's nginx proxy)
**Default Port:** `8080` (override with `PORT` environment variable)
**Content-Type:** `application/json`

The Expense Backend v1 is a Node.js/Express REST API managing categories and
expenses in MySQL, with Prometheus metrics (`/metrics`) and structured `pino`
logs correlated by request ID.

---

## Database Schema

```sql
CREATE TABLE categories (
  id         INT AUTO_INCREMENT PRIMARY KEY,
  name       VARCHAR(100) NOT NULL UNIQUE,
  icon       VARCHAR(50)  NOT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE expenses (
  id            INT AUTO_INCREMENT PRIMARY KEY,
  category_id   INT NOT NULL REFERENCES categories(id) ON DELETE RESTRICT,
  amount        DECIMAL(10,2) NOT NULL CHECK (amount > 0),
  expense_date  DATE NOT NULL,
  notes         VARCHAR(500) NULL,
  created_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

### Category Object

| Field        | Type    | Description                       |
|--------------|---------|------------------------------------|
| id           | integer | Auto-generated unique identifier   |
| name         | string  | Unique category name (max 100 chars) |
| icon         | string  | lucide-react icon name (max 50 chars) |
| created_at   | string  | ISO timestamp                      |

Categories are user-defined — there is no fixed enum, unlike earlier
versions of this app.

### Expense Object

| Field         | Type    | Description                              |
|---------------|---------|--------------------------------------------|
| id            | integer | Auto-generated unique identifier           |
| category_id   | integer | FK to `categories.id`                      |
| amount        | decimal | Must be > 0                                |
| expense_date  | string  | `YYYY-MM-DD`                               |
| notes         | string  | Optional, max 500 chars                    |
| category_name | string  | Joined from `categories` (read-only)       |
| category_icon | string  | Joined from `categories` (read-only)       |
| created_at    | string  | ISO timestamp                              |
| updated_at    | string  | ISO timestamp                              |

---

## Environment Variables

| Variable              | Default         | Description                          |
|------------------------|-----------------|----------------------------------------|
| `PORT`                 | `8080`          | Port the server listens on             |
| `SERVICE_NAME`         | `expense-backend` | Used in logs/metrics labels          |
| `DB_HOST`              | `127.0.0.1`     | MySQL host address                     |
| `DB_PORT`              | `3306`          | MySQL port                             |
| `DB_USER`              | `expense_app`   | MySQL username                         |
| `DB_PASSWORD`          | *(empty)*       | MySQL password                         |
| `DB_NAME`              | `expense_db`    | MySQL database name                    |
| `LOG_LEVEL`            | `info`          | pino log level                         |
| `ENABLE_DEBUG_ROUTES`  | `false`         | Enables `/debug/*` fault-injection routes |

---

## Endpoints

### Health Check

#### `GET /health`

**Response `200 OK`**
```json
{ "status": "ok" }
```

**Response `503 Service Unavailable`** — MySQL pool unreachable
```json
{ "status": "degraded" }
```

---

### Categories

#### `GET /api/categories`

Returns all categories, alphabetical by name.

**Response `200 OK`**
```json
[
  { "id": 1, "name": "Food", "icon": "utensils", "created_at": "2026-01-01T00:00:00.000Z" }
]
```

#### `POST /api/categories`

**Request Body**
```json
{ "name": "Food", "icon": "utensils" }
```

| Field  | Type   | Required | Constraints            |
|--------|--------|----------|--------------------------|
| `name` | string | Yes      | Non-empty, unique       |
| `icon` | string | Yes      | Non-empty               |

**Response `201 Created`** — the created category object

**Response `400 Bad Request`**
```json
{ "error": "name is required" }
```

**Response `409 Conflict`** — duplicate name
```json
{ "error": "category name already exists" }
```

#### `DELETE /api/categories/:id`

**Response `204 No Content`**

**Response `404 Not Found`**
```json
{ "error": "category not found" }
```

**Response `409 Conflict`** — category still has expenses (`ON DELETE RESTRICT`)
```json
{ "error": "category has expenses and cannot be deleted" }
```

---

### Expenses

#### `GET /api/expenses`

Returns all expenses, most recent `expense_date` first, joined with category
name/icon.

**Response `200 OK`**
```json
[
  {
    "id": 3, "category_id": 1, "amount": "500.00", "expense_date": "2026-07-30",
    "notes": "Groceries", "category_name": "Food", "category_icon": "utensils",
    "created_at": "2026-07-30T10:00:00.000Z", "updated_at": "2026-07-30T10:00:00.000Z"
  }
]
```

#### `POST /api/expenses`

**Request Body**
```json
{ "category_id": 1, "amount": 500, "expense_date": "2026-07-30", "notes": "Groceries" }
```

| Field          | Type    | Required | Constraints                          |
|----------------|---------|----------|----------------------------------------|
| `category_id`  | integer | Yes      | Must reference an existing category  |
| `amount`       | number  | Yes      | > 0                                    |
| `expense_date` | string  | Yes      | `YYYY-MM-DD`, valid date               |
| `notes`        | string  | No       | Max 500 chars                          |

**Response `201 Created`** — the created expense object (joined)

**Response `400 Bad Request`**
```json
{ "error": "amount must be a number greater than 0" }
```

#### `PUT /api/expenses/:id`

Same body/validation as `POST`. Replaces all fields.

**Response `200 OK`** — the updated expense object

**Response `404 Not Found`**
```json
{ "error": "expense not found" }
```

#### `DELETE /api/expenses/:id`

**Response `204 No Content`**

**Response `404 Not Found`**
```json
{ "error": "expense not found" }
```

---

## Debug Routes (`ENABLE_DEBUG_ROUTES=true` only)

Not mounted unless explicitly enabled — useful for demoing failure modes,
not for production traffic.

| Method | Path              | Behavior                                              |
|--------|-------------------|--------------------------------------------------------|
| GET    | `/debug/error`    | Always returns `500`, logs an error                   |
| GET    | `/debug/slow`     | Sleeps `?ms=` (default 3000) before responding `200`  |
| GET    | `/debug/cardinality` | Increments a metric with a random label (cardinality demo) |

---

## Metrics

#### `GET /metrics`

Prometheus exposition format. Includes default Node process metrics plus
app-specific counters/histograms (HTTP request duration by route template,
`expenses_created_total`, `expenses_deleted_total`, `expense_amount_total`).

---

## Quick Reference

| Method | Path                    | Description                    |
|--------|-------------------------|----------------------------------|
| GET    | `/health`               | Service health check            |
| GET    | `/metrics`              | Prometheus metrics              |
| GET    | `/api/categories`       | List all categories             |
| POST   | `/api/categories`       | Create a category                |
| DELETE | `/api/categories/:id`   | Delete a category                |
| GET    | `/api/expenses`         | List all expenses (joined)      |
| POST   | `/api/expenses`         | Create an expense                |
| PUT    | `/api/expenses/:id`     | Update an expense                 |
| DELETE | `/api/expenses/:id`     | Delete an expense                 |

---

## Dependencies

| Package     | Version  | Purpose                          |
|-------------|----------|------------------------------------|
| express     | ^4.19.2  | HTTP server framework              |
| mysql2      | ^3.11.0  | MySQL database driver (pooled)     |
| prom-client | ^15.1.3  | Prometheus metrics                 |
| pino        | ^9.3.2   | Structured JSON logging            |
| uuid        | ^10.0.0  | Request ID generation              |
| dotenv      | ^16.4.5  | `.env` loading                     |
