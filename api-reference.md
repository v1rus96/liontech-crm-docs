# LionTech CRM — API Reference

**Version:** 1.0
**Base URL:** `https://<your-crm>.railway.app/api/v1` (production) | `http://localhost:3000/api/v1` (local)
**OpenAPI / Swagger UI:** `GET /api-docs` — interactive Swagger UI, non-production only. Import the spec into Postman or Insomnia from the same URL.

> **Note:** This document is validated against the production source code. All endpoint paths, request shapes, response schemas, and error codes are confirmed.

---

## Overview

The CRM API is a RESTful JSON API. All requests and responses use `application/json`.

### Authentication

JWT Bearer tokens (HS256 algorithm).

1. `POST /api/v1/auth/login` → receive `accessToken` + `refreshToken`
2. Add `Authorization: Bearer <accessToken>` to every subsequent request
3. Access tokens expire in **15 minutes**; rotate via `POST /api/v1/auth/refresh`

> **Rate limits:** General: 100 req / 15 min. Auth endpoints: **10 req / 15 min**.

### List Response Envelope

```json
{
  "data": [ ... ],
  "meta": { "total": 100, "page": 1, "limit": 20, "pages": 5 }
}
```

Single-resource responses return the object directly (no wrapper).

Sub-resource list endpoints (`/contacts/:id/activities`, `/contacts/:id/deals`, `/companies/:id/contacts`, `/companies/:id/deals`, `/deals/:id/activities`) return `{ "data": [...] }` with **no `meta`** object and fields in **snake_case** (raw DB columns).

### Error Shapes

```json
// Validation error (400)
{ "error": "Validation failed", "details": [{ "field": "email", "message": "..." }] }

// Auth/permission error
{ "error": "Unauthorized" }
{ "error": "Forbidden", "detail": "Required permission: contacts:read" }
```

### Pagination

| Param | Type | Default | Max |
|-------|------|---------|-----|
| `page` | integer | `1` | — |
| `limit` | integer | `20` | `200` |

---

## RBAC Permissions

Four roles seeded at migration:

| Role | Permissions |
|------|-------------|
| `admin` | `*` (full access) |
| `manager` | `contacts:*`, `companies:*`, `deals:*`, `pipeline_stages:*`, `activities:*`, `users:read` |
| `sales_rep` | `contacts:read/create/update`, `companies:read/create/update`, `deals:read/create/update`, `activities:*` |
| `viewer` | `contacts:read`, `companies:read`, `deals:read`, `activities:read` |

Permission format: `<resource>:<action>`. Wildcard `*` grants all actions.

---

## Health Check

No auth required.

```http
GET /health
```

`200 { "status": "ok", "db": "ok", "timestamp": "..." }` | `503` if DB is unreachable (`"db": "degraded"`).

---

## Auth

### POST /api/v1/auth/login

```json
// Request
{ "email": "user@example.com", "password": "secret" }

// Response 200
{
  "accessToken": "eyJ...",
  "refreshToken": "<opaque-string>",
  "expiresIn": "15m",
  "user": { "id": "uuid", "email": "user@example.com", "role": "sales_rep" }
}
```

`401` on invalid credentials or inactive account. Failure path is constant-time (anti-enumeration).

---

### POST /api/v1/auth/refresh

Rotates the refresh token — old token is revoked, new pair returned.

```json
// Request body (required)
{ "refreshToken": "<opaque-string>" }

// Response 200
{ "accessToken": "eyJ...", "refreshToken": "<new-opaque-string>" }
```

> Pass `refreshToken` in the **request body**, not in an `Authorization` header.

---

### POST /api/v1/auth/logout

Requires `Authorization: Bearer <accessToken>`.

```json
// Request body (optional)
{ "refreshToken": "<opaque-string>" }
```

`204` on success.

---

### GET /api/v1/auth/me

Returns the authenticated user's full profile.

```json
{
  "id": "uuid",
  "email": "user@example.com",
  "firstName": "Jane",
  "lastName": "Doe",
  "avatarUrl": null,
  "role": {
    "id": "uuid",
    "name": "sales_rep",
    "permissions": ["contacts:read", "contacts:create", "contacts:update"]
  },
  "lastLoginAt": "2026-03-27T18:00:00.000Z",
  "createdAt": "2026-01-01T00:00:00.000Z"
}
```

