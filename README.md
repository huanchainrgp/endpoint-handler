## API Test Documentation - Ludo Game Backend

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

### Endpoint Overview

| Module | Method | Path | Auth | Description |
|---|---|---|---|---|
| IAM | POST | `/iam/register` | No | Create a new user |
| IAM | POST | `/iam/login` | No | Obtain access and refresh tokens |
| IAM | POST | `/iam/refresh` | No | Exchange refresh for new access token |
| IAM | GET | `/iam/users/:id` | Bearer | Get user by ID |
| IAM | POST | `/iam/logout` | Bearer | Revoke all refresh tokens |
| Wallet | POST | `/wallet/create-user-balance` | Bearer | Ensure user balance exists |
| Wallet | POST | `/wallet/credit` | Bearer | Enqueue credit job |
| Wallet | POST | `/wallet/debit` | Bearer | Enqueue debit job |
| Wallet | GET | `/wallet/balance` | Bearer | Get current balance |
| Wallet | GET | `/wallet/transactions` | Bearer | List transactions with filters |

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

Request schema (CreateUserDTO):
```json
{
  "username": "string",
  "email": "string",
  "password": "string",
  "phone_number": "string",
  "name": "string"
}
```

Success sample (wrapped):
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "username": "tester1",
    "email": "tester1@example.com",
    "phone_number": "+1234567890",
    "name": "Tester One",
    "is_active": true,
    "created_at": "2025-01-01T00:00:00Z",
    "updated_at": "2025-01-01T00:00:00Z"
  },
  "error": null,
  "timestamp": "2025-01-01T00:00:00Z"
}
```

Common errors:
- 400 invalid payload (missing username/email/password)
```bash
curl -s -X POST "http://localhost:8080/iam/register" -H "Content-Type: application/json" -d '{"username":"","email":"bad","password":""}' | jq .
```
Expected 400 with validation details in `error.details`.

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

Request schema:
```json
{
  "username": "string",
  "password": "string",
  "remember_me": true
}
```

Success sample (wrapped):
```json
{
  "success": true,
  "data": {
    "access_token": "<jwt>",
    "refresh_token": "<jwt>",
    "access_expires_at": "2025-01-01T00:59:59Z",
    "refresh_expires_at": "2025-02-01T00:00:00Z"
  },
  "error": null,
  "timestamp": "2025-01-01T00:00:00Z"
}
```

Common errors:
- 401 invalid username or password
```bash
curl -s -X POST "http://localhost:8080/iam/login" -H "Content-Type: application/json" -d '{"username":"tester1","password":"wrong"}' | jq .
```

Export tokens for reuse:
```bash
ACCESS_TOKEN="<paste-access-token>"
REFRESH_TOKEN="<paste-refresh-token>"
USER_ID="<paste-user-id-if-needed>"
```

Or extract automatically via jq:
```bash
ACCESS_TOKEN=$(curl -s -X POST "http://localhost:8080/iam/login" -H "Content-Type: application/json" -d '{"username":"tester1","password":"StrongP@ssw0rd","remember_me":true}' | jq -r '.data.access_token')
REFRESH_TOKEN=$(curl -s -X POST "http://localhost:8080/iam/login" -H "Content-Type: application/json" -d '{"username":"tester1","password":"StrongP@ssw0rd","remember_me":true}' | jq -r '.data.refresh_token')
echo "AT len: ${#ACCESS_TOKEN}, RT len: ${#REFRESH_TOKEN}"
```

### 3) Refresh Access Token
```bash
curl -X POST "http://localhost:8080/iam/refresh" \
  -H "Content-Type: application/json" \
  -d "{ \"refresh_token\": \"$REFRESH_TOKEN\" }"
