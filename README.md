# Database Design Recommendation - Laundry POS System

## Overview

Database ini dirancang untuk aplikasi Laundry POS yang mendukung:

- Multi Cabang
- Multi Kasir
- Membership dengan Saldo
- Tier Membership
- Masa Berlaku Membership
- Promo / Voucher
- Pembelian Membership
- Tracking Status Laundry
- Dashboard Pendapatan
- Audit Transaksi

---

# Entity Relationship Diagram (Concept)

```
Admin

Cabang
│
├── Kasir
│
├── Order
│     ├── Layanan
│     ├── Membership
│     ├── Promo
│     └── Order Logs
│
└── Membership Transaction

Membership
│
├── Tier
├── Order
└── Membership Transaction
```

---

# 1. Admin

Digunakan untuk login Super Admin.

| Field | Type | Keterangan |
|---------|------|------------|
| id | BIGINT | PK |
| username | VARCHAR(100) | UNIQUE |
| password | VARCHAR(255) | Password Hash |
| created_at | TIMESTAMP | |
| updated_at | TIMESTAMP | |

---

# 2. Cabang

Master data cabang laundry.

| Field | Type |
|---------|------|
| id | BIGINT |
| nama | VARCHAR(150) |
| alamat | TEXT |
| no_hp | VARCHAR(20) |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

## Catatan

Tidak perlu menyimpan kolom:

- pendapatan

Pendapatan dihitung langsung dari transaksi order.

---

# 3. Kasir

Setiap kasir hanya berada pada satu cabang.

| Field | Type |
|---------|------|
| id | BIGINT |
| id_cabang | FK |
| nama | VARCHAR(150) |
| username | VARCHAR(100) |
| password | VARCHAR(255) |
| login_status | BOOLEAN |
| last_login | TIMESTAMP |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Relasi

```
Cabang
1 ---- n Kasir
```

---

# 4. Layanan

Master layanan laundry.

| Field | Type |
|---------|------|
| id | BIGINT |
| nama | VARCHAR(150) |
| harga | DECIMAL(12,2) |
| estimasi_hari | INT |
| deskripsi | TEXT |
| is_active | BOOLEAN |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Contoh

| Nama | Harga |
|------|-------:|
| Reguler | 7000/kg |
| Express | 12000/kg |
| Setrika | 5000/kg |

---

# 5. Tier Membership

Tier merupakan paket membership yang dijual.

Kasir **tidak dapat mengisi saldo secara manual**, melainkan hanya memilih paket Tier.

| Field | Type |
|---------|------|
| id | BIGINT |
| nama | VARCHAR(100) |
| harga | DECIMAL(12,2) |
| saldo_didapat | DECIMAL(12,2) |
| masa_berlaku_hari | INT |
| keterangan | TEXT |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Contoh

| Tier | Harga | Saldo Masuk | Masa Berlaku |
|------|-------:|------------:|-------------:|
| Silver | 50000 | 55000 | 30 Hari |
| Gold | 100000 | 120000 | 90 Hari |
| Platinum | 100000 | 150000 | 180 Hari |

---

# 6. Membership

Data pelanggan yang memiliki membership.

| Field | Type |
|---------|------|
| id | BIGINT |
| id_tier | FK |
| nama | VARCHAR(150) |
| no_hp | VARCHAR(20) UNIQUE|
| alamat | TEXT |
| saldo | DECIMAL(12,2) |
| berlaku_sampai | DATE |
| status | ENUM('ACTIVE','EXPIRED','BLOCKED') |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

## Aturan

Membership hanya memiliki satu Tier aktif.

Jika customer membeli Tier baru:

- id_tier diperbarui
- saldo bertambah sesuai saldo_didapat
- masa berlaku diperbarui
- transaksi dicatat pada Membership Transaction

---

# 7. Membership Transaction

Riwayat pembelian membership.

Setiap pembelian Tier akan menghasilkan satu record.

| Field | Type |
|---------|------|
| id | BIGINT |
| id_membership | FK |
| id_cabang | FK |
| id_kasir | FK |
| id_tier_lama | FK NULL |
| id_tier_baru | FK |
| harga | DECIMAL(12,2) |
| saldo_masuk | DECIMAL(12,2) |
| berlaku_sebelum | DATE |
| berlaku_setelah | DATE |
| metode_bayar | ENUM('Cash','Transfer','QRIS') |
| created_at | TIMESTAMP |

---

## Contoh

Customer

```
Tier

Gold

Saldo

20000

Expired

2026-08-20
```

Membeli

```
Platinum
```

Maka sistem otomatis

```
Tier

Gold

↓

Platinum

Saldo

20000

+

150000

=

170000

Expired

2026-08-20

↓

2027-02-16
```

Kasir tidak memasukkan nominal saldo secara manual.

---

# 8. Promo

Master promo.

| Field | Type |
|---------|------|
| id | BIGINT |
| kode | VARCHAR(50) UNIQUE |
| nama | VARCHAR(150) |
| tipe_diskon | ENUM('PERCENT','NOMINAL') |
| nilai_diskon | DECIMAL(12,2) |
| minimal_transaksi | DECIMAL(12,2) |
| maksimal_diskon | DECIMAL(12,2) |
| tanggal_mulai | DATE |
| tanggal_selesai | DATE |
| is_active | BOOLEAN |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Contoh

