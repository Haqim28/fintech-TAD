# PT. XYZ Loan App — System Design

---

## 📋 Table of Contents

- [Overview](#overview)
- [1. High-Level Architecture](#1-high-level-architecture)
- [2. Screen Flow](#2-screen-flow)
- [3. Entity Relationship Diagram (ERD)](#3-entity-relationship-diagram-erd)
- [4. API Design](#4-api-design)
- [5. Screen Behavior](#5-screen-behavior)
- [Tech Stack](#tech-stack)

---

## Overview

PT. XYZ adalah perusahaan fintech yang mengembangkan aplikasi pinjaman online (mobile). Dokumen ini mencakup desain arsitektur sistem secara menyeluruh mulai dari high-level architecture, screen flow, ERD, hingga detail API dan screen behavior.

### Key Business Rules
- Pinjaman maksimal **Rp 12.000.000** dengan tenor maksimal **12 bulan**
- User **tidak dapat** mengajukan pinjaman baru jika masih ada pinjaman aktif
- Setiap pengajuan diproses melalui **credit scoring** (BI Checking)
- Notifikasi dikirim via **email dan SMS** setelah keputusan kredit

---

## 1. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│            Mobile Client Layer (React Native / Flutter)      │
│   [Auth Screen] [Dashboard] [Loan Request] [Repayment]       │
└──────────────────────┬───────────────────────────────────────┘
                       │ HTTPS / REST
┌──────────────────────▼───────────────────────────────────────┐
│    API Gateway — Auth Middleware · Rate Limiting · SSL        │
└──────────────────────┬───────────────────────────────────────┘
                       │
┌──────────────────────▼───────────────────────────────────────┐
│                  Backend Services Layer                       │
│  [User Svc] [Loan Svc] [Scoring Svc] [Notif Svc] [Pay Svc]  │
└─────────┬────────────────────────────────────┬───────────────┘
          │                                    │
┌─────────▼──────────┐              ┌──────────▼──────────────┐
│     Data Layer     │              │    External Services     │
│  PostgreSQL        │              │  SMS Gateway             │
│  Redis (cache)     │              │  Email (SES)             │
│  S3 (KTP/foto)     │              │  KYC / BI Checking API   │
└─────────┬──────────┘              └──────────┬──────────────┘
          │                                    │
┌─────────▼────────────────────────────────────▼──────────────┐
│   Message Queue (RabbitMQ / Kafka)                           │
│   Async events: approval · notification · disbursement       │
└──────────────────────────────────────────────────────────────┘
```

### Komponen Utama

| Layer | Teknologi | Fungsi |
|-------|-----------|--------|
| Mobile Client | React Native / Flutter | UI aplikasi iOS & Android |
| API Gateway | Kong / AWS API GW | Auth, rate limiting, routing |
| User Service | Node.js / Go | Registrasi, profil, KYC |
| Loan Service | Node.js / Go | Pengajuan & manajemen pinjaman |
| Scoring Service | Python | Credit scoring, BI Checking |
| Notification Service | Node.js | Email & SMS dispatcher |
| Payment Service | Node.js / Go | Pembayaran cicilan |
| Database | PostgreSQL | Data utama |
| Cache | Redis | Session & cache |
| Storage | AWS S3 | Foto & dokumen KTP |
| Message Queue | RabbitMQ / Kafka | Async event processing |

---

## 2. Screen Flow

```
[Splash Screen]
      │
[Welcome Screen]
  ├─ Daftar ──► [1. Data Diri] ──► [2. Upload KTP + Foto]
  │              ──► [3. Verifikasi OTP] ──► [4. Set Password]
  │              ──► [Registrasi Sukses]
  │                        │
  └─ Login ──► [Login Screen (Password / Biometric)]
               ── Validasi Auth ──►
                        │
               ┌────────▼────────┐
               │    DASHBOARD    │
               │ Saldo · Tagihan │
               └──┬──────────┬──┘
                  │          │
          [Ajukan Pinjaman]  [Bayar Cicilan]
                  │          │
         [Form Pinjaman]  [Detail Tagihan]
         [Review + Konfirmasi] [Pilih Metode Bayar]
         [Proses Scoring]   [Pembayaran Berhasil ✓]
              │
         ┌────┴────┐
    [Disetujui]  [Ditolak]
    [Notif Email]  [Tampil Alasan]
    [+ SMS]
```

### Screen List

| Screen | Deskripsi |
|--------|-----------|
| Splash Screen | Loading + cek session |
| Welcome Screen | CTA Login / Daftar |
| Registrasi | 4-step: Data Diri → KTP/Foto → OTP → Password |
| Login | Email/HP + Password atau Biometric |
| Dashboard | Saldo hutang, tagihan bulan ini, riwayat |
| Form Pinjaman | Slider nominal + pilih tenor |
| Review Pinjaman | Ringkasan + syarat & ketentuan |
| Status Pinjaman | Hasil proses (approved/rejected) |
| Detail Tagihan | Jadwal cicilan dengan status |
| Pilih Pembayaran | Metode: Transfer / VA / e-Wallet |

---

## 3. Entity Relationship Diagram (ERD)

> File PlantUML tersedia di [`docs/erd.puml`](docs/erd.puml)

```
USERS ||--o{ LOAN_APPLICATIONS   : "mengajukan"
USERS ||--o{ BIOMETRIC_TOKENS    : "memiliki"
USERS ||--o{ NOTIFICATIONS       : "menerima"

LOAN_APPLICATIONS ||--|| SCORING_RESULTS  : "dinilai oleh"
LOAN_APPLICATIONS ||--o{ LOAN_SCHEDULES   : "memiliki"
LOAN_APPLICATIONS ||--o{ PAYMENTS         : "dibayar melalui"

LOAN_SCHEDULES    ||--o{ PAYMENTS         : "dilunasi oleh"
```

### Tabel Utama

**USERS**
```sql
id              UUID        PRIMARY KEY
full_name       VARCHAR(100)
email           VARCHAR(100) UNIQUE
phone_number    VARCHAR(20)  UNIQUE
password_hash   VARCHAR(255)
ktp_number      VARCHAR(20)  UNIQUE
photo_url       VARCHAR(255)
ktp_photo_url   VARCHAR(255)
status          ENUM('PENDING','ACTIVE','SUSPENDED')
created_at      TIMESTAMP
updated_at      TIMESTAMP
```

**LOAN_APPLICATIONS**
```sql
id                UUID        PRIMARY KEY
user_id           UUID        REFERENCES users(id)
requested_amount  DECIMAL(15,2)
tenor_months      INT         -- 3, 6, 9, atau 12
approved_amount   DECIMAL(15,2)
interest_rate     DECIMAL(5,2)
status            ENUM('PENDING','APPROVED','REJECTED','ACTIVE','COMPLETED')
rejection_reason  TEXT
applied_at        TIMESTAMP
processed_at      TIMESTAMP
```

**LOAN_SCHEDULES**
```sql
id                  UUID        PRIMARY KEY
loan_id             UUID        REFERENCES loan_applications(id)
installment_number  INT
due_date            DATE
principal_amount    DECIMAL(15,2)
interest_amount     DECIMAL(15,2)
total_amount        DECIMAL(15,2)
remaining_balance   DECIMAL(15,2)
status              ENUM('PENDING','PAID','OVERDUE')
```

**PAYMENTS**
```sql
id               UUID        PRIMARY KEY
loan_id          UUID        REFERENCES loan_applications(id)
schedule_id      UUID        REFERENCES loan_schedules(id)
amount_paid      DECIMAL(15,2)
payment_method   VARCHAR(50) -- BANK_TRANSFER, VIRTUAL_ACCOUNT, GOPAY, OVO
transaction_ref  VARCHAR(100)
paid_at          TIMESTAMP
status           ENUM('SUCCESS','FAILED','PENDING')
```

---

## 4. API Design

> Spesifikasi lengkap tersedia di [`api/api-spec.md`](api/api-spec.md)  
> Sequence Diagram tersedia di [`docs/sequence-diagram.puml`](docs/sequence-diagram.puml)

### Endpoint Summary

| Method | Endpoint | Auth | Deskripsi |
|--------|----------|------|-----------|
| `POST` | `/auth/register` | — | Daftar pengguna baru |
| `POST` | `/auth/otp/verify` | — | Verifikasi OTP |
| `POST` | `/auth/login` | — | Login dengan password |
| `POST` | `/auth/login/biometric` | — | Login dengan biometric |
| `POST` | `/auth/refresh` | Refresh Token | Refresh JWT |
| `POST` | `/auth/logout` | Bearer JWT | Logout |
| `GET` | `/users/me` | Bearer JWT | Profil user |
| `PUT` | `/users/me` | Bearer JWT | Update profil |
| `POST` | `/users/me/kyc` | Bearer JWT | Upload KTP + foto |
| `GET` | `/loans` | Bearer JWT | Daftar pinjaman |
| `POST` | `/loans/apply` | Bearer JWT | Ajukan pinjaman baru |
| `GET` | `/loans/:id` | Bearer JWT | Detail pinjaman |
| `GET` | `/loans/:id/schedules` | Bearer JWT | Jadwal cicilan |
| `GET` | `/payments` | Bearer JWT | Riwayat pembayaran |
| `POST` | `/payments` | Bearer JWT | Bayar cicilan |
| `GET` | `/notifications` | Bearer JWT | Daftar notifikasi |

### Loan Application Flow (Sequence)

```
Mobile App → API Gateway → Loan Service
                               │
                     Cek active loan?
                         ├── Ya → 400 Error
                         └── Tidak
                               │
                          Scoring Service
                          (BI Check + Score)
                               │
                    ┌──────────┴──────────┐
                  Approved             Rejected
                    │                    │
             Generate Schedules    Save Rejection
                    │                    │
             Notification Service  Notification Service
             (Email + SMS)         (Email)
```

---

## 5. Screen Behavior

### Login Screen
- Tombol **Masuk** aktif hanya jika email/HP dan password terisi valid
- Tombol **Biometrik** hanya tampil jika:
  - Device mendukung Face ID / Fingerprint
  - User sudah mendaftarkan biometric sebelumnya
- Error kredensial salah: inline error merah di bawah field password
- Max **5x** percobaan gagal → akun terkunci 15 menit
- Sukses → JWT disimpan di Secure Storage (Keychain/Keystore)

### Registrasi Screen
- **Progress bar 4 langkah**: Data Diri → Dokumen → OTP → Password
- Upload KTP: validasi format JPG/PNG, ukuran maks 5MB, preview thumbnail
- Selfie: in-app camera dengan **liveness detection** (cegah spoofing)
- OTP 6 digit, berlaku **5 menit**, resend setelah **60 detik**

### Dashboard Screen
- Jika tidak ada pinjaman aktif: CTA "Ajukan Pinjaman" ditonjolkan
- Saldo sisa hutang diambil dari active loan via `GET /loans`
- Badge tagihan berwarna **merah** jika ≤3 hari sebelum due date atau sudah lewat
- Push notification deep-link langsung ke screen relevan

### Form Pinjaman Screen
- Slider nominal: **Rp 1.000.000 – Rp 12.000.000** (step Rp 500.000)
- Pilihan tenor: **3, 6, 9, 12 bulan**
- Simulasi cicilan & total bunga kalkulasi **real-time**
- Tombol "Ajukan" **disabled** jika ada pinjaman aktif/pending:
  > *"Selesaikan pinjaman aktif terlebih dahulu"*
- Sebelum submit: **bottom sheet** konfirmasi + checkbox syarat & ketentuan

### Detail Tagihan & Pembayaran Screen
- Status cicilan dengan kode warna:
  - 🟢 **Hijau** = Lunas
  - 🟡 **Kuning** = Akan jatuh tempo (≤7 hari)
  - 🔴 **Merah** = Overdue
  - ⚪ **Abu** = Mendatang
- Metode pembayaran: **Transfer Bank**, **Virtual Account**, **GoPay**, **OVO**
- Setelah sukses: status cicilan update otomatis + notifikasi push + email
- Pelunasan lebih awal: field jumlah bayar **editable**, sisa hutang auto-kalkulasi

---

## Tech Stack

| Kategori | Teknologi |
|----------|-----------|
| Mobile | React Native / Flutter |
| Backend | Node.js (Express) / Go (Gin) |
| Scoring | Python (FastAPI) |
| Database | PostgreSQL 15 |
| Cache | Redis 7 |
| Storage | AWS S3 |
| Message Queue | RabbitMQ / Apache Kafka |
| API Gateway | Kong / AWS API Gateway |
| Notification | AWS SES (Email), Twilio (SMS) |
| Auth | JWT (access + refresh token) |
| KYC | Verihubs / Vida |
| CI/CD | GitHub Actions |
| Container | Docker + Kubernetes |

---

## 📁 File Structure

```
ptxyz-loan-app/
├── README.md
├── docs/
│   ├── sequence-diagram.puml    ← API sequence diagram (PlantUML)
│   └── erd.puml                 ← Entity relationship diagram (PlantUML)
└── api/
    └── api-spec.md              ← Detail API endpoint specification
```

---