```

Success sample (wrapped):
```json
{
  "success": true,
  "data": {
    "access_token": "<jwt>",
    "access_expires_at": "2025-01-01T01:59:59Z"
  },
  "error": null,
  "timestamp": "2025-01-01T00:00:00Z"
}
```

Common errors:
- 400 malformed refresh token payload
- 401 invalid/expired refresh token

### 4) Get User By ID (protected)
```bash
curl -X GET "http://localhost:8080/iam/users/$USER_ID" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Common errors:
- 401 missing/invalid bearer token
- 404 user not found

### 5) Logout (protected)
```bash
curl -X POST "http://localhost:8080/iam/logout" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Success sample:
```json
{
  "success": true,
  "data": {"message": "logged out"},
  "error": null,
  "timestamp": "2025-01-01T00:00:00Z"
}
```

Common errors: 401 when token missing/invalid.

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

Response schema (UserBalanceDTO):
```json
{
  "id": "uuid",
  "user_id": "uuid",
  "balance": 0,
  "currency": "VND",
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-01-01T00:00:00Z"
}
```

Common errors:
- 401 when token missing/invalid
- 404 when user not found

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

Request schema (CreditBalanceDTO):
```json
{
  "amount": 100000,
  "reference_id": "uuid|null",
  "metadata": {"k":"v"}
}
```

Common errors:
- 400 invalid payload (amount < 1)
- 400 invalid `reference_id` (must be UUID)
- 404 user not found

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

Common errors:
- 400 invalid payload (amount < 1)
- 400 invalid `reference_id` (must be UUID)
- 404 user not found

### 4) Get Balance
```bash
curl -X GET "http://localhost:8080/wallet/balance" \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

Expected: 200 OK, returns balance DTO in `data` with updated `balance` after jobs are processed.

Common errors:
- 401 when token missing/invalid
- 404 when user not found

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

Response schema (GetTransactionsResponseDTO):
```json
{
  "transactions": [
    {
      "id": "uuid",
      "user_id": "uuid",
      "transaction_type": "credit|debit",
      "amount": 1000,
      "balance_before": 0,
      "balance_after": 1000,
      "reference_id": "uuid|null",
      "status": "pending|success|failed",
      "metadata": {},
      "created_at": "2025-01-01T00:00:00Z",
      "updated_at": "2025-01-01T00:00:00Z"
    }
  ],
  "total": 1,
  "limit": 20,
  "offset": 0
}
```

Common errors:
- 400 invalid query params (e.g., non-integer limit/offset)
- 401 when token missing/invalid
- 404 user not found

---

### Notes
- Ensure the background queue workers are running for credit/debit operations to take effect.
- `reference_id` must be a valid UUID when provided.
- All protected endpoints require `Authorization: Bearer <access_token>`.

#### Useful helpers
- Start API server:
```bash
make dev
```
- Start worker and monitor:
```bash
make worker
make monitor
```
- Pretty print responses:
```bash
alias curlj='curl -sS | jq .'
```

---

## End-to-End Smoke Test Script