---

## Contacts

Required permission: `contacts:<action>`

**Contact object** (list and `GET /:id` responses):

| Field | Type |
|-------|------|
| `id` | UUID |
| `firstName`, `lastName` | string |
| `fullName` | string — computed |
| `email`, `phone`, `jobTitle` | string |
| `companyId` | UUID \| null |
| `companyName` | string \| null — joined from companies |
| `assignedTo` | UUID \| null |
| `tags`, `segments` | string[] |
| `source`, `status`, `notes` | string |
| `createdBy` | UUID |
| `createdAt`, `updatedAt` | datetime |

> `companyName` is **not** present in `POST` (create) or `PATCH` (update) responses — only in `GET` responses.

**Status values:** `lead` (default), `prospect`, `customer`, `churned`

### GET /api/v1/contacts

| Param | Type | Notes |
|-------|------|-------|
| `q` | string | Trigram ILIKE on full name |
| `email` | string | Exact match |
| `status` | string | Enum above |
| `companyId` | UUID | |
| `tags` | string \| string[] | Array overlap filter |
| `segments` | string \| string[] | Array overlap filter |
| `sortBy` | string | `first_name`, `last_name`, `created_at`(default), `updated_at` |
| `order` | string | `asc`, `desc`(default) |
| `page`, `limit` | integer | |

```bash
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:3000/api/v1/contacts?q=smith&status=customer&limit=10"
```

### POST /api/v1/contacts

| Field | Required | Constraints |
|-------|----------|-------------|
| `firstName` | Yes | max 128 |
| `lastName` | Yes | max 128 |
| `email` | No | valid email, max 320 |
| `phone` | No | max 32 |
| `jobTitle` | No | max 128 |
| `companyId` | No | UUID |
| `assignedTo` | No | UUID |
| `tags` | No | string[], default `[]` |
| `segments` | No | string[], default `[]` |
| `source` | No | max 64 |
| `status` | No | enum, default `lead` |
| `notes` | No | max 10000 |

`201` — Contact object (without `companyName`).

### GET /api/v1/contacts/:id — `200` or `404`
### PATCH /api/v1/contacts/:id — all fields optional, `200` (without `companyName`) or `404`
### DELETE /api/v1/contacts/:id — `contacts:delete`, `204` or `404`

### GET /api/v1/contacts/:id/activities
### GET /api/v1/contacts/:id/deals

Both return `{ "data": [...] }` with raw snake_case column names. No pagination.

---

## Companies

Required permission: `companies:<action>`

**Company object:** `id`, `name`, `domain`, `industry`, `size`, `website`, `phone`, `address` (object), `description`, `createdBy`, `createdAt`, `updatedAt`.

**`size` values:** `startup`, `smb`, `mid_market`, `enterprise`

### GET /api/v1/companies

| Param | Notes |
|-------|-------|
| `q` | ILIKE search on name |
| `sortBy` | `name`, `created_at`(default), `updated_at` |
| `order` | `asc`, `desc`(default) |
| `page`, `limit` | |

### POST /api/v1/companies

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | max 255 |
| `domain` | No | max 255 |
| `industry` | No | max 128 |
| `size` | No | enum above |
| `website` | No | valid URI |
| `phone` | No | max 32 |
| `address` | No | `{ street, city, state, country, zip }` |
| `description` | No | max 10000 |

`201` — Company object.

### GET /api/v1/companies/:id — `200` or `404`
### PATCH /api/v1/companies/:id — all fields optional, `200` or `404`
### DELETE /api/v1/companies/:id — `companies:delete`, `204` or `404`
### GET /api/v1/companies/:id/contacts — `{ data: [...] }`, raw snake_case, no pagination
### GET /api/v1/companies/:id/deals — `{ data: [...] }`, raw snake_case + `stage_name`, no pagination

---

## Deals

Required permission: `deals:<action>`

**Deal object:**

| Field | Type |
|-------|------|
| `id` | UUID |
| `title` | string |
| `stageId` | UUID |
| `stageName` | string — joined |
| `contactId`, `companyId`, `assignedTo` | UUID \| null |
| `value` | number (float) |
| `currency` | string (ISO 4217) |
| `expectedClose` | date \| null |
| `closedAt` | datetime \| null — auto-set on move to closed stage |
| `description`, `source` | string |
| `tags` | string[] |
| `createdBy` | UUID |
| `createdAt`, `updatedAt` | datetime |

