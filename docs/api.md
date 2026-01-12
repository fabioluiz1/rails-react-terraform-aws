# API Documentation

Base URL: `/api/v1`

## Authentication

**Demo Mode**: This application uses demo authentication for testing purposes.

- Send `X-User-Id` header with user ID to specify which user to act as
- If no header is provided, the API falls back to `User.first`
- In production, this would be replaced with proper authentication (JWT, OAuth,
  etc.)

Example:

```bash
curl -H "X-User-Id: 1" http://localhost:3001/api/v1/users/me
```

## Endpoints

### GET /users/me

Get the current user's profile and points balance.

**Headers**:

- `X-User-Id` (optional): User ID to act as

**Success Response** (200 OK):

```json
{
  "id": 1,
  "email": "demo@example.com",
  "points_balance": 500
}
```

**Error Responses**:

- `404 Not Found`: User not found

  ```json
  { "error": "User not found" }
  ```

---

### GET /rewards

Get a paginated list of available rewards (excludes unavailable rewards).

**Query Parameters**:

- `limit` (optional): Number of results per page (default: 20, max: 100, min: 1)
- `offset` (optional): Number of results to skip (default: 0, min: 0)

**Success Response** (200 OK):

```json
[
  {
    "id": 1,
    "name": "Free Coffee",
    "description": "Enjoy a complimentary coffee of your choice, any size.",
    "points_cost": 100,
    "image_url": "https://images.unsplash.com/...",
    "available": true
  }
]
```

**Notes**:

- Results are ordered by `points_cost` (ascending)
- Only rewards with `available: true` are returned
- Pagination: Use `limit` and `offset` for paginated access

---

### GET /redemptions

Get the current user's redemption history.

**Headers**:

- `X-User-Id` (optional): User ID to act as

**Success Response** (200 OK):

```json
[
  {
    "id": 1,
    "points_spent": 100,
    "redeemed_at": "2026-01-12T10:30:00Z",
    "reward": {
      "id": 1,
      "name": "Free Coffee",
      "description": "Enjoy a complimentary coffee...",
      "points_cost": 100,
      "image_url": "https://...",
      "available": true
    }
  }
]
```

**Notes**:

- Results are ordered by `redeemed_at` (descending - most recent first)
- Includes nested reward details for each redemption

---

### POST /redemptions

Redeem a reward for the current user.

**Headers**:

- `X-User-Id` (optional): User ID to act as
- `Content-Type: application/json`

**Request Body**:

```json
{
  "reward_id": 1
}
```

**Success Response** (201 Created):

```json
{
  "id": 1,
  "points_spent": 100,
  "redeemed_at": "2026-01-12T10:30:00Z",
  "reward": {
    "id": 1,
    "name": "Free Coffee",
    "description": "Enjoy a complimentary coffee...",
    "points_cost": 100,
    "image_url": "https://...",
    "available": true
  }
}
```

**Error Responses**:

- `400 Bad Request`: Missing reward_id

  ```json
  { "error": "reward_id is required" }
  ```

- `404 Not Found`: Reward doesn't exist

  ```json
  { "error": "Reward not found" }
  ```

- `422 Unprocessable Entity`: Insufficient points

  ```json
  { "error": "Insufficient points" }
  ```

- `422 Unprocessable Entity`: Reward unavailable

  ```json
  { "error": "Reward unavailable" }
  ```

**Transaction Guarantees**:

- Uses pessimistic locking to prevent race conditions
- Atomically deducts points and creates redemption record
- Rolls back on failure

---

### GET /up

Rails health check endpoint for load balancers and monitoring.

**Success Response** (200 OK):

```text
Rails app is running
```

**Error Response** (500 Internal Server Error):

```text
Rails app failed to boot
```

## Data Models

### User

| Field | Type | Validations |
|-------|------|-------------|
| id | Integer | Primary key |
| email | String | Required, unique, valid email format |
| points_balance | Integer | Required, >= 0 |
| created_at | DateTime | Auto-generated |
| updated_at | DateTime | Auto-generated |

**Associations**:

- `has_many :redemptions`

### Reward

| Field | Type | Validations |
|-------|------|-------------|
| id | Integer | Primary key |
| name | String | Required |
| description | Text | Optional |
| points_cost | Integer | Required, > 0 |
| image_url | String | Optional |
| available | Boolean | Required, defaults to true |
| created_at | DateTime | Auto-generated |
| updated_at | DateTime | Auto-generated |

**Associations**:

- `has_many :redemptions`

**Scopes**:

- `available`: Returns only rewards where `available = true`

### Redemption

| Field | Type | Validations |
|-------|------|-------------|
| id | Integer | Primary key |
| user_id | Integer | Foreign key, required |
| reward_id | Integer | Foreign key, required |
| points_spent | Integer | Required, > 0 |
| redeemed_at | DateTime | Required |
| created_at | DateTime | Auto-generated |
| updated_at | DateTime | Auto-generated |

**Associations**:

- `belongs_to :user`
- `belongs_to :reward`

## Business Logic

### RedemptionService

Handles the complex business logic for redeeming rewards with transaction safety.

**Method**: `RedemptionService.redeem(user_id:, reward_id:)`

**Process**:

1. Start database transaction
2. Lock user record (pessimistic locking)
3. Find reward
4. Validate reward availability
5. Validate user has sufficient points
6. Deduct points from user
7. Create redemption record
8. Commit transaction (or rollback on error)

**Custom Exceptions**:

- `RedemptionService::InsufficientPointsError`
- `RedemptionService::RewardUnavailableError`

## Serializers

The API uses Alba for JSON serialization.

### UserResource

- Attributes: `id`, `email`, `points_balance`

### RewardResource

- Attributes: `id`, `name`, `description`, `points_cost`, `image_url`, `available`

### RedemptionResource

- Attributes: `id`, `points_spent`, `redeemed_at`
- Nested: `reward` (full RewardResource)

## Error Handling

All errors are returned as JSON with appropriate HTTP status codes:

```json
{
  "error": "Human-readable error message"
}
```

Sentry integration captures all unhandled exceptions with request context.
