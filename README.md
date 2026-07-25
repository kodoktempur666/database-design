# Database Design Recommendation - Laundry POS System

## Overview

This database is designed for a Laundry POS application with support for:

- Multi Branch
- Multi Cashier
- Membership Balance
- Membership Tier
- Membership Expiration
- Promotion / Voucher
- Membership Purchase
- Laundry Order Tracking
- Revenue Dashboard
- Transaction Audit

---

# Entity Relationship Diagram (Concept)

```text
Admins

Branches
│
├── Cashiers
│
├── Orders
│     ├── Services
│     ├── Memberships
│     ├── Promotions
│     └── Order Logs
│
└── Membership Transactions

Memberships
│
├── Membership Tiers
├── Orders
└── Membership Transactions
```

---

# 1. Admins

Used for Super Admin authentication.

| Column | Type | Description |
|---------|------|-------------|
| id | BIGINT | Primary Key |
| username | VARCHAR(100) | Unique username |
| password | VARCHAR(255) | Password hash |
| created_at | TIMESTAMP | Created timestamp |
| updated_at | TIMESTAMP | Updated timestamp |

---

# 2. Branches

Stores all laundry branch information.

| Column | Type |
|---------|------|
| id | BIGINT |
| name | VARCHAR(150) |
| address | TEXT |
| phone_number | VARCHAR(20) |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

## Notes

Do **not** store revenue in this table.

Revenue should always be calculated from completed orders.

---

# 3. Cashiers

Each cashier belongs to one branch.

| Column | Type |
|---------|------|
| id | BIGINT |
| branch_id | FK → branches.id |
| name | VARCHAR(150) |
| username | VARCHAR(100) |
| password | VARCHAR(255) |
| is_logged_in | BOOLEAN |
| last_login_at | TIMESTAMP |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |

Relationship

```text
Branch
1 ----- n Cashiers
```

---

# 4. Services

Master data for laundry services.

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

# 5. Membership Tiers

Represents membership packages sold by the cashier.

The cashier **cannot manually enter balance values**.

The cashier only selects a membership package.

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

# 6. Memberships

Stores member information.

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

Each membership only has one active tier.

When a customer purchases another membership package:

- Update `tier_id`
- Add balance based on the selected tier
- Extend membership validity
- Save transaction history in `membership_transactions`

---

# 7. Membership Transactions

Stores all membership purchase history.

Each membership purchase creates one record.

| Column | Type |
|---------|------|
| id | BIGINT |
| membership_id | FK → memberships.id |
| branch_id | FK → branches.id |
| cashier_id | FK → cashiers.id |
| previous_tier_id | FK → membership_tiers.id (Nullable) |
| current_tier_id | FK → membership_tiers.id |
| purchase_price | DECIMAL(12,2) |
| balance_added | DECIMAL(12,2) |
| previous_expiry_date | DATE |
| new_expiry_date | DATE |
| payment_method | ENUM('CASH','TRANSFER','QRIS') |
| created_at | TIMESTAMP |

---

## Example

Current Membership

```text
Tier        : Gold
Balance     : 20000
Expires At  : 2026-08-20
```

Customer purchases

```text
Platinum
```

System automatically updates

```text
Tier

Gold
↓

Platinum

Balance

20000
+
150000
=
170000

Expiration

2026-08-20
↓

2027-02-16
```

---

# 8. Promotions

Stores promotional campaigns.

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

# 9. Orders

Stores laundry transactions.

| Column | Type |
|---------|------|
| id | BIGINT |
| invoice_number | VARCHAR(100) UNIQUE |
| branch_id | FK → branches.id |
| cashier_id | FK → cashiers.id |
| membership_id | FK → memberships.id (Nullable) |
| promotion_id | FK → promotions.id (Nullable) |
| customer_name | VARCHAR(150) |
| phone_number | VARCHAR(20) |
| address | TEXT |
| payment_method | ENUM('CASH','TRANSFER','QRIS','MEMBERSHIP') |
| subtotal | DECIMAL(12,2) |
| discount_amount | DECIMAL(12,2) |
| total_amount | DECIMAL(12,2) |
| status | ENUM('WAITING','PROCESSING','WASHING','DRYING','IRONING','READY_FOR_PICKUP','COMPLETED','CANCELLED') |
| created_at | TIMESTAMP |
| updated_at | TIMESTAMP |
| completed_at | TIMESTAMP NULL |

---

# 10. Order Items

Stores services included in an order.

This design allows one order to contain multiple laundry services.

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

# 11. Order Logs

Tracks every status change.

| Column | Type |
|---------|------|
| id | BIGINT |
| order_id | FK → orders.id |
| status | ENUM |
| notes | TEXT |
| updated_by | FK → cashiers.id |
| created_at | TIMESTAMP |

Example

```text
WAITING

↓

PROCESSING

↓

WASHING

↓

DRYING

↓

IRONING

↓

READY_FOR_PICKUP

↓

COMPLETED
```

---

# Relationships

```text
Branches

1 ----- n Cashiers
1 ----- n Orders
1 ----- n Membership Transactions
```

```text
Cashiers

1 ----- n Orders
1 ----- n Membership Transactions
1 ----- n Order Logs
```

```text
Membership Tiers

1 ----- n Memberships
1 ----- n Membership Transactions
```

```text
Memberships

1 ----- n Orders
1 ----- n Membership Transactions
```

```text
Promotions

1 ----- n Orders
```

```text
Orders

1 ----- n Order Items
1 ----- n Order Logs
```

```text
Services

1 ----- n Order Items
```

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
SUM(total_amount)
WHERE status = 'COMPLETED'
```

---

## Monthly Revenue

```sql
SUM(total_amount)
GROUP BY YEAR(created_at), MONTH(created_at)
```

---

## Revenue by Branch

```sql
SUM(total_amount)
GROUP BY branch_id
```

---

## Revenue by Cashier

```sql
SUM(total_amount)
GROUP BY cashier_id
```

---

## Total Orders

```sql
COUNT(id)
```

---

## Total Members

```sql
COUNT(id)
FROM memberships
```

---

## Active Memberships

```sql
COUNT(*)
WHERE status='ACTIVE'
```

---

## Expired Memberships

```sql
COUNT(*)
WHERE status='EXPIRED'
```

---

## Membership Sales

```sql
SUM(purchase_price)
FROM membership_transactions
```

---

## Best Selling Membership Tier

```sql
COUNT(current_tier_id)
GROUP BY current_tier_id
```

---

## Most Used Promotion

```sql
COUNT(promotion_id)
GROUP BY promotion_id
```

---

# Recommended Indexes

## Orders

- invoice_number
- branch_id
- cashier_id
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

## Promotions

- code
- is_active

## Order Logs

- order_id
- created_at

---

# Advantages

- ✅ Third Normal Form (3NF)
- ✅ Multi-Branch Support
- ✅ Multi-Cashier Support
- ✅ Membership Balance Management
- ✅ Membership Expiration
- ✅ Membership Upgrade/Downgrade History
- ✅ Flexible Promotions
- ✅ Multiple Services per Order
- ✅ Historical Pricing
- ✅ Complete Revenue Dashboard
- ✅ Full Transaction Audit
- ✅ Easily extensible for refunds, loyalty points, delivery services, and advanced reporting
