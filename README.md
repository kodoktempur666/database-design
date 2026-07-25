# Database Design Recommendation - Laundry POS System (Revised)

## Overview

Database ini dirancang untuk aplikasi Laundry POS dengan dukungan:

- Multi Branch
- Multi Staff / Multi Cashier
- Staff Attendance (QR Code based)
- Membership Balance
- Membership Tier
- Membership Expiration
- Promotion / Voucher
- Membership Purchase
- Laundry Order Tracking
- Revenue Dashboard
- Transaction Audit (termasuk audit penggunaan saldo membership)

---

## Ringkasan Perbaikan dari Draft Sebelumnya

Berikut perbaikan utama yang dilakukan pada draft ini:

1. **Penomoran entitas dirapikan** — sebelumnya nomor tabel loncat dan duplikat (dua entitas bernomor "4", "5", "6", dst).
2. **`cashiers.id` dihapus** — tabel `cashiers` tidak pernah didefinisikan di draft awal. Semua kasir sebenarnya adalah baris di tabel `staffs` dengan `role_id` mengarah ke role "Cashier". Semua FK yang sebelumnya menunjuk `cashiers.id` (di `orders`, `membership_transactions`, `order_logs`) sekarang diarahkan ke `staffs.id`.
3. **Bagian "Updated Relationships" dan "Relationships" yang saling tumpang tindih/kontradiktif digabung** menjadi satu bagian relasi tunggal di akhir dokumen agar tidak ada informasi yang bertentangan.
4. **Ditambahkan `membership_amount_used` pada tabel `orders`** — draft awal punya `payment_method = 'MEMBERSHIP'` tapi tidak pernah mencatat berapa saldo yang terpakai, sehingga saldo membership tidak bisa direkonsiliasi.
5. **Ditambahkan tabel baru `membership_balance_logs`** — draft awal hanya mencatat penambahan saldo (`membership_transactions`), tapi tidak mencatat pengurangan saldo saat dipakai belanja. Tanpa ini, klaim "Full Transaction Audit" tidak lengkap.
6. **Query dashboard "Revenue by Cashier" diperjelas** dengan join eksplisit ke `staff_roles`.
7. **Konsistensi FK dan penamaan kolom** dirapikan di seluruh tabel.

---

# Entity Relationship Diagram (Concept)

```text
Admins

Branches
│
├── Staff Roles
│     └── Staffs
│           ├── Attendance QR Codes → Staff Attendances
│           ├── Orders
│           ├── Membership Transactions
│           └── Order Logs
│
├── Orders
│     ├── Order Items → Services
│     ├── Memberships
│     ├── Promotions
│     └── Order Logs
│
└── Membership Transactions

Memberships
│
├── Membership Tiers
├── Orders
├── Membership Transactions
└── Membership Balance Logs
```

---

# 1. Admins

Digunakan untuk autentikasi Super Admin.

| Column | Type | Description |
|---------|------|-------------|
| id | BIGINT | Primary Key |
| username | VARCHAR(100) | Unique username |
| password | VARCHAR(255) | Password hash |
| created_at | TIMESTAMP | Created timestamp |
| updated_at | TIMESTAMP | Updated timestamp |

---

# 2. Branches

Menyimpan informasi seluruh cabang laundry.

| Column | Type |
|---------|------|
| id | BIGINT |
| name | VARCHAR(150) |
| address | TEXT |
| phone_number | VARCHAR(20) |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

## Notes

**Tidak** menyimpan revenue di tabel ini.

Revenue selalu dihitung dari order yang sudah `COMPLETED`.

---

# 3. Staff Roles

Menyimpan daftar role staff yang tersedia.

| Column | Type |
|---------|------|
| id | BIGINT |
| name | VARCHAR(100) |
| description | TEXT |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Example

| id | Name |
|----|------|
| 1 | Super Admin (khusus tabel `admins`, bukan bagian `staffs`) |
| 2 | Branch Manager |
| 3 | Cashier |
| 4 | Laundry Staff |
| 5 | Courier |

Relationship

```text
Staff Roles
1 ----- n Staffs
```

---

# 4. Staffs

Menyimpan seluruh pegawai. Setiap staff dimiliki oleh satu cabang dan memiliki satu role.

Kasir bukan tabel terpisah — kasir adalah `staff` dengan `role_id` = role "Cashier".

| Column | Type |
|---------|------|
| id | BIGINT |
| branch_id | FK → branches.id |
| role_id | FK → staff_roles.id |
| full_name | VARCHAR(150) |
| username | VARCHAR(100) UNIQUE |
| password | VARCHAR(255) |
| phone_number | VARCHAR(20) |
| address | TEXT |
| qr_code | VARCHAR(255) NULL |
| is_logged_in | BOOLEAN |
| last_login_at | TIMESTAMP NULL |
| is_active | BOOLEAN |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Relationship

