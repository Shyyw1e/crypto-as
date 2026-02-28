# Admin API MVP

## General Rules
- Base path: `/admin/v1`
- Auth header: `X-Admin-Token: <ADMIN_API_TOKEN>`
- Content-Type: `application/json`
- Time format: RFC3339 in UTC
- All responses are JSON

## Common Response Shape

### Success
```json
{
  "ok": true,
  "data": {}
}
```

### Error
```json
{
  "ok": false,
  "error": {
    "code": "invalid_transition",
    "message": "trial already used"
  }
}
```

## Access Object (Unified DTO)
```json
{
  "chat_id": 123456789,
  "user_id": 42,
  "status": "active",
  "trial_used": true,
  "trial_started_at": "2026-02-25T10:00:00Z",
  "trial_ends_at": "2026-02-27T10:00:00Z",
  "subscription_started_at": "2026-02-27T10:01:00Z",
  "subscription_ends_at": "2026-03-27T10:01:00Z",
  "blocked_reason": null,
  "updated_at": "2026-02-27T10:01:00Z"
}
```

## Endpoints

## 1. Get Access by Chat ID
- Method: `GET`
- Path: `/admin/v1/users/{chat_id}/access`

### Response 200
```json
{
  "ok": true,
  "data": {
    "access": {
      "chat_id": 123456789,
      "user_id": 42,
      "status": "trial",
      "trial_used": true,
      "trial_started_at": "2026-02-25T10:00:00Z",
      "trial_ends_at": "2026-02-27T10:00:00Z",
      "subscription_started_at": null,
      "subscription_ends_at": null,
      "blocked_reason": null,
      "updated_at": "2026-02-25T10:00:00Z"
    }
  }
}
```

## 2. Grant Trial
- Method: `POST`
- Path: `/admin/v1/users/{chat_id}/trial`

### Request Body
```json
{
  "days": 2,
  "reason": "manual trial"
}
```

### Rules
- If `trial_used=true`, return `409`.
- Sets status to `trial`.
- Sets `trial_started_at=now`, `trial_ends_at=now+days`, `trial_used=true`.
- Writes action log with `action_type=grant_trial`.

### Response 200
```json
{
  "ok": true,
  "data": {
    "access": {}
  }
}
```

## 3. Activate Subscription
- Method: `POST`
- Path: `/admin/v1/users/{chat_id}/activate`

### Request Body
```json
{
  "until": "2026-03-31T23:59:59Z",
  "reason": "manual grant"
}
```

### Rules
- Sets status to `active`.
- Sets `subscription_started_at=now`, `subscription_ends_at=until`.
- Clears `blocked_reason`.
- Writes action log with `action_type=activate_sub`.

### Response 200
```json
{
  "ok": true,
  "data": {
    "access": {}
  }
}
```

## 4. Block User
- Method: `POST`
- Path: `/admin/v1/users/{chat_id}/block`

### Request Body
```json
{
  "reason": "fraud suspected"
}
```

### Rules
- Sets status to `blocked`.
- Sets `blocked_reason`.
- Writes action log with `action_type=block`.

### Response 200
```json
{
  "ok": true,
  "data": {
    "access": {}
  }
}
```

## 5) Unblock User
- Method: `POST`
- Path: `/admin/v1/users/{chat_id}/unblock`

### Request Body
```json
{
  "to_status": "expired",
  "until": null,
  "reason": "review passed"
}
```

or

```json
{
  "to_status": "active",
  "until": "2026-03-31T23:59:59Z",
  "reason": "review passed, grant access"
}
```

### Rules
- `to_status` allowed values: `expired`, `active`.
- If `to_status=active`, `until` is required.
- Clears `blocked_reason`.
- Writes action log with `action_type=unblock`.

### Response 200
```json
{
  "ok": true,
  "data": {
    "access": {}
  }
}
```

## 6. List User Actions
- Method: `GET`
- Path: `/admin/v1/users/{chat_id}/actions?limit=50`

### Query Params
- `limit` optional, default `50`, max `200`.

### Response 200
```json
{
  "ok": true,
  "data": {
    "items": [
      {
        "id": 101,
        "admin_name": "root",
        "target_user_id": 42,
        "action_type": "block",
        "old_status": "active",
        "new_status": "blocked",
        "payload_json": {
          "reason": "fraud suspected"
        },
        "created_at": "2026-02-27T11:00:00Z"
      }
    ]
  }
}
```

## Error Codes
- `401` - invalid admin token
- `404` - user not found
- `409` - invalid transition (e.g. trial already used)
- `422` - invalid request body/date
- `500` - internal server error

## Notes
- Server must not trust client clocks.
- All state updates and action log writes must be atomic (single transaction).
- Access checks in runtime should treat `blocked` as highest priority.
