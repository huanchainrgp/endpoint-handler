# Module Wallet - Hệ Thống Quản Lý Ví Điện Tử

## 📋 Mục Lục

1. [Tổng Quan](#tổng-quan)
2. [Kiến Trúc Module](#kiến-trúc-module)
3. [Tạo User và User Balance](#tạo-user-và-user-balance)
4. [Hoạt Động Của Hệ Thống Wallet](#hoạt-động-của-hệ-thống-wallet)
5. [Các API Endpoints](#các-api-endpoints)
6. [Xử Lý Giao Dịch (Transactions)](#xử-lý-giao-dịch-transactions)
7. [Hệ Thống Queue](#hệ-thống-queue)
8. [Hướng Dẫn Testing](#hướng-dẫn-testing) ⭐
9. [Các Vấn Đề Thường Gặp](#các-vấn-đề-thường-gặp)
10. [Best Practices](#best-practices)

---

## 🎯 Tổng Quan

Module Wallet là hệ thống quản lý ví điện tử cho phép:
- Tạo và quản lý số dư (balance) cho người dùng
- Thực hiện các giao dịch nạp tiền (credit) và rút tiền (debit)
- Lưu trữ lịch sử giao dịch với đầy đủ thông tin audit
- Xử lý bất đồng bộ qua hệ thống queue để đảm bảo hiệu suất cao
- Đảm bảo tính toàn vẹn dữ liệu thông qua transactions và idempotency

### Các Tính Năng Chính

- ✅ **Tự động tạo balance** khi user được tạo
- ✅ **Idempotency** - Tránh xử lý trùng lặp giao dịch
- ✅ **Atomic operations** - Tất cả thao tác balance đều trong transaction
- ✅ **Queue-based processing** - Xử lý bất đồng bộ để tối ưu hiệu suất
- ✅ **Transaction history** - Lưu trữ đầy đủ lịch sử với metadata
- ✅ **Balance locking** - Hỗ trợ khóa balance để tránh race condition

---

## 🏗️ Kiến Trúc Module

Module Wallet tuân theo **Domain-Driven Design (DDD)** với 4 layers:

```
wallet/
├── domain/              # Business logic, entities, value objects
│   ├── entities/        # UserBalance, Transaction
│   ├── repositories/    # Repository interfaces
│   └── constants/       # Error definitions
├── application/         # Use cases, DTOs
│   ├── use-cases/       # CreditBalance, DebitBalance, GetBalance, etc.
│   └── dtos/            # Data Transfer Objects
├── infrastructure/      # Repository implementations, database
│   └── persistence/
│       ├── models/      # Database models
│       ├── repositories/# Repository implementations
│       └── migrations/  # Database migrations
└── interfaces/          # HTTP handlers, routes
    └── http/
```

### Domain Entities

#### 1. UserBalance
Quản lý số dư của người dùng:
- `UserID`: ID của người dùng
- `Balance`: Số dư hiện tại (đơn vị nhỏ nhất, ví dụ: cents)
- `Currency`: Loại tiền tệ (mặc định: USD)
- `IsDeleted`: Flag soft delete

**Methods:**
- `Credit(amount)`: Cộng tiền vào balance
- `Debit(amount)`: Trừ tiền từ balance
- `HasSufficientFunds(amount)`: Kiểm tra số dư có đủ không

#### 2. Transaction
Lưu trữ lịch sử giao dịch:
- `TransactionType`: `credit` hoặc `debit`
- `Status`: `pending`, `completed`, `failed`, `cancelled`
- `BalanceBefore`: Số dư trước giao dịch
- `BalanceAfter`: Số dư sau giao dịch
- `ReferenceID`: ID tham chiếu từ hệ thống ngoài (để idempotency)
- `Metadata`: Thông tin bổ sung dạng JSON

---

## 👤 Tạo User và User Balance

### 1. Tạo User (IAM Module)

Trước tiên, bạn cần tạo user thông qua IAM module:

```bash
POST /iam/register
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "secret123",
  "name": "John Doe",
  "phone_number": "+1234567890"
}
```

### 2. Tạo User Balance

Sau khi user được tạo, bạn có thể tạo balance cho user bằng cách:

#### Cách 1: Gọi API trực tiếp (Khuyến nghị)

```bash
POST /wallet/create-user-balance
Authorization: Bearer <access_token>
```

API này sẽ:
- ✅ Kiểm tra user có tồn tại không
- ✅ Kiểm tra balance đã tồn tại chưa (idempotent)
- ✅ Tạo balance mới với số dư = 0 nếu chưa có
- ✅ Trả về balance hiện tại

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "user_id": "uuid",
    "balance": 0,
    "currency": "USD",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

#### Cách 2: Tự động tạo khi Credit

Balance sẽ được tự động tạo khi bạn thực hiện credit lần đầu tiên cho user đó.

### 3. Database Schema

Bảng `user_balances`:
```sql
CREATE TABLE user_balances (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    balance BIGINT NOT NULL DEFAULT 0,
    currency VARCHAR(10) NOT NULL DEFAULT 'USD',
    is_locked BOOLEAN NOT NULL DEFAULT FALSE,
    is_deleted BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Đảm bảo mỗi user chỉ có 1 balance active
CREATE UNIQUE INDEX ux_user_balances_user_id_active
    ON user_balances(user_id)
    WHERE is_deleted = FALSE;
```

**Lưu ý quan trọng:**
- Mỗi user chỉ có **1 balance active** (unique constraint)
- Balance được tạo với `balance = 0` và `currency = 'USD'`
- Soft delete được sử dụng (`is_deleted` flag)

---

## 💰 Hoạt Động Của Hệ Thống Wallet

### Flow Credit (Nạp Tiền)

```
1. Client gọi API: POST /wallet/credit
   ↓
2. Handler xác thực JWT và lấy user_id
   ↓
3. Validate user tồn tại
   ↓
4. Enqueue job vào queue (wallet-credit)
   ↓
5. Trả về 202 Accepted ngay lập tức
   ↓
6. Queue Worker nhận job và xử lý:
   - Kiểm tra idempotency (tránh duplicate)
   - Lấy hoặc tạo balance
   - Credit balance trong transaction
   - Tạo transaction record
   - Commit transaction
```

### Flow Debit (Rút Tiền)

```
1. Client gọi API: POST /wallet/debit
   ↓
2. Handler xác thực JWT và lấy user_id
   ↓
3. Validate user tồn tại
   ↓
4. Enqueue job vào queue (wallet-debit)
   ↓
5. Trả về 202 Accepted ngay lập tức
   ↓
6. Queue Worker nhận job và xử lý:
   - Lấy balance (phải tồn tại)
   - Kiểm tra số dư đủ không
   - Debit balance trong transaction
   - Tạo transaction record
   - Commit transaction
```

### Điểm Khác Biệt Quan Trọng

| Thao Tác | Credit | Debit |
|----------|--------|-------|
| **Tự động tạo balance** | ✅ Có | ❌ Không (phải tồn tại) |
| **Kiểm tra số dư** | ❌ Không cần | ✅ Phải đủ số dư |
| **Idempotency** | ✅ Có (dùng ReferenceID hoặc hash) | ❌ Không |
| **Transaction Status** | `completed` ngay | `completed` ngay |

---

## 🔌 Các API Endpoints

### 1. Tạo User Balance

```http
POST /wallet/create-user-balance
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "user_id": "uuid",
    "balance": 0,
    "currency": "USD",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

### 2. Nạp Tiền (Credit)

```http
POST /wallet/credit
Authorization: Bearer <token>
Content-Type: application/json

{
  "amount": 10000,                    // Số tiền (đơn vị nhỏ nhất)
  "reference_id": "uuid-optional",    // Optional: Để idempotency
  "metadata": {                       // Optional: Thông tin bổ sung
    "source": "deposit",
    "payment_method": "bank_transfer"
  }
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "status": "enqueued"  // Job đã được đưa vào queue
  }
}
```

### 3. Rút Tiền (Debit)

```http
POST /wallet/debit
Authorization: Bearer <token>
Content-Type: application/json

{
  "amount": 5000,
  "reference_id": "uuid-optional",
  "metadata": {
    "purpose": "withdrawal",
    "bank_account": "xxx"
  }
}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "status": "enqueued"
  }
}
```

### 4. Xem Số Dư

```http
GET /wallet/balance
Authorization: Bearer <token>
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "user_id": "uuid",
    "balance": 10000,
    "currency": "USD",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

### 5. Lịch Sử Giao Dịch

```http
GET /wallet/transactions?limit=20&offset=0&transactionType=credit&startDate=2024-01-01&endDate=2024-01-31
Authorization: Bearer <token>
```

**Query Parameters:**
- `limit`: Số lượng kết quả (mặc định: 20)
- `offset`: Vị trí bắt đầu (mặc định: 0)
- `transactionType`: `credit` hoặc `debit` (optional)
- `startDate`: Ngày bắt đầu (ISO 8601) (optional)
- `endDate`: Ngày kết thúc (ISO 8601) (optional)

**Response:**
```json
{
  "success": true,
  "data": {
    "transactions": [
      {
        "id": "uuid",
        "user_id": "uuid",
        "transaction_type": "credit",
        "amount": 10000,
        "balance_before": 0,
        "balance_after": 10000,
        "reference_id": "uuid",
        "status": "completed",
        "metadata": {},
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-01T00:00:00Z"
      }
    ],
    "total": 1,
    "limit": 20,
    "offset": 0
  }
}
```

---

## 📊 Xử Lý Giao Dịch (Transactions)

### Transaction Types

- `credit`: Nạp tiền vào ví
- `debit`: Rút tiền từ ví

### Transaction Status

- `pending`: Giao dịch đang chờ xử lý
- `completed`: Giao dịch đã hoàn thành
- `failed`: Giao dịch thất bại
- `cancelled`: Giao dịch đã bị hủy

**Lưu ý:** Hiện tại, cả credit và debit đều tạo transaction với status `completed` ngay lập tức vì xử lý trong transaction atomic.

### Database Schema

```sql
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    transaction_type VARCHAR(20) NOT NULL,  -- 'credit' hoặc 'debit'
    amount BIGINT NOT NULL,
    balance_before BIGINT NOT NULL,
    balance_after BIGINT NOT NULL,
    reference_id UUID,                      -- Để idempotency
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    metadata JSONB,                         -- Thông tin bổ sung
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    
    FOREIGN KEY(user_id) REFERENCES users(id) ON DELETE CASCADE
);
```

### Metadata

Metadata là JSONB field cho phép lưu thông tin bổ sung:
- Nguồn giao dịch (`source`, `payment_method`)
- Thông tin ngân hàng
- Ghi chú, mô tả
- Bất kỳ thông tin nào khác

**Ví dụ:**
```json
{
  "source": "deposit",
  "payment_method": "bank_transfer",
  "bank_name": "Vietcombank",
  "transaction_fee": 0,
  "notes": "Deposit from bank account"
}
```

---

## 🚀 Hệ Thống Queue

### Tại Sao Sử Dụng Queue?

1. **Performance**: API trả về ngay (202 Accepted), không phải chờ xử lý
2. **Scalability**: Có thể scale workers độc lập với API server
3. **Reliability**: Jobs được retry tự động nếu thất bại
4. **Decoupling**: Tách biệt HTTP layer và business logic

### Queue Implementation

Sử dụng **Asynq** (Redis-based queue):

- **Queue Names:**
  - `wallet-credit`: Queue cho credit operations
  - `wallet-debit`: Queue cho debit operations

- **Job Types:**
  - `wallet:credit`: Credit job
  - `wallet:debit`: Debit job

### Cấu Hình Queue

```go
// Trong .env
REDIS_ADDRESS=localhost:6379
REDIS_PASSWORD=
REDIS_DATABASE=0
QUEUE_CONCURRENCY=10
```

### Worker Configuration

Worker được cấu hình với:
- **Concurrency**: 10 workers đồng thời
- **Queue Priority**: 
  - `wallet-credit`: Priority 7
  - `wallet-debit`: Priority 7
- **Retry**: Tối đa 3 lần retry
- **Timeout**: 30 giây mỗi job

### Chạy Worker

```bash
# Build worker
make build-worker

# Chạy worker
make worker
```

### Monitor Queue

Xem dashboard của Asynq:
```bash
make monitor
# Truy cập: http://localhost:9090/monitor
```

---

## 🧪 Hướng Dẫn Testing

Phần này dành cho **Tester** để test các chức năng của Wallet module.

### 📌 Chuẩn Bị Test Environment

#### 1. Kiểm Tra Services Đang Chạy

```bash
# 1. Kiểm tra API server
curl http://localhost:3003/health || echo "API server không chạy"

# 2. Kiểm tra Redis (cần cho queue)
redis-cli ping || echo "Redis không chạy"

# 3. Kiểm tra Worker (xử lý credit/debit jobs)
ps aux | grep worker || echo "Worker không chạy"
```

**Lưu ý:** Phải đảm bảo Worker đang chạy để xử lý credit/debit jobs!

#### 2. Setup Test User

```bash
# Tạo test user
curl -X POST http://localhost:3003/iam/register \
  -H "Content-Type: application/json" \
  -d '{
    "username": "test_user_001",
    "email": "testuser001@example.com",
    "password": "Test123456!",
    "name": "Test User",
    "phone_number": "+84123456789"
  }'

# Login để lấy token
curl -X POST http://localhost:3003/iam/login \
  -H "Content-Type: application/json" \
  -d '{
    "username": "test_user_001",
    "password": "Test123456!"
  }'

# Lưu token vào biến (thay YOUR_TOKEN)
export TOKEN="YOUR_TOKEN_HERE"
```

---

### ✅ Test Cases Chi Tiết

#### Test Case 1: Tạo User Balance

**Mục đích:** Test API tạo balance cho user

**Request:**
```bash
curl -X POST http://localhost:3003/wallet/create-user-balance \
  -H "Authorization: Bearer $TOKEN"
```

**Expected Response (201 Created):**
```json
{
  "success": true,
  "data": {
    "id": "uuid-string",
    "user_id": "uuid-string",
    "balance": 0,
    "currency": "USD",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  },
  "error": null,
  "timestamp": "2024-01-01T00:00:00Z"
}
```

**Test Steps:**
1. ✅ Gọi API lần đầu → Nhận 201 Created với balance = 0
2. ✅ Gọi API lần 2 (idempotent) → Vẫn nhận 201 với cùng data (không tạo duplicate)
3. ✅ Kiểm tra database: Mỗi user chỉ có 1 balance record

**Edge Cases:**
- ❌ Gọi API không có token → 401 Unauthorized
- ❌ Gọi API với token invalid → 401 Unauthorized
- ❌ User không tồn tại → 404 Not Found

---

#### Test Case 2: Credit (Nạp Tiền) - Happy Path

**Mục đích:** Test nạp tiền thành công

**Request:**
```bash
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 100000,
    "reference_id": "test-payment-001",
    "metadata": {
      "source": "test",
      "payment_method": "manual"
    }
  }'
```

**Expected Response (202 Accepted):**
```json
{
  "success": true,
  "data": {
    "status": "enqueued"
  },
  "error": null,
  "timestamp": "2024-01-01T00:00:00Z"
}
```

**Verification Steps:**
1. ✅ Nhận 202 Accepted ngay lập tức
2. ✅ Đợi 2-5 giây để worker xử lý
3. ✅ Gọi GET /wallet/balance → Balance tăng đúng 100000
4. ✅ Gọi GET /wallet/transactions → Có transaction mới với:
   - `transaction_type`: "credit"
   - `amount`: 100000
   - `balance_before`: 0 (hoặc số cũ)
   - `balance_after`: 100000 (hoặc số mới)
   - `status`: "completed"
   - `reference_id`: "test-payment-001"

**Expected Behavior:**
- Balance được cập nhật chính xác
- Transaction record được tạo với đầy đủ thông tin
- Metadata được lưu đúng

---

#### Test Case 3: Credit - Idempotency Test

**Mục đích:** Test không xử lý duplicate với cùng reference_id

**Steps:**
```bash
# Request 1
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "reference_id": "duplicate-test-001"
  }'

# Request 2 - Gửi lại với cùng reference_id (ngay lập tức)
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 50000,
    "reference_id": "duplicate-test-001"
  }'
```

**Expected:**
- ✅ Cả 2 requests đều trả về 202 Accepted
- ✅ Chỉ có 1 transaction được tạo với `reference_id: "duplicate-test-001"`
- ✅ Balance chỉ tăng 50000 (không phải 100000)

---

#### Test Case 4: Debit (Rút Tiền) - Happy Path

**Mục đích:** Test rút tiền thành công khi có đủ số dư

**Precondition:** User phải có balance > 0

**Request:**
```bash
curl -X POST http://localhost:3003/wallet/debit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 30000,
    "reference_id": "test-withdrawal-001",
    "metadata": {
      "purpose": "test_withdrawal",
      "bank_account": "1234567890"
    }
  }'
```

**Expected Response (202 Accepted):**
```json
{
  "success": true,
  "data": {
    "status": "enqueued"
  }
}
```

**Verification:**
1. ✅ Nhận 202 Accepted
2. ✅ Đợi 2-5 giây
3. ✅ GET /wallet/balance → Balance giảm đúng 30000
4. ✅ GET /wallet/transactions → Có debit transaction với status "completed"

---

#### Test Case 5: Debit - Insufficient Funds

**Mục đích:** Test debit khi không đủ số dư

**Precondition:** Balance = 50000

**Request:**
```bash
curl -X POST http://localhost:3003/wallet/debit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 100000
  }'
```

**Expected Response (Sau khi worker xử lý):**
- ❌ Transaction không được tạo (hoặc status = "failed")
- ❌ Balance không thay đổi
- ⚠️ Có thể có error trong worker logs

**Note:** API vẫn trả về 202, nhưng worker sẽ reject và không debit.

---

#### Test Case 6: Debit - Balance Not Found

**Mục đích:** Test debit khi chưa có balance

**Precondition:** User chưa có balance (chưa gọi create-user-balance)

**Request:**
```bash
curl -X POST http://localhost:3003/wallet/debit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 10000
  }'
```

**Expected:**
- ❌ Worker sẽ fail với lỗi "Balance not found"
- ❌ Transaction không được tạo
- ⚠️ Check worker logs để xác nhận

---

#### Test Case 7: Get Balance

**Mục đích:** Test API lấy số dư

**Request:**
```bash
curl -X GET http://localhost:3003/wallet/balance \
  -H "Authorization: Bearer $TOKEN"
```

**Expected Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "id": "uuid",
    "user_id": "uuid",
    "balance": 70000,
    "currency": "USD",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T00:00:00Z"
  }
}
```

**Test Steps:**
1. ✅ Balance chưa tồn tại → 404 Not Found
2. ✅ Balance tồn tại → 200 OK với đúng số dư
3. ✅ Kiểm tra balance sau mỗi credit/debit → Phải chính xác

**Edge Cases:**
- ❌ Không có token → 401 Unauthorized
- ❌ Token invalid → 401 Unauthorized

---

#### Test Case 8: Get Transactions - Basic

**Mục đích:** Test API lấy lịch sử giao dịch

**Request:**
```bash
curl -X GET "http://localhost:3003/wallet/transactions?limit=10&offset=0" \
  -H "Authorization: Bearer $TOKEN"
```

**Expected Response (200 OK):**
```json
{
  "success": true,
  "data": {
    "transactions": [
      {
        "id": "uuid",
        "user_id": "uuid",
        "transaction_type": "credit",
        "amount": 100000,
        "balance_before": 0,
        "balance_after": 100000,
        "reference_id": "test-payment-001",
        "status": "completed",
        "metadata": {},
        "created_at": "2024-01-01T00:00:00Z",
        "updated_at": "2024-01-01T00:00:00Z"
      }
    ],
    "total": 1,
    "limit": 10,
    "offset": 0
  }
}
```

**Verification:**
- ✅ Transactions được sắp xếp theo thời gian mới nhất trước
- ✅ Pagination hoạt động đúng (limit, offset)
- ✅ Total đúng với số lượng thực tế

---

#### Test Case 9: Get Transactions - Filtering

**Mục đích:** Test filter transactions theo type và date

**Requests:**
```bash
# Filter by type
curl -X GET "http://localhost:3003/wallet/transactions?transactionType=credit" \
  -H "Authorization: Bearer $TOKEN"

# Filter by date range
curl -X GET "http://localhost:3003/wallet/transactions?startDate=2024-01-01T00:00:00Z&endDate=2024-01-31T23:59:59Z" \
  -H "Authorization: Bearer $TOKEN"

# Combined filters
curl -X GET "http://localhost:3003/wallet/transactions?transactionType=debit&startDate=2024-01-01T00:00:00Z&limit=5" \
  -H "Authorization: Bearer $TOKEN"
```

**Expected:**
- ✅ `transactionType=credit` → Chỉ trả về credit transactions
- ✅ `transactionType=debit` → Chỉ trả về debit transactions
- ✅ Date range filter hoạt động đúng
- ✅ Combined filters hoạt động đúng

---

### 🔍 Edge Cases & Negative Testing

#### 1. Invalid Amount

```bash
# Amount = 0
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount": 0}'
# Expected: 400 Bad Request

# Amount < 0
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount": -1000}'
# Expected: 400 Bad Request

# Amount không phải số
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount": "invalid"}'
# Expected: 400 Bad Request
```

#### 2. Invalid Reference ID Format

```bash
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 10000,
    "reference_id": "not-a-uuid"
  }'
# Expected: 400 Bad Request với message "invalid reference id format"
```

#### 3. Missing Required Fields

```bash
# Thiếu amount
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{}'
# Expected: 400 Bad Request
```

#### 4. Large Metadata

```bash
# Metadata quá lớn (test limit)
curl -X POST http://localhost:3003/wallet/credit \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 10000,
    "metadata": {
      "large_field": "'$(python3 -c "print('x' * 10000)")'"
    }
  }'
# Expected: Có thể 400 hoặc 500 tùy implementation
```

#### 5. Concurrent Requests (Race Condition)

```bash
# Gửi nhiều debit requests cùng lúc
for i in {1..10}; do
  curl -X POST http://localhost:3003/wallet/debit \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d "{\"amount\": 1000}" &
done
wait

# Expected: Chỉ các requests đủ số dư mới thành công
# Balance phải đúng (không bị negative)
```

---

### 📋 Test Checklist

Sử dụng checklist này để đảm bảo test đầy đủ:

#### ✅ Authentication & Authorization
- [ ] Test tất cả APIs không có token → 401
- [ ] Test với token invalid → 401
- [ ] Test với token expired → 401
- [ ] Test với token của user khác (nếu có check) → 403 hoặc 404

#### ✅ Create User Balance
- [ ] Tạo balance lần đầu → 201 Created
- [ ] Tạo balance lần 2 (idempotent) → 201 Created (cùng data)
- [ ] Kiểm tra balance = 0 khi tạo mới
- [ ] Kiểm tra currency = "USD"

#### ✅ Credit Operations
- [ ] Credit với amount hợp lệ → 202 Accepted
- [ ] Credit tăng balance đúng số tiền
- [ ] Credit tạo transaction record đúng
- [ ] Credit với reference_id → Idempotency hoạt động
- [ ] Credit không có reference_id → Vẫn hoạt động
- [ ] Credit với metadata → Metadata được lưu
- [ ] Credit amount = 0 → 400 Bad Request
- [ ] Credit amount < 0 → 400 Bad Request
- [ ] Credit với invalid JSON → 400 Bad Request

#### ✅ Debit Operations
- [ ] Debit với đủ số dư → 202 Accepted
- [ ] Debit giảm balance đúng số tiền
- [ ] Debit tạo transaction record đúng
- [ ] Debit không đủ số dư → Job fail, balance không đổi
- [ ] Debit khi chưa có balance → Job fail
- [ ] Debit amount = 0 → 400 Bad Request
- [ ] Debit amount < 0 → 400 Bad Request

#### ✅ Get Balance
- [ ] Get balance khi có balance → 200 OK với đúng số dư
- [ ] Get balance khi chưa có → 404 Not Found
- [ ] Balance update sau credit → Get balance phản ánh đúng
- [ ] Balance update sau debit → Get balance phản ánh đúng

#### ✅ Get Transactions
- [ ] Get transactions cơ bản → 200 OK
- [ ] Pagination hoạt động (limit, offset)
- [ ] Filter by transactionType → Đúng loại
- [ ] Filter by date range → Đúng range
- [ ] Combined filters → Hoạt động đúng
- [ ] Empty result → Trả về array rỗng

#### ✅ Integration Tests
- [ ] Flow hoàn chỉnh: Register → Create Balance → Credit → Debit → Check Balance
- [ ] Multiple credits → Balance tích lũy đúng
- [ ] Multiple debits → Balance giảm đúng
- [ ] Credit rồi debit ngay → Balance đúng
- [ ] Concurrent requests → Không có race condition

#### ✅ Queue & Worker
- [ ] Worker đang chạy khi test
- [ ] Jobs được xử lý (check trong Asynq monitor)
- [ ] Failed jobs được retry (nếu có)
- [ ] Job timeout hoạt động (30s)

#### ✅ Database Integrity
- [ ] Mỗi user chỉ có 1 balance active
- [ ] Transactions có balance_before và balance_after đúng
- [ ] Metadata được lưu đúng format JSONB
- [ ] Timestamps được set đúng (created_at, updated_at)

---

### 🛠️ Tools & Commands Hữu Ích

#### 1. Test Script Template

Tạo file `test_wallet.sh`:
```bash
#!/bin/bash

BASE_URL="http://localhost:3003"
TOKEN="YOUR_TOKEN_HERE"

echo "=== Testing Wallet APIs ==="

# Test Create Balance
echo "1. Testing Create Balance..."
curl -X POST "$BASE_URL/wallet/create-user-balance" \
  -H "Authorization: Bearer $TOKEN" \
  -w "\nHTTP Status: %{http_code}\n\n"

# Test Credit
echo "2. Testing Credit..."
curl -X POST "$BASE_URL/wallet/credit" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"amount": 100000, "reference_id": "test-001"}' \
  -w "\nHTTP Status: %{http_code}\n\n"

sleep 3  # Đợi worker xử lý

# Test Get Balance
echo "3. Testing Get Balance..."
curl -X GET "$BASE_URL/wallet/balance" \
  -H "Authorization: Bearer $TOKEN" \
  -w "\nHTTP Status: %{http_code}\n\n"

# Test Get Transactions
echo "4. Testing Get Transactions..."
curl -X GET "$BASE_URL/wallet/transactions?limit=10" \
  -H "Authorization: Bearer $TOKEN" \
  -w "\nHTTP Status: %{http_code}\n\n"
```

#### 2. Monitor Queue

```bash
# Xem queue trong Redis
redis-cli
> LLEN asynq:wallet-credit
> LLEN asynq:wallet-debit
> KEYS asynq:*

# Hoặc dùng Asynq Monitor
make monitor
# Truy cập: http://localhost:9090/monitor
```

#### 3. Check Database

```sql
-- Xem tất cả balances
SELECT id, user_id, balance, currency, created_at 
FROM user_balances 
ORDER BY created_at DESC;

-- Xem transactions của user
SELECT id, transaction_type, amount, balance_before, balance_after, status, created_at
FROM transactions
WHERE user_id = 'YOUR_USER_UUID'
ORDER BY created_at DESC;

-- Kiểm tra balance calculation
SELECT 
  ub.user_id,
  ub.balance as current_balance,
  SUM(CASE WHEN t.transaction_type = 'credit' THEN t.amount ELSE 0 END) as total_credit,
  SUM(CASE WHEN t.transaction_type = 'debit' THEN t.amount ELSE 0 END) as total_debit,
  (SUM(CASE WHEN t.transaction_type = 'credit' THEN t.amount ELSE 0 END) - 
   SUM(CASE WHEN t.transaction_type = 'debit' THEN t.amount ELSE 0 END)) as calculated_balance
FROM user_balances ub
LEFT JOIN transactions t ON t.user_id = ub.user_id AND t.status = 'completed'
WHERE ub.user_id = 'YOUR_USER_UUID'
GROUP BY ub.user_id, ub.balance;
```

---

### 🐛 Common Issues When Testing

#### Issue 1: Balance Không Update Sau Credit/Debit

**Nguyên nhân:** Worker không chạy hoặc job fail

**Cách fix:**
1. Kiểm tra worker đang chạy: `ps aux | grep worker`
2. Xem logs của worker
3. Check queue trong Redis hoặc Asynq monitor

#### Issue 2: API Trả Về 202 Nhưng Không Có Transaction

**Nguyên nhân:** Worker xử lý chậm hoặc job fail

**Cách fix:**
1. Đợi 5-10 giây rồi check lại
2. Xem worker logs để tìm lỗi
3. Check database xem có transaction không

#### Issue 3: Duplicate Transactions

**Nguyên nhân:** Không dùng reference_id hoặc idempotency không hoạt động

**Cách fix:**
- Luôn dùng reference_id khi test credit
- Test idempotency với cùng reference_id

---

### 📝 Test Report Template

Khi báo cáo bug, include các thông tin sau:

```
**Bug Title:** [Mô tả ngắn gọn]

**Test Case:** TC-XXX

**Steps to Reproduce:**
1. ...
2. ...
3. ...

**Expected:** [Kết quả mong đợi]

**Actual:** [Kết quả thực tế]

**Environment:**
- API URL: http://localhost:3003
- Worker Status: [Running/Stopped]
- Redis Status: [Connected/Disconnected]

**Request:**
```json
{
  ...
}
```

**Response:**
```json
{
  ...
}
```

**Screenshots/Logs:**
[Attach nếu có]
```

---

## ⚠️ Các Vấn Đề Thường Gặp

### 1. Balance Not Found

**Nguyên nhân:**
- User chưa có balance được tạo
- Balance đã bị soft delete

**Giải pháp:**
```bash
# Tạo balance cho user
POST /wallet/create-user-balance
```

### 2. Insufficient Funds

**Nguyên nhân:**
- Số dư không đủ để thực hiện debit

**Giải pháp:**
- Kiểm tra số dư trước khi debit:
```bash
GET /wallet/balance
```
- Nạp thêm tiền nếu cần

### 3. Duplicate Transaction

**Nguyên nhân:**
- Client gửi request trùng lặp
- Network timeout dẫn đến retry

**Giải pháp:**
- Sử dụng `reference_id` để đảm bảo idempotency:
```json
{
  "amount": 10000,
  "reference_id": "unique-payment-id-12345"
}
```

### 4. Job Không Được Xử Lý

**Nguyên nhân:**
- Worker không chạy
- Redis không kết nối được
- Queue bị đầy

**Giải pháp:**
1. Kiểm tra worker đang chạy:
```bash
ps aux | grep worker
```

2. Kiểm tra Redis connection:
```bash
redis-cli ping
```

3. Kiểm tra queue trong monitor:
```bash
make monitor
```

### 5. Transaction Status Luôn Là "pending"

**Nguyên nhân:**
- Worker không xử lý được job
- Database transaction rollback

**Giải pháp:**
- Xem logs của worker để tìm lỗi
- Kiểm tra database connection
- Đảm bảo worker đang chạy

### 6. Race Condition Khi Debit Đồng Thời

**Nguyên nhân:**
- Nhiều request debit cùng lúc
- Balance bị update không đúng

**Giải pháp:**
- Hệ thống đã được thiết kế với database transactions và locking
- Tất cả operations đều trong transaction để đảm bảo atomicity

---

## ✅ Best Practices

### 1. Luôn Tạo Balance Khi Tạo User

```go
// Trong IAM module, sau khi tạo user thành công:
// Tự động tạo balance
walletService.CreateUserBalance(ctx, userID)
```

### 2. Sử Dụng Reference ID Cho Idempotency

```json
{
  "amount": 10000,
  "reference_id": "payment-abc-123-xyz"
}
```

### 3. Xử Lý Response 202 Accepted

API trả về `202 Accepted` nghĩa là job đã được đưa vào queue, không phải đã xử lý xong:
- ✅ Job sẽ được xử lý bất đồng bộ
- ⚠️ Cần polling hoặc webhook để biết kết quả
- 📊 Có thể check transaction status qua API

### 4. Validate Amount

- Luôn validate `amount > 0`
- Sử dụng đơn vị nhỏ nhất (cents) để tránh lỗi floating point

### 5. Sử Dụng Metadata Để Audit

```json
{
  "amount": 10000,
  "metadata": {
    "source": "deposit",
    "payment_gateway": "stripe",
    "payment_id": "pi_xxx",
    "customer_id": "cus_xxx"
  }
}
```

### 6. Kiểm Tra Balance Trước Khi Debit

```bash
# 1. Get balance
GET /wallet/balance

# 2. Kiểm tra đủ không
if balance >= amount:
    # 3. Debit
    POST /wallet/debit
```

### 7. Monitoring và Logging

- Monitor queue size qua Asynq dashboard
- Log tất cả transactions để audit
- Alert khi có lỗi từ worker

### 8. Error Handling

- Luôn handle các error từ API
- Retry logic cho network errors
- Check transaction status sau khi enqueue

---

## 📝 Ví Dụ Sử Dụng Hoàn Chỉnh

### Scenario: User Đăng Ký và Nạp Tiền

```bash
# 1. Đăng ký user
POST /iam/register
{
  "username": "player1",
  "email": "player1@example.com",
  "password": "secret123",
  "name": "Player One",
  "phone_number": "+84123456789"
}

# Response: { "success": true, "data": { "id": "user-uuid", ... } }

# 2. Login để lấy token
POST /iam/login
{
  "username": "player1",
  "password": "secret123"
}

# Response: { "access_token": "jwt-token", ... }

# 3. Tạo balance cho user
POST /wallet/create-user-balance
Authorization: Bearer jwt-token

# Response: { "success": true, "data": { "balance": 0, ... } }

# 4. Nạp tiền 100,000 (đơn vị nhỏ nhất, ví dụ: 100000 cents = 1000 USD)
POST /wallet/credit
Authorization: Bearer jwt-token
{
  "amount": 100000,
  "reference_id": "payment-12345",
  "metadata": {
    "source": "bank_transfer",
    "bank_name": "Vietcombank"
  }
}

# Response: { "success": true, "data": { "status": "enqueued" } }

# 5. Kiểm tra số dư (sau vài giây)
GET /wallet/balance
Authorization: Bearer jwt-token

# Response: { "success": true, "data": { "balance": 100000, ... } }

# 6. Xem lịch sử giao dịch
GET /wallet/transactions?limit=10
Authorization: Bearer jwt-token

# Response: { "success": true, "data": { "transactions": [...], ... } }

# 7. Rút tiền 50,000
POST /wallet/debit
Authorization: Bearer jwt-token
{
  "amount": 50000,
  "reference_id": "withdrawal-67890",
  "metadata": {
    "purpose": "withdrawal",
    "bank_account": "xxx"
  }
}

# Response: { "success": true, "data": { "status": "enqueued" } }

# 8. Kiểm tra số dư cuối cùng
GET /wallet/balance
Authorization: Bearer jwt-token

# Response: { "success": true, "data": { "balance": 50000, ... } }
```

---

## 🔍 Debugging

### Kiểm Tra Worker Logs

```bash
# Xem logs của worker
tail -f logs/worker.log
```

### Kiểm Tra Queue

```bash
# Connect Redis
redis-cli

# Xem jobs trong queue
LLEN asynq:wallet-credit
LLEN asynq:wallet-debit
```

### Kiểm Tra Database

```sql
-- Xem tất cả balances
SELECT * FROM user_balances;

-- Xem transactions gần đây
SELECT * FROM transactions 
ORDER BY created_at DESC 
LIMIT 10;

-- Xem balance của user cụ thể
SELECT * FROM user_balances 
WHERE user_id = 'uuid';
```

---

## 📚 Tài Liệu Tham Khảo

- [Asynq Documentation](https://github.com/hibiken/asynq)
- [PostgreSQL Transactions](https://www.postgresql.org/docs/current/tutorial-transactions.html)
- [Idempotency Patterns](https://stripe.com/docs/api/idempotent_requests)

---

## ❓ FAQ

**Q: Tại sao API trả về 202 thay vì 200?**  
A: Vì job được xử lý bất đồng bộ qua queue. 202 Accepted nghĩa là request đã được chấp nhận và sẽ được xử lý sau.

**Q: Làm sao biết transaction đã thành công?**  
A: Poll API `/wallet/transactions` với `reference_id` để kiểm tra status.

**Q: Có thể debit mà không có balance không?**  
A: Không. Debit yêu cầu balance phải tồn tại và có đủ số dư.

**Q: Reference ID có bắt buộc không?**  
A: Không, nhưng nên dùng để đảm bảo idempotency và tránh duplicate transactions.

**Q: Metadata có giới hạn không?**  
A: Không có giới hạn cứng, nhưng nên giữ nhỏ gọn vì được lưu trong database.

---

**Tài liệu này được tạo bởi: Ludo Backend Team**  
**Phiên bản: 1.0**  
**Cập nhật: 2024**