```text
Branches
1 ----- n Staffs

Staff Roles
1 ----- n Staffs
```

---

# 5. Attendance QR Codes

Menyimpan QR code yang dipakai untuk presensi. QR code bisa berganti setiap hari atau setiap sesi untuk mencegah penyalahgunaan.

| Column | Type |
|---------|------|
| id | BIGINT |
| branch_id | FK → branches.id |
| qr_token | VARCHAR(255) UNIQUE |
| valid_from | TIMESTAMP |
| valid_until | TIMESTAMP |
| created_at | TIMESTAMP |

Example

```text
QR Code
↓
Valid Today 08:00 - 17:00
↓
Expired
```

---

# 6. Staff Attendances

Menyimpan riwayat presensi. Presensi tercatat saat staff memindai QR code yang masih valid.

| Column | Type |
|---------|------|
| id | BIGINT |
| staff_id | FK → staffs.id |
| branch_id | FK → branches.id |
| attendance_qr_code_id | FK → attendance_qr_codes.id |
| attendance_type | ENUM('CHECK_IN','CHECK_OUT') |
| scanned_at | TIMESTAMP |
| latitude | DECIMAL(10,7) NULL |
| longitude | DECIMAL(10,7) NULL |
| notes | TEXT |
| created_at | TIMESTAMP |

Relationship

```text
Staffs
1 ----- n Staff Attendances

Attendance QR Codes
1 ----- n Staff Attendances
```

---

# 7. Services

Master data untuk layanan laundry.

| Column | Type |
|---------|------|
| id | BIGINT |
| name | VARCHAR(150) |
| price | DECIMAL(12,2) |
| type | VARCHAR(150) |
| estimated_hours | INT |
| description | TEXT |
| is_active | BOOLEAN |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Example

| Service | Price |
|----------|------:|
| Regular Laundry | 7000/kg |
| Express Laundry | 12000/kg |
| Ironing | 5000/kg |

---

# 8. Membership Tiers

Merepresentasikan paket membership yang dijual oleh kasir.

Kasir **tidak bisa** menginput nilai saldo secara manual — kasir hanya memilih paket membership.

| Column | Type |
|---------|------|
| id | BIGINT |
| name | VARCHAR(100) |
| purchase_price | DECIMAL(12,2) |
| balance_amount | DECIMAL(12,2) |
| validity_days | INT |
| description | TEXT |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Example

| Tier | Purchase Price | Balance Added | Validity |
|------|---------------:|--------------:|----------:|
| Silver | 50000 | 55000 | 30 Days |
| Gold | 100000 | 120000 | 90 Days |
| Platinum | 100000 | 150000 | 180 Days |

---

# 9. Memberships

Menyimpan informasi member.

| Column | Type |
|---------|------|
| id | BIGINT |
| tier_id | FK → membership_tiers.id |
| customer_name | VARCHAR(150) |
| phone_number | VARCHAR(20) UNIQUE |
| address | TEXT |
| balance | DECIMAL(12,2) |
| expires_at | DATE |
| status | ENUM('ACTIVE','EXPIRED','BLOCKED') |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

## Business Rules

Setiap membership hanya punya satu tier aktif.

Saat customer membeli paket membership baru:

- Update `tier_id`
- Tambahkan saldo sesuai tier yang dipilih
- Perpanjang masa berlaku
- Simpan riwayat transaksi di `membership_transactions`

Saat saldo membership dipakai untuk membayar order:

