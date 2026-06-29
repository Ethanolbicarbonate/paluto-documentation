![[Untitled(2).png]]

```postgresql
// ==========================================
// ENUMS
// ==========================================
Enum affiliate_type {
  driver
  tour_guide
}

Enum commission_status {
  Pending
  Claimed
}

Enum branch_names {
  Bitoon
  Passi
}

Enum food_stamp_type {
  "Standard"
  "Unli Paluto"
  "Sugba Converted"
}


// ==========================================
// CENTRAL AUTHENTICATION
// ==========================================
Table users {
  id            int       [pk, increment]
  username      varchar   [unique, not null]
  password_hash varchar   [not null]
  email         varchar
  created_at    timestamp [default: `now()`]

  Note: 'Base table for all logins across all roles'
}


// ==========================================
// ROLES & PROFILES
// ==========================================
Table admins {
  id         int       [pk, increment]
  user_id    int       [unique, not null]  // FIXED: was missing not null
  full_name  varchar
  created_at timestamp [default: `now()`]

  Note: 'HQ / Executive users — full access to all branches'
}

Table affiliate_managers {
  id         int       [pk, increment]
  user_id    int       [unique, not null]  // FIXED: was missing not null
  branch_id  int       [not null]          // FIXED: was missing not null
  full_name  varchar
  phone      varchar
  created_at timestamp [default: `now()`]

  Note: 'Branch-level staff who onboard drivers and log transactions'
}

Table affiliates {
  id                int            [pk, increment]
  user_id           int            [unique, not null, note: 'Links to users table for mobile app login']  // FIXED: was missing not null
  manager_id        int            [note: 'The manager who onboarded them — nullable, set null on manager delete']
  branch_id         int            [not null, note: 'Their primary branch']                              // FIXED: was missing not null
  type              affiliate_type [not null]
  driver_id         varchar        [unique, note: 'Visual ID on physical card, e.g., D-0055']
  first_name        varchar
  last_name         varchar
  contact_number    varchar
  affiliation_group varchar        [note: 'e.g., TAXI, GRAB, MAXIM, VAN, BTODA']
  commission_rate   decimal        [note: 'Percentage or flat rate used to compute earnings']
  created_at        timestamp      [default: `now()`]

  Note: 'Drivers and Tour Guides'
}


// ==========================================
// LOGISTICS & TRANSACTIONS
// ==========================================
Table branches {
  id         int          [pk, increment]
  name       branch_names [not null]  // FIXED: was missing not null
  location   varchar
  created_at timestamp    [default: `now()`]
}

Table transactions {
  id                int             [pk, increment]
  affiliate_id      int             [not null, note: 'Links to affiliates.id — required']           // FIXED: was missing not null
  branch_id         int             [not null]                                                       // FIXED: was missing not null
  headcount         int             [not null, note: 'Total headcount — must be at least 1']        // FIXED: was missing not null
  total_revenue     decimal         [not null, note: 'Trip Amount/Total Bill — must be greater than 0']  // FIXED: was missing not null
  transaction_date  timestamp       [not null, note: 'When the customer actually visited']          // FIXED: was missing not null
  referral_success  boolean         [default: true, note: 'Always true — a logged transaction means the referral already succeeded']
  free_meal_claimed boolean         [default: false, note: 'Did the driver eat their free meal for THIS trip?']
  food_stamp        boolean         [default: false, note: 'Was a voucher given?']
  food_stamp_type   food_stamp_type [null, note: 'Filled ONLY if food_stamp is true']              // FIXED: was `enum`, now references food_stamp_type enum
  notes             text            [note: 'E.g., "First Referral", "Will claim meal next visit"']
  created_at        timestamp       [default: `now()`]
}

Table commission_wallet {
  id             int               [pk, increment]
  transaction_id int               [not null, note: 'FK to transactions — multiple wallet entries allowed for split commissions']  // FIXED: was missing not null
  amount         decimal           [not null, note: 'Calculated Earnings per Trip — must be zero or positive']                     // FIXED: was missing not null
  status         commission_status [default: 'Pending']
  payout_date    timestamp         [null, note: 'Populated when status changes to Claimed']
  created_at     timestamp         [default: `now()`]
}


// ==========================================
// RELATIONSHIPS (Foreign Keys)
// ==========================================

// Auth Links
Ref "admin access":       users.id < admins.user_id
Ref "manager access":     users.id < affiliate_managers.user_id
Ref "affiliate access":   users.id < affiliates.user_id

// Branch Assignments
Ref "manages branch":     branches.id < affiliate_managers.branch_id
Ref "belongs to branch":  branches.id < affiliates.branch_id
Ref "records at branch":  branches.id < transactions.branch_id

// Operational Links
Ref "onboards":           affiliate_managers.(id, branch_id) < affiliates.(manager_id, branch_id) [delete: set null]
Ref "creates":            affiliates.id < transactions.affiliate_id
Ref "funds":              transactions.id < commission_wallet.transaction_id
```

