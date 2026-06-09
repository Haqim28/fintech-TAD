# API Specification — PT. XYZ Loan App

## Base URL
```
https://api.ptxyz.com/v1
```

## Authentication
Semua endpoint (kecuali auth) menggunakan **Bearer JWT** di header:
```
Authorization: Bearer <access_token>
```

---

## 1. Auth Service

### POST `/auth/register`
Registrasi pengguna baru.

**Request Body:**
```json
{
  "full_name": "Budi Santoso",
  "email": "budi@email.com",
  "phone_number": "08123456789",
  "ktp_number": "3201234567890001"
}
```

**Response `201`:**
```json
{
  "user_id": "uuid",
  "message": "OTP sent to 08123456789"
}
```

---

### POST `/auth/otp/verify`
Verifikasi OTP registrasi.

**Request Body:**
```json
{
  "user_id": "uuid",
  "otp_code": "123456"
}
```

**Response `200`:**
```json
{
  "message": "OTP verified. Please set your password."
}
```

---

### POST `/auth/login`
Login dengan password.

**Request Body:**
```json
{
  "email": "budi@email.com",
  "password": "SecurePass123!"
}
```

**Response `200`:**
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "eyJhbGci...",
  "expires_in": 3600
}
```

**Error `401`:**
```json
{
  "error": "Invalid credentials"
}
```

---

### POST `/auth/login/biometric`
Login dengan biometric token.

**Request Body:**
```json
{
  "device_id": "device-uuid",
  "token_hash": "hashed-biometric-token"
}
```

**Response `200`:**
```json
{
  "access_token": "eyJhbGci...",
  "refresh_token": "eyJhbGci...",
  "expires_in": 3600
}
```

---

### POST `/auth/refresh`
Refresh access token.

**Request Body:**
```json
{
  "refresh_token": "eyJhbGci..."
}
```

---

### POST `/auth/logout`
Invalidate session.

**Response `200`:**
```json
{
  "message": "Logged out successfully"
}
```

---

## 2. User Service

### GET `/users/me`
Ambil profil pengguna.

**Response `200`:**
```json
{
  "id": "uuid",
  "full_name": "Budi Santoso",
  "email": "budi@email.com",
  "phone_number": "08123456789",
  "ktp_number": "3201234567890001",
  "photo_url": "https://cdn.ptxyz.com/photos/uuid.jpg",
  "ktp_photo_url": "https://cdn.ptxyz.com/ktp/uuid.jpg",
  "status": "ACTIVE",
  "created_at": "2026-01-01T00:00:00Z"
}
```

---

### PUT `/users/me`
Update data diri.

**Request Body:**
```json
{
  "full_name": "Budi Santoso Updated",
  "phone_number": "08987654321"
}
```

---

### POST `/users/me/kyc`
Upload foto & KTP untuk KYC. (multipart/form-data)

**Form Fields:**
- `photo` — file selfie dengan KTP
- `ktp_photo` — file foto KTP

**Response `200`:**
```json
{
  "message": "KYC documents uploaded successfully",
  "status": "PENDING_REVIEW"
}
```

---

## 3. Loan Service

### GET `/loans`
Daftar semua pinjaman milik user.

**Query Params:**
- `status` — filter: `PENDING`, `APPROVED`, `ACTIVE`, `COMPLETED`, `REJECTED`
- `page` — default: 1
- `limit` — default: 10

**Response `200`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "requested_amount": 5000000,
      "tenor_months": 6,
      "approved_amount": 5000000,
      "interest_rate": 1.5,
      "status": "ACTIVE",
      "applied_at": "2026-06-01T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 1
  }
}
```

---

### POST `/loans/apply`
Ajukan pinjaman baru.

**Request Body:**
```json
{
  "requested_amount": 5000000,
  "tenor_months": 6
}
```

**Validasi:**
- `requested_amount`: min Rp 1.000.000, max Rp 12.000.000
- `tenor_months`: 3, 6, 9, atau 12
- User tidak boleh punya pinjaman aktif/pending

**Response `201`:**
```json
{
  "loan_id": "uuid",
  "status": "PENDING",
  "message": "Loan application submitted. Processing..."
}
```

**Error `400` — Active loan:**
```json
{
  "error": "Cannot apply for a new loan while you have an active loan"
}
```

---

### GET `/loans/:id`
Detail pinjaman tertentu.

**Response `200`:**
```json
{
  "id": "uuid",
  "requested_amount": 5000000,
  "tenor_months": 6,
  "approved_amount": 5000000,
  "interest_rate": 1.5,
  "status": "ACTIVE",
  "applied_at": "2026-06-01T10:00:00Z",
  "processed_at": "2026-06-01T10:05:00Z",
  "scoring": {
    "credit_score": 720,
    "is_approved": true
  }
}
```

---

### GET `/loans/:id/schedules`
Jadwal cicilan pinjaman.

**Response `200`:**
```json
{
  "loan_id": "uuid",
  "schedules": [
    {
      "id": "uuid",
      "installment_number": 1,
      "due_date": "2026-07-25",
      "principal_amount": 833333,
      "interest_amount": 75000,
      "total_amount": 908333,
      "remaining_balance": 4166667,
      "status": "PENDING"
    }
  ]
}
```

---

## 4. Payment Service

### GET `/payments`
Riwayat pembayaran user.

**Response `200`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "loan_id": "uuid",
      "schedule_id": "uuid",
      "amount_paid": 908333,
      "payment_method": "VIRTUAL_ACCOUNT",
      "transaction_ref": "TXN-2026060901",
      "paid_at": "2026-06-09T14:00:00Z",
      "status": "SUCCESS"
    }
  ]
}
```

---

### POST `/payments`
Lakukan pembayaran cicilan.

**Request Body:**
```json
{
  "loan_id": "uuid",
  "schedule_id": "uuid",
  "amount": 908333,
  "payment_method": "VIRTUAL_ACCOUNT"
}
```

**Payment Methods:** `BANK_TRANSFER`, `VIRTUAL_ACCOUNT`, `GOPAY`, `OVO`

**Response `200`:**
```json
{
  "payment_id": "uuid",
  "transaction_ref": "TXN-2026060901",
  "status": "SUCCESS",
  "message": "Payment successful"
}
```

---

## 5. Notification Service

### GET `/notifications`
Daftar notifikasi user.

**Response `200`:**
```json
{
  "data": [
    {
      "id": "uuid",
      "channel": "EMAIL",
      "subject": "Pinjaman Anda Disetujui",
      "is_sent": true,
      "sent_at": "2026-06-01T10:05:00Z"
    }
  ]
}
```

---

## Error Codes

| Code | Meaning |
|------|---------|
| `400` | Bad Request — validasi gagal |
| `401` | Unauthorized — token tidak valid / expired |
| `403` | Forbidden — akses ditolak |
| `404` | Not Found — resource tidak ditemukan |
| `429` | Too Many Requests — rate limit |
| `500` | Internal Server Error |
