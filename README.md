## API Test Guide: IAM and Wallet Modules

Reference pattern adapted from [endpoint-handler API test documentation](https://github.com/hoanghiepnb/endpoint-handler/blob/main/API_TEST_DOCUMENTATION.md).

### Base Setup
- **Base URL**: `http://localhost:8080`
- **Headers**:
  - `Content-Type: application/json`
  - Optional: `X-Request-ID: <any-string>`
- **Auth**: Bearer token from IAM login

### Response Conventions
- Success:
```json
{
  "success": true,
  "data": {},
  "error": null,
  "timestamp": "2024-01-01T00:00:00Z"
}
```
- Error:
```json
{
  "success": false,
  "error": {
    "code": "E2004",
    "message": "Error message",
    "details": {}
  },
  "timestamp": "2024-01-01T00:00:00Z"
}
```

---

## IAM Module

### 1) Register
```bash
curl -X POST "http://localhost:8080/iam/register" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "tester1",
    "email": "tester1@example.com",
    "password": "StrongP@ssw0rd"
  }'
```

Expected: 201 Created, returns user DTO in `data`.

### 2) Login (get access + refresh tokens)
```bash
curl -X POST "http://localhost:8080/iam/login" \
  -H "Content-Type: application/json" \
  -d '{
    "username": "tester1",
    "password": "StrongP@ssw0rd",
    "remember_me": true
  }'
```

Copy `access_token`, `refresh_token` from `data`.

Export tokens for reuse:
```bash
ACCESS_TOKEN="<paste-access-token>"
REFRESH_TOKEN="<paste-refresh-token>"
USER_ID="<paste-user-id-if-needed>"
```

### 3) Refresh Access Token
```bash
curl -X POST "http://localhost:8080/iam/refresh" \
  -H "Content-Type: application/json" \
  -d "{ \"refresh_token\": \"$REFRESH_TOKEN\" }"
```

### 4) Get User By ID (protected)
```bash
curl -X GET "http://localhost:8080/iam/users/$USER_ID" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

### 5) Logout (protected)
```bash
curl -X POST "http://localhost:8080/iam/logout" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

---

## Wallet Module (all protected)

Before testing, ensure you are authenticated:
```bash
test -n "$ACCESS_TOKEN" || echo "ACCESS_TOKEN is not set"
```

### 1) Create User Balance (idempotent)
```bash
curl -X POST "http://localhost:8080/wallet/create-user-balance" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Expected: 201 Created, returns current balance DTO in `data`.

### 2) Credit Balance (enqueue job)
```bash
curl -X POST "http://localhost:8080/wallet/credit" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 100000,
    "reference_id": "f4f6c6d2-7f2d-4f85-9d8e-0f7db7f4e123",
    "metadata": {"source": "manual_topup"}
  }'
```

Expected: 202 Accepted, `{ "status": "enqueued" }` in `data`.

### 3) Debit Balance (enqueue job)
```bash
curl -X POST "http://localhost:8080/wallet/debit" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "reference_id": "c2d3b4a1-1234-4e00-9a9b-0a1b2c3d4e5f",
    "metadata": {"reason": "test_withdraw"}
  }'
```

Expected: 202 Accepted, `{ "status": "enqueued" }` in `data`.

Note: Credits/debits are processed by background workers. Allow some time before checking balance/transactions.

### 4) Get Balance
```bash
curl -X GET "http://localhost:8080/wallet/balance" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Expected: 200 OK, returns balance DTO in `data` with updated `balance` after jobs are processed.

### 5) Get Transactions (filters optional)
```bash
curl -G "http://localhost:8080/wallet/transactions" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  --data-urlencode "transactionType=credit" \
  --data-urlencode "startDate=2024-01-01T00:00:00Z" \
  --data-urlencode "endDate=2025-12-31T23:59:59Z" \
  --data-urlencode "limit=20" \
  --data-urlencode "offset=0"
```

Expected: 200 OK, returns transactions list and pagination in `data`.

---

### Notes
- Ensure the background queue workers are running for credit/debit operations to take effect.
- `reference_id` must be a valid UUID when provided.
- All protected endpoints require `Authorization: Bearer <access_token>`.