---

### 🗂️ Global Enums (Fixed Values)
Before the tables, the database uses three "Enums" (lists of allowed values) to keep data consistent and prevent typos:
*   **`affiliate_type`**: Can only be `driver` or `tour_guide`.
*   **`commission_status`**: Can only be `Pending` or `Claimed`.
*   **`branch_names`**: Can only be `Bitoon`, `Passi`, or `HQ`.

---

### 🔒 Central Authentication
#### **Table: `users`**
> **Purpose:** The base security table. Every person who logs into the app (Admin, Manager, or Affiliate) has exactly one row here.

| Prop                | Note (What it is & Where it connects)                                                                 |
| :------------------ | :---------------------------------------------------------------------------------------------------- |
| **`id`**            | Primary Key. Auto-increments. This is the master ID used to link profiles to their login credentials. |
| **`username`**      | Unique text. The username they use to log in.                                                         |
| **`password_hash`** | Text. The securely encrypted version of their password.                                               |
| **`email`**         | Text (Optional). User's email address.                                                                |
| **`created_at`**    | Timestamp. When the account was created. Defaults to exact time of creation.                          |

---

### 🧑‍💼 Roles & Profiles

#### **Table: `admins`**
> **Purpose:** Profile details for HQ / Executive users who have full access to all branches.

| Prop | Note (What it is & Where it connects) |
| :--- | :--- |
| **`id`** | Primary Key. Auto-increments. |
| **`user_id`** | **Foreign Key.** Connects to `users.id`. Links this admin profile to their login credentials. Must be unique. |
| **`full_name`** | Text. The admin's full name. |
| **`created_at`** | Timestamp. When the profile was created. |

#### **Table: `affiliate_managers`**
> **Purpose:** Profile details for Branch-level staff (the ones who log transactions and onboard drivers).

| Prop | Note (What it is & Where it connects) |
| :--- | :--- |
| **`id`** | Primary Key. Auto-increments. |
| **`user_id`** | **Foreign Key.** Connects to `users.id`. Links this manager to their login credentials. |
| **`branch_id`** | **Foreign Key.** Connects to `branches.id`. Restricts this manager's access so they only see and operate within their assigned branch. |
| **`full_name`** | Text. The manager's full name. |
| **`phone`** | Text. The manager's contact number. |
| **`created_at`** | Timestamp. When the profile was created. |

#### **Table: `affiliates`**
> **Purpose:** Profile details for the Drivers and Tour Guides.