Run the following script to exercise the full flow. Requires `jq`.

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_URL=${BASE_URL:-http://localhost:8080}
USERNAME=${USERNAME:-tester1}
EMAIL=${EMAIL:-tester1@example.com}
PASSWORD=${PASSWORD:-StrongP@ssw0rd}

echo "1) Register (idempotent if username/email unique constraints exist)"
curl -sS -X POST "$BASE_URL/iam/register" \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"$USERNAME\",\"email\":\"$EMAIL\",\"password\":\"$PASSWORD\"}" | jq . || true

echo "2) Login"
LOGIN_JSON=$(curl -sS -X POST "$BASE_URL/iam/login" \
  -H 'Content-Type: application/json' \
  -d "{\"username\":\"$USERNAME\",\"password\":\"$PASSWORD\",\"remember_me\":true}")
echo "$LOGIN_JSON" | jq . > /dev/null
ACCESS_TOKEN=$(echo "$LOGIN_JSON" | jq -r '.data.access_token')
REFRESH_TOKEN=$(echo "$LOGIN_JSON" | jq -r '.data.refresh_token')
test -n "$ACCESS_TOKEN" && test -n "$REFRESH_TOKEN"

echo "3) Refresh access token"
curl -sS -X POST "$BASE_URL/iam/refresh" \
  -H 'Content-Type: application/json' \
  -d "{\"refresh_token\":\"$REFRESH_TOKEN\"}" | jq .

echo "4) Whoami via Get User by ID (extract from login if present)"
# Optionally fetch user id by trying wallet create first and reading response later. If you know the user id, export USER_ID beforehand
if [ -z "${USER_ID:-}" ]; then
  echo "USER_ID not provided; proceeding without get-by-id check."
else
  curl -sS -X GET "$BASE_URL/iam/users/$USER_ID" -H "Authorization: Bearer $ACCESS_TOKEN" | jq .
fi

echo "5) Create user balance"
curl -sS -X POST "$BASE_URL/wallet/create-user-balance" \
  -H "Authorization: Bearer $ACCESS_TOKEN" | jq .

echo "6) Credit 100000"
curl -sS -X POST "$BASE_URL/wallet/credit" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"amount":100000,"metadata":{"source":"smoke"}}' | jq .

echo "7) Debit 50000"
curl -sS -X POST "$BASE_URL/wallet/debit" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H 'Content-Type: application/json' \
  -d '{"amount":50000,"metadata":{"reason":"smoke"}}' | jq .

echo "8) Wait for queue processing (10s)"
sleep 10

echo "9) Get balance"
curl -sS -X GET "$BASE_URL/wallet/balance" -H "Authorization: Bearer $ACCESS_TOKEN" | jq .

echo "10) Get transactions (latest)"
curl -sS -G "$BASE_URL/wallet/transactions" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  --data-urlencode "limit=10" --data-urlencode "offset=0" | jq .

echo "11) Logout"
curl -sS -X POST "$BASE_URL/iam/logout" -H "Authorization: Bearer $ACCESS_TOKEN" | jq .

echo "Done."
```

---

## Negative Test Cases

### IAM
- Register: missing required fields
```bash
curl -sS -X POST "$BASE_URL/iam/register" -H 'Content-Type: application/json' -d '{"username":"","email":"bad","password":""}' | jq .
```
- Login: wrong password
```bash
curl -sS -X POST "$BASE_URL/iam/login" -H 'Content-Type: application/json' -d '{"username":"'$USERNAME'","password":"wrong","remember_me":false}' | jq .
```
- Refresh: invalid token
```bash
curl -sS -X POST "$BASE_URL/iam/refresh" -H 'Content-Type: application/json' -d '{"refresh_token":"invalid"}' | jq .
```
- Get user: unauthorized
```bash
curl -sS -X GET "$BASE_URL/iam/users/00000000-0000-0000-0000-000000000000" | jq .
```

### Wallet
- Create user balance: unauthorized
```bash
curl -sS -X POST "$BASE_URL/wallet/create-user-balance" | jq .
```
- Credit: invalid amount
```bash
curl -sS -X POST "$BASE_URL/wallet/credit" -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' -d '{"amount":0}' | jq .
```
- Credit: invalid reference_id
```bash
curl -sS -X POST "$BASE_URL/wallet/credit" -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' -d '{"amount":100,"reference_id":"not-a-uuid"}' | jq .
```
- Debit: invalid amount
```bash
curl -sS -X POST "$BASE_URL/wallet/debit" -H "Authorization: Bearer $ACCESS_TOKEN" -H 'Content-Type: application/json' -d '{"amount":0}' | jq .
```
- Balance: unauthorized
```bash
curl -sS -X GET "$BASE_URL/wallet/balance" | jq .
```
- Transactions: invalid query params
```bash
curl -sS -G "$BASE_URL/wallet/transactions" -H "Authorization: Bearer $ACCESS_TOKEN" --data-urlencode "limit=abc" | jq .
```