- Kurangi `balance` pada tabel ini
- Simpan riwayat pengurangan di `membership_balance_logs` (lihat entitas #14)

---

# 10. Membership Transactions

Menyimpan seluruh riwayat **pembelian/penambahan** membership. Setiap pembelian membership membuat satu record.

| Column | Type |
|---------|------|
| id | BIGINT |
| membership_id | FK → memberships.id |
| branch_id | FK → branches.id |
| staff_id | FK → staffs.id |
| previous_tier_id | FK → membership_tiers.id (Nullable) |
| current_tier_id | FK → membership_tiers.id |
| purchase_price | DECIMAL(12,2) |
| balance_added | DECIMAL(12,2) |
| previous_expiry_date | DATE |
| new_expiry_date | DATE |
| payment_method | ENUM('CASH','TRANSFER','QRIS') |
| created_at | TIMESTAMP |

## Example

Current Membership

```text
Tier        : Gold
Balance     : 20000
Expires At  : 2026-08-20
```

Customer membeli

```text
Platinum
```

Sistem otomatis update

```text
Tier
Gold → Platinum

Balance
20000 + 150000 = 170000

Expiration
2026-08-20 → 2027-02-16
```

---

# 11. Promotions

Menyimpan kampanye promosi.

| Column | Type |
|---------|------|
| id | BIGINT |
| code | VARCHAR(50) UNIQUE |
| name | VARCHAR(150) |
| discount_type | ENUM('PERCENTAGE','FIXED_AMOUNT') |
| discount_value | DECIMAL(12,2) |
| minimum_purchase | DECIMAL(12,2) |
| maximum_discount | DECIMAL(12,2) |
| start_date | DATE |
| end_date | DATE |
| is_active | BOOLEAN |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Example

| Promotion | Discount |
|-----------|----------|
| NEW10 | 10% |
| SAVE20 | 20000 |
| MEMBER5 | 5% |

---

# 12. Orders

Menyimpan transaksi laundry.

| Column | Type |
|---------|------|
| id | BIGINT |
| invoice_number | VARCHAR(100) UNIQUE |
| branch_id | FK → branches.id |
| staff_id | FK → staffs.id |
| membership_id | FK → memberships.id (Nullable) |
| promotion_id | FK → promotions.id (Nullable) |
| customer_name | VARCHAR(150) |
| phone_number | VARCHAR(20) |
| address | TEXT |
| payment_method | ENUM('CASH','TRANSFER','QRIS','MEMBERSHIP') |
| subtotal | DECIMAL(12,2) |
| discount_amount | DECIMAL(12,2) |
| membership_amount_used | DECIMAL(12,2) DEFAULT 0 |
| total_amount | DECIMAL(12,2) |
| status | ENUM('WAITING','PROCESSING','WASHING','DRYING','IRONING','READY_FOR_PICKUP','COMPLETED','CANCELLED') |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |
| completed_at | TIMESTAMP NULL |

**Catatan perbaikan:** kolom `membership_amount_used` ditambahkan karena `payment_method` bisa bernilai `MEMBERSHIP`, tapi draft sebelumnya tidak mencatat berapa nominal saldo yang benar-benar terpakai — sehingga saldo di tabel `memberships` tidak bisa direkonsiliasi terhadap order.

---

# 13. Order Items

Menyimpan layanan yang termasuk dalam satu order. Desain ini memungkinkan satu order berisi banyak layanan laundry.

| Column | Type |
|---------|------|
| id | BIGINT |
| order_id | FK → orders.id |
| service_id | FK → services.id |
| quantity | DECIMAL(8,2) |
| unit_price | DECIMAL(12,2) |
| subtotal | DECIMAL(12,2) |
| created_at | TIMESTAMP |

Example

| Service | Quantity | Unit Price |
|----------|---------:|-----------:|
| Regular Laundry | 5 kg | 7000 |
| Ironing | 5 kg | 5000 |

---

# 14. Order Logs

Melacak setiap perubahan status order.

| Column | Type |
|---------|------|
| id | BIGINT |
| order_id | FK → orders.id |
| status | ENUM |
| notes | TEXT |
| updated_by | FK → staffs.id |
| created_at | TIMESTAMP |

Example

```text
WAITING → PROCESSING → WASHING → DRYING → IRONING → READY_FOR_PICKUP → COMPLETED
```

---

# 15. Membership Balance Logs (Tabel Baru)

Mencatat setiap **pengurangan/penggunaan** saldo membership, terpisah dari `membership_transactions` yang hanya mencatat penambahan saldo. Tabel ini melengkapi audit trail agar saldo membership sepenuhnya bisa direkonsiliasi (penambahan maupun pemakaian).

| Column | Type |
|---------|------|
| id | BIGINT |
| membership_id | FK → memberships.id |
| order_id | FK → orders.id (Nullable, untuk kasus non-order seperti koreksi manual admin) |
| branch_id | FK → branches.id |
| staff_id | FK → staffs.id |
| type | ENUM('USAGE','ADJUSTMENT','REFUND') |
| amount | DECIMAL(12,2) |
| balance_before | DECIMAL(12,2) |
| balance_after | DECIMAL(12,2) |
| notes | TEXT |
| created_at | TIMESTAMP |

---

# Membership Purchase Flow

```text
Cashier
↓
Find Membership
↓
Select Membership Tier
↓
Customer Pays
↓
System Loads Tier Information
↓
Update Membership Balance
↓
Update Membership Tier
↓
Extend Expiration Date
↓
Create Membership Transaction
↓
Done
```

---

# Laundry Order Flow

```text
Cashier
↓
Select Customer
↓
Create Order
↓
Add One or More Services
↓
Apply Promotion (Optional)
↓
Use Membership Balance (Optional)
↓
    If Membership Balance Used:
    - Deduct memberships.balance
    - Log to membership_balance_logs (type = USAGE)
    - Set orders.membership_amount_used
↓
Calculate Total
↓
Select Payment Method
↓
Save Order
↓
Print Receipt
```

---

# Dashboard

## Daily Revenue

```sql
SELECT SUM(total_amount)
FROM orders
WHERE status = 'COMPLETED'
  AND DATE(completed_at) = CURDATE();
```

## Monthly Revenue

```sql
SELECT YEAR(completed_at) AS year, MONTH(completed_at) AS month, SUM(total_amount)
FROM orders
WHERE status = 'COMPLETED'
GROUP BY YEAR(completed_at), MONTH(completed_at);
```

## Revenue by Branch

```sql
SELECT branch_id, SUM(total_amount)
FROM orders
WHERE status = 'COMPLETED'
GROUP BY branch_id;
```

## Revenue by Cashier

```sql
SELECT o.staff_id, s.full_name, SUM(o.total_amount)
FROM orders o
JOIN staffs s ON s.id = o.staff_id
JOIN staff_roles r ON r.id = s.role_id
WHERE o.status = 'COMPLETED'
  AND r.name = 'Cashier'
GROUP BY o.staff_id, s.full_name;
```

## Total Orders

```sql
SELECT COUNT(id) FROM orders;
```

## Total Members

```sql
SELECT COUNT(id) FROM memberships;
```

## Active Memberships

```sql
SELECT COUNT(*) FROM memberships WHERE status = 'ACTIVE';
```

## Expired Memberships

```sql
SELECT COUNT(*) FROM memberships WHERE status = 'EXPIRED';
```

## Membership Sales

```sql
SELECT SUM(purchase_price) FROM membership_transactions;
```

## Best Selling Membership Tier

```sql
SELECT current_tier_id, COUNT(*) AS total
FROM membership_transactions
GROUP BY current_tier_id
ORDER BY total DESC;
```

## Most Used Promotion

```sql
SELECT promotion_id, COUNT(*) AS total
FROM orders
WHERE promotion_id IS NOT NULL
GROUP BY promotion_id
ORDER BY total DESC;
```

## Membership Balance Usage (Query Baru)

```sql
SELECT membership_id, SUM(amount) AS total_used
FROM membership_balance_logs
WHERE type = 'USAGE'
GROUP BY membership_id;
```

---

# Recommended Indexes

## Staffs
- branch_id
- role_id
- username

## Attendance QR Codes
- branch_id
- qr_token
- valid_from, valid_until

## Staff Attendances
- staff_id
- attendance_qr_code_id
- scanned_at

## Orders
- invoice_number
- branch_id
- staff_id
- membership_id
- promotion_id
- status
- created_at

## Order Items
- order_id
- service_id

## Memberships
- phone_number
- tier_id
- expires_at

## Membership Transactions
- membership_id
- current_tier_id
- created_at

## Membership Balance Logs
- membership_id
- order_id
- created_at

## Promotions
- code
- is_active

## Order Logs
- order_id
- created_at

---

# Final Relationships (Satu Sumber Kebenaran)

```text
Branches
1 ----- n Staffs
1 ----- n Attendance QR Codes
1 ----- n Orders
1 ----- n Membership Transactions
1 ----- n Membership Balance Logs

Staff Roles
1 ----- n Staffs

Staffs
1 ----- n Staff Attendances
1 ----- n Orders
1 ----- n Membership Transactions
1 ----- n Order Logs
1 ----- n Membership Balance Logs

Attendance QR Codes
1 ----- n Staff Attendances

Membership Tiers
1 ----- n Memberships
1 ----- n Membership Transactions

Memberships
1 ----- n Orders
1 ----- n Membership Transactions
1 ----- n Membership Balance Logs

Promotions
1 ----- n Orders

Orders
1 ----- n Order Items
1 ----- n Order Logs
1 ----- n Membership Balance Logs (via order_id, nullable)

Services
1 ----- n Order Items
```

---

# Advantages

- ✅ Third Normal Form (3NF)
- ✅ Multi-Branch Support
- ✅ Multi-Staff / Multi-Cashier Support (via `staffs` + `staff_roles`, tanpa tabel duplikat)
- ✅ Staff Attendance dengan QR Code
- ✅ Membership Balance Management (penambahan **dan** pengurangan tercatat)
- ✅ Membership Expiration
- ✅ Membership Upgrade/Downgrade History
- ✅ Flexible Promotions
- ✅ Multiple Services per Order
- ✅ Historical Pricing
- ✅ Complete Revenue Dashboard
- ✅ Full Transaction Audit (termasuk audit saldo membership)
- ✅ Tidak ada FK yang menunjuk ke tabel yang tidak terdefinisi
- ✅ Easily extensible untuk refund, loyalty points, delivery service, dan reporting lanjutan