### GET /api/v1/deals

| Param | Notes |
|-------|-------|
| `q` | ILIKE on title |
| `stageId`, `assignedTo`, `companyId`, `contactId` | UUID filters |
| `minValue`, `maxValue` | number |
| `sortBy` | `title`, `value`, `expected_close`, `created_at`(default), `updated_at` |
| `order` | `asc`, `desc`(default) |
| `page`, `limit` | |

### POST /api/v1/deals

| Field | Required | Notes |
|-------|----------|-------|
| `title` | Yes | max 255 |
| `stageId` | Yes | UUID |
| `contactId`, `companyId`, `assignedTo` | No | UUID |
| `value` | No | default `0`, ≥ 0 |
| `currency` | No | ISO 4217, default `USD` |
| `expectedClose` | No | ISO 8601 date |
| `description` | No | max 10000 |
| `source` | No | max 64 |
| `tags` | No | string[], default `[]` |

`201` — Deal object (includes `stageName`).

### GET /api/v1/deals/:id — `200` or `404`
### PATCH /api/v1/deals/:id

Handles stage transitions: when `stageId` changes to a closed stage (`isClosedWon` or `isClosedLost`), `closedAt` is automatically set to the current timestamp.

### DELETE /api/v1/deals/:id — `deals:delete`, `204` or `404`
### GET /api/v1/deals/:id/activities — `{ data: [...] }`, raw snake_case, no pagination

---

## Pipeline Stages

Required permission: `pipeline_stages:<action>`

**Default stages** (seeded by migration):

| Position | Name | Probability | Closed Won | Closed Lost |
|----------|------|-------------|------------|-------------|
| 1 | Prospecting | 10% | No | No |
| 2 | Qualification | 25% | No | No |
| 3 | Proposal | 50% | No | No |
| 4 | Negotiation | 75% | No | No |
| 5 | Closed Won | 100% | **Yes** | No |
| 6 | Closed Lost | 0% | No | **Yes** |

### GET /api/v1/pipeline-stages

Returns stages ordered by `position`. No pagination.

| Param | Notes |
|-------|-------|
| `pipeline` | Filter by pipeline name. Default: `default`. |

### POST /api/v1/pipeline-stages

| Field | Required | Notes |
|-------|----------|-------|
| `name` | Yes | max 128 |
| `pipeline` | No | default `default` |
| `position` | Yes | integer ≥ 1, unique per pipeline |
| `probability` | No | 0–100, default `0` |
| `isClosedWon` | No | boolean, default `false` |
| `isClosedLost` | No | boolean, default `false` |

### POST /api/v1/pipeline-stages/reorder

Bulk-reorder stages. Required: `pipeline_stages:update`.

```json
// Request body
{
  "pipeline": "default",
  "order": ["<stage-uuid-1>", "<stage-uuid-2>", "..."]
}
```

`order` must be a non-empty array containing all stage UUIDs in the desired order. Returns `{ "data": [...] }` (ordered stages).

Uses a two-pass transaction to avoid unique position constraint violations during reorder.

### GET /api/v1/pipeline-stages/:id — `200` or `404`
### PATCH /api/v1/pipeline-stages/:id — all fields optional, `200` or `404`
### DELETE /api/v1/pipeline-stages/:id

`204` on success. `409 Conflict` if any deals reference this stage — move or reassign those deals first.

---

## Activities

Required permission: `activities:<action>`

**Constraint:** Must specify at least one of `contactId`, `dealId`, or `companyId`.

**Activity object:** `id`, `type`, `subject`, `body`, `outcome`, `contactId`, `dealId`, `companyId`, `assignedTo`, `scheduledAt`, `completedAt`, `durationMin`, `createdBy`, `createdAt`, `updatedAt`.

### GET /api/v1/activities

Results ordered by `createdAt DESC` (no sort params).

| Param | Notes |
|-------|-------|
| `type` | `call`, `email`, `meeting`, `note`, `task` |
| `contactId`, `dealId`, `companyId`, `assignedTo` | UUID filters |
| `page`, `limit` | |