| Prop | Note (What it is & Where it connects) |
| :--- | :--- |
| **`id`** | Primary Key. Auto-increments. The internal database ID. |
| **`user_id`** | **Foreign Key.** Connects to `users.id`. Allows the driver to log into the mobile app to check their earnings. |
| **`manager_id`** | **Foreign Key.** Connects to `affiliate_managers.id`. Tracks *who* originally registered/onboarded this driver. |
| **`branch_id`** | **Foreign Key.** Connects to `branches.id`. The driver's "home" base or primary branch. |
| **`type`** | Enum. Specifies if they are a `driver` or `tour_guide`. |
| **`driver_code`** | Unique Text. The visual ID printed on their physical ID card (e.g., `D-0055`). This is what managers type into the search bar. |
| **`first_name`** | Text. Driver's first name. |
| **`last_name`** | Text. Driver's last name. |
| **`contact_number`** | Text. Driver's phone number. |
| **`affiliation_group`** | Text. The transport group/agency they belong to (e.g., `TAXI`, `GRAB`, `MAXIM`, `BTODA`). Used heavily for generating data match reports. |
| **`commission_rate`** | Decimal. A stored percentage or flat rate. *(Note: Generally overridden by the global ₱50 per ₱1000 formula, but kept in DB for special edge cases).* |
| **`created_at`** | Timestamp. When the driver was registered. |

---

### 🏢 Logistics & Transactions

#### **Table: `branches`**
> **Purpose:** Stores the physical restaurant locations.

| Prop | Note (What it is & Where it connects) |
| :--- | :--- |
| **`id`** | Primary Key. Auto-increments. |
| **`name`** | Enum. The name of the branch (`Bitoon`, `Passi`, or `HQ`). |
| **`location`** | Text. The physical address or detailed location string. |
| **`created_at`** | Timestamp. When the branch was added to the system. |

#### **Table: `transactions`**
> **Purpose:** The core operational table. Every time a driver drops off customers, a row is created here.

| Prop | Note (What it is & Where it connects) |
| :--- | :--- |
| **`id`** | Primary Key. Auto-increments. |
| **`affiliate_id`** | **Foreign Key.** Connects to `affiliates.id`. Identifies *which* driver brought the customers. |
| **`branch_id`** | **Foreign Key.** Connects to `branches.id`. Identifies *where* the drop-off happened. |
| **`headcount`** | Integer (Number). Manually inputted by manager. How many customers/guests were in this specific trip? |
| **`total_revenue`** | Decimal. The "Trip Amount" or total food bill paid by the customers. |
| **`transaction_date`** | Timestamp. The actual date/time the customer visited (can be backdated if the manager forgot to log it yesterday). |
| **`referral_success`** | Boolean (True/False). Informational tag to note if it was a successful referral. *(Does not block meal/commission payouts).* |
| **`free_meal_claimed`** | Boolean (True/False). Did the driver eat their free meal *for this specific drop-off*? If False, it counts as a "Banked/Unclaimed" meal for the future. |
| **`food_stamp`** | Boolean (True/False). Was a physical paper voucher/stamp given to the driver? |
| **`food_stamp_type`** | Text (Nullable). Filled *only* if `food_stamp` is True. Indicates the type of voucher (e.g., "Standard", "Unli Paluto"). |
| **`notes`** | Text. Any extra context typed by the manager (e.g., "First Referral", "Driver was in a rush, will eat next time"). |
| **`created_at`** | Timestamp. When the manager actually typed this record into the computer. |

#### **Table: `commission_wallet`**
> **Purpose:** The financial ledger. Tracks the actual money owed and paid to the drivers.

| Prop                 | Note (What it is & Where it connects)                                                                                                   |
| :------------------- | :-------------------------------------------------------------------------------------------------------------------------------------- |
| **`id`**             | Primary Key. Auto-increments.                                                                                                           |
| **`transaction_id`** | **Foreign Key.** Connects to `transactions.id`. Links this money back to the specific trip/drop-off that generated it.                  |
| **`amount`**         | Decimal. The "Earnings per Trip". This is the calculated cash amount (e.g., ₱150.00) the driver earned from the transaction.            |
| **`status`**         | Enum (`Pending` or `Claimed`). If Pending, the restaurant owes the driver this money. If Claimed, the manager has handed them the cash. |
| **`payout_date`**    | Timestamp (Nullable). Starts blank. Automatically fills with the exact date/time the manager clicks "Mark as Claimed".                  |
| **`created_at`**     | Timestamp. When this earning record was created (usually the same exact time as the transaction).                                       |

---