| Promo | Diskon |
|--------|--------|
| NEW10 | 10% |
| HEMAT20 | Rp20.000 |
| MEMBER5 | 5% |

---

# 9. Order

Data transaksi laundry.

| Field | Type |
|---------|------|
| id | BIGINT |
| kode_invoice | VARCHAR(100) UNIQUE |
| id_cabang | FK |
| id_kasir | FK |
| id_layanan | FK |
| id_membership | FK NULL |
| id_promo | FK NULL |
| nama_customer | VARCHAR(150) |
| no_hp | VARCHAR(20) |
| alamat | TEXT |
| berat | DECIMAL(8,2) |
| harga_per_kg | DECIMAL(12,2) |
| subtotal | DECIMAL(12,2) |
| diskon | DECIMAL(12,2) |
| total | DECIMAL(12,2) |
| metode_bayar | ENUM('Cash','Transfer','QRIS','Membership') |
| status | ENUM('Waiting','Processing','Washing','Drying','Ironing','Ready Pickup','Completed','Cancelled') |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |
| selesai_at | TIMESTAMP NULL |

## Kenapa menyimpan harga_per_kg?

Karena harga layanan dapat berubah sewaktu-waktu.

History transaksi tetap valid.

---

# 10. Order Logs

Mencatat seluruh perubahan status laundry.

| Field | Type |
|---------|------|
| id | BIGINT |
| id_order | FK |
| status | ENUM |
| keterangan | TEXT |
| changed_by | FK Kasir |
| created_at | TIMESTAMP |

Contoh

```
Waiting

↓

Processing

↓

Washing

↓

Drying

↓

Ironing

↓

Ready Pickup

↓

Completed
```

---

# Relasi Database

```
Cabang

1 ---- n Kasir

1 ---- n Order

1 ---- n MembershipTransaction
```

```
Kasir

1 ---- n Order

1 ---- n MembershipTransaction

1 ---- n OrderLogs
```

```
Tier

1 ---- n Membership

1 ---- n MembershipTransaction
```

```
Membership

1 ---- n Order

1 ---- n MembershipTransaction
```

```
Promo

1 ---- n Order
```

```
Layanan

1 ---- n Order
```

```
Order

1 ---- n OrderLogs
```

---

# Flow Pembelian Membership

```
Kasir

↓

Cari Membership

↓

Pilih Tier

↓

Customer Bayar

↓

Sistem mengambil data Tier

↓

Saldo Membership += saldo_didapat

↓

Tier Membership diganti

↓

Perpanjang masa berlaku

↓

Simpan Membership Transaction

↓

Selesai
```

Kasir **tidak dapat memasukkan nominal saldo secara manual.**

---

# Flow Order Laundry

```
Kasir

↓

Pilih Customer

↓

Pilih Layanan

↓

Input Berat

↓

Pilih Promo (Opsional)

↓

Pilih Membership (Opsional)

↓

Hitung Total

↓

Pilih Metode Bayar

↓

Simpan Order

↓

Cetak Invoice
```

---

# Dashboard Admin

## Pendapatan Hari Ini

```
SUM(order.total)
WHERE status='Completed'
```

---

## Pendapatan Bulanan

```
SUM(order.total)
GROUP BY YEAR(created_at), MONTH(created_at)
```

---

## Pendapatan per Cabang

```
SUM(order.total)
GROUP BY id_cabang
```

---

## Pendapatan per Kasir

```
SUM(order.total)
GROUP BY id_kasir
```

---

## Total Order

```
COUNT(order.id)
```

---

## Total Member

```
COUNT(membership.id)
```

---

## Membership Aktif

```
COUNT(status='ACTIVE')
```

---

## Membership Expired

```
COUNT(status='EXPIRED')
```

---

## Pendapatan Penjualan Membership

```
SUM(membership_transaction.harga)
```

---

## Tier Terlaris

```
COUNT(id_tier_baru)
GROUP BY id_tier_baru
```

---

## Promo Terbanyak Digunakan

```
COUNT(id_promo)
GROUP BY id_promo
```

---

# Index yang Disarankan

## Order

- id_cabang
- id_kasir
- id_membership
- id_promo
- created_at
- status

## Membership

- kode_member
- no_hp
- berlaku_sampai

## Membership Transaction

- id_membership
- id_tier_baru
- created_at

## Promo

- kode

## Order Logs

- id_order
- created_at

---

# Kelebihan Desain

- ✅ Fully Normalized (3NF)
- ✅ Multi Cabang
- ✅ Multi Kasir
- ✅ Membership dengan masa berlaku
- ✅ Saldo Membership otomatis sesuai Tier
- ✅ Riwayat perubahan Tier (upgrade/downgrade)
- ✅ Promo dan voucher
- ✅ Tracking status laundry
- ✅ Harga transaksi historis tetap valid
- ✅ Dashboard pendapatan lengkap
- ✅ Mudah dikembangkan untuk fitur paket laundry, refund, poin loyalitas, QRIS, dan analitik
- ✅ Audit transaksi yang lengkap melalui `Membership Transaction` dan `Order Logs`