### POST /api/v1/activities

| Field | Required | Notes |
|-------|----------|-------|
| `type` | Yes | `call`, `email`, `meeting`, `note`, `task` |
| `subject` | Yes | max 255 |
| `body` | No | max 50000 |
| `outcome` | No | max 128 |
| `contactId` | No* | At least one required |
| `dealId` | No* | At least one required |
| `companyId` | No* | At least one required |
| `assignedTo` | No | UUID |
| `scheduledAt`, `completedAt` | No | ISO 8601 datetime |
| `durationMin` | No | integer ≥ 0 |

`201` — Activity object.

### GET /api/v1/activities/:id — `200` or `404`
### PATCH /api/v1/activities/:id — all fields optional, `200` or `404`
### DELETE /api/v1/activities/:id — `activities:delete`, `204` or `404`

---

## Users

Required permission: varies (see each endpoint).

**User object:** `id`, `email`, `firstName`, `lastName`, `avatarUrl`, `isActive`, `roleId`, `roleName`, `lastLoginAt`, `createdAt`, `updatedAt`.

### GET /api/v1/users — Requires: `users:read`

| Param | Notes |
|-------|-------|
| `q` | ILIKE on `first_name + last_name + email` |
| `roleId` | UUID filter |
| `page`, `limit` | |

### POST /api/v1/users — Admin only (role check beyond permission)

| Field | Required |
|-------|----------|
| `email` | Yes |
| `password` | Yes (8–128 chars) |
| `firstName`, `lastName` | Yes |
| `roleId` | Yes |
| `avatarUrl` | No |

`201` — User object.

### GET /api/v1/users/roles — Requires: `users:read`

Returns `{ "data": [{ "id", "name", "permissions" }] }`.

### GET /api/v1/users/:id — Requires: `users:read`, `200` or `404`

### PATCH /api/v1/users/:id

No specific permission required beyond authentication. Ownership enforced in controller: managers and admins can update any user's basic profile; non-privileged users can only update their own.

| Field | Who can set |
|-------|-------------|
| `firstName`, `lastName`, `avatarUrl` | Self, manager, or admin |
| `roleId`, `isActive` | Admin only |
| `currentPassword` + `newPassword` | Self only (currentPassword required) |

`400` if `newPassword` provided without `currentPassword`, or if `currentPassword` is wrong.

### DELETE /api/v1/users/:id — Admin only

Soft-delete: sets `isActive = false`. User record is retained; the account cannot log in. Returns `400` if you attempt to delete your own account.

---

## Error Reference

| Status | Meaning |
|--------|---------|
| 400 | Validation failed, or no fields to update, or business rule violation |
| 401 | Missing, invalid, or expired token |
| 403 | Insufficient permissions |
| 404 | Resource not found |
| 409 | Conflict — e.g. stage has deals, duplicate unique value |
| 429 | Rate limit exceeded |
| 500 | Internal server error |
| 503 | DB unreachable |

---

## End-to-End Example

```bash
# Login
TOKENS=$(curl -s -X POST http://localhost:3000/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"secret123"}')
TOKEN=$(echo $TOKENS | jq -r .accessToken)
REFRESH=$(echo $TOKENS | jq -r .refreshToken)

# Create a contact
curl -s -X POST http://localhost:3000/api/v1/contacts \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"firstName":"Jane","lastName":"Smith","email":"jane@acme.com","status":"lead","tags":["enterprise"]}'

# Create a deal linked to a stage
STAGE_ID=$(curl -s -H "Authorization: Bearer $TOKEN" \
  http://localhost:3000/api/v1/pipeline-stages | jq -r '.data[0].id')

curl -s -X POST http://localhost:3000/api/v1/deals \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"Acme Enterprise\",\"stageId\":\"$STAGE_ID\",\"value\":50000}"

# Refresh token before expiry
NEW_TOKENS=$(curl -s -X POST http://localhost:3000/api/v1/auth/refresh \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$REFRESH\"}")
TOKEN=$(echo $NEW_TOKENS | jq -r .accessToken)

# Logout
curl -s -X POST http://localhost:3000/api/v1/auth/logout \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d "{\"refreshToken\":\"$(echo $NEW_TOKENS | jq -r .refreshToken)\"}"
```

