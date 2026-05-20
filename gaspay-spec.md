# GasPay Platform: System Mechanism, Database ERD Deep Dive, & API Reference

Welcome to the **GasPay Developer Portal & Operational Reference**. This comprehensive document covers the technical architecture, operating workflows, database schema specifications, and full API specifications of the **GasPay** multi-tenant SaaS utility billing platform.

---

## 1. System Overview & Core Workflow

GasPay is designed as a secure, high-concurrency multi-tenant SaaS platform where gas distribution companies (**Tenants**) can manage geographic property hierarchies, residents (**Customers**), gas hardware (**Meters**), consumption indices (**Readings**), complex tiered volumetric schedules (**Slabs & Charges**), automated **Invoicing**, financial ledger bookkeeping (**Payments & Ledger Transactions**), and supply-chain logistics (**Inventory & Warehouses**).

### 1.1 The Operational Operations Flow

```mermaid
graph TD
    A[1. Tenant Registration] -->|Subscription Plan Activation| B[2. Physical Infrastructure Setup]
    B -->|Create Branches, Buildings, Flats, Stores| C[3. Customer Occupancy & Meter Mapping]
    C -->|Assign Active Tenant/Owner & Hardware serials| D[4. Charge Rate & Slab Configuration]
    D -->|Set Unit Slabs / Fixed / Percentage Taxes| E[5. Periodical Meter Readings Entry]
    E -->|Log Current Index vs Previous Index| F[6. Monthly Billing Calculation Run]
    F -->|Calculate Slabs, Subtotal Charges, VAT, Carried Dues| G[7. Communication Queue Alerts Dispatch]
    G -->|Asynchronous SMS & Email Invoicing Links| H[8. Payment Collection & Recording]
    H -->|BKash Gateway / Cash Ledger Ledger Bookkeeping| I[9. Schedulers & Background Workers]
    I -->|Nightly Penalty Cron & Low Stock Alerts| J[10. Inventory & Stock Distributions]
```

### 1.2 Step-by-Step Operations in Practice

1. **Multi-Tenant Onboarding**: A gas company registers on the platform. The system provisions their workspace with subdomain separation (e.g., `metro.gaspay.cc`) to enforce data isolation, and checks registration counts against their **Subscription Plan** limits (max branches, buildings, flats).
2. **Hierarchy Definition**: The Tenant Admin configures their physical hierarchy:
   - **Branch**: Operational territories (*Dhaka Division*, *Chittagong Division*).
   - **Building**: Physical estates (*Rose Valley Heights*).
   - **Flat**: Specific residential or business customer units (*Flat 3A*, *Flat 3B*).
3. **Customer Registration & Meter Assignment**:
   - A customer is registered. The customer is mapped to a flat via **Customer-Flat Mappings** as a Renter or Owner.
   - A physical Gas Meter is registered and installed inside the flat.
4. **Billing Charges Setup (The Pricing Engine)**: The admin configures the active pricing rules for the building:
   - **UNIT Charge (Gas consumption fee)**: Volumetric tiered slabs:
     - *Slab 1*: 0 to 100 units $\rightarrow$ **$120 / unit**
     - *Slab 2*: 101+ units $\rightarrow$ **$180 / unit**
   - **FIXED Charge (Maintenance fee)**: **$500 / month** flat.
   - **PERCENTAGE Charge (VAT)**: **5%** applied on the subtotal.
5. **Periodic Meter Reading**: At the end of the billing month, a Meter Reader visits the building and logs the current index index:
   - *Previous Reading*: **120 units** (auto-populated by the database from the last reading)
   - *Current Reading*: **200 units** (manually input by the reader)
   - *Calculated Consumption*: $200 - 120 = 80$ units.
6. **Billing Run**:
   - The billing engine takes the consumption ($80$ units) and references the building's active charge configurations.
   - **Step 1: Gas Fee**: $80 \times 120$ (Slab 1 rate) = **$9,600**
   - **Step 2: Service Charge**: Fixed rate = **$500**
   - **Step 3: Subtotal**: $9,600 + 500 = $10,100
   - **Step 4: Percentage VAT**: $10,100 \times 0.05$ (5%) = **$505**
   - **Step 5: Total Calculated**: $10,100 + 505 = $10,605
   - **Step 6: Previous Due Carryover**: The system checks if there are unpaid bills from the previous month. If there is a previous due of **$1,200**, it is added to the due amount.
   - **Step 7: Total Payable (Due Amount)**: $10,605 + 1,200 = **$11,805**
   - The invoice is finalized, creating a master record in the `bills` table and detailed audit rows in the `bill_items` table.
7. **Communication Dispatch**: The system automatically adds an outbound record to the `notification_queue`. An asynchronous worker picks it up and sends an SMS to the customer containing a summarized breakdown and a link to view/download their PDF bill.
8. **Payment & Bookkeeping Ledger**:
   - The customer receives an SMS containing a secure payment link.
   - They complete the payment digitally via the bKash payment gateway.
   - The billing platform captures the checkout event, maps the callback, marks the bill as `PAID` (or `PARTIALLY_PAID` if a partial payment was received), and records a corresponding transaction entry in both the `payments` ledger and double-entry style `transactions` table.
   - If an overpayment is recorded (e.g. paying $12,000 for a $11,805 bill), the surplus $195 is held on the customer profile as a credit balance to be applied to next month's invoice.

---

## 2. Database Schema ERD Deep Dive (All 38 Tables)

The GasPay schema is implemented in PostgreSQL with Prisma ORM. Below is the exhaustive specification of every table, its columns, data types, physical constraints, and relations.

---

### 2.1 Tenant & System Configuration Core

#### 1. `subscription_plans`

Defines subscription billing structures and system resource limits for GasPay tenants.

- **Schema fields**:
  - `id` (`BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY`): Unique plan identifier.
  - `name` (`VARCHAR(100)`): Tier name (e.g. *Basic*, *Enterprise*).
  - `price_monthly` / `price_yearly` (`DECIMAL`): Plan pricing.
  - `features` (`JSONB`): Active module features list.
  - `max_branch` / `max_building` / `max_flats` (`INT`): Resource capacity caps.
  - `is_active` (`BOOLEAN`): Availability status.
  - `created_at` (`TIMESTAMP`): Creation audit timestamp.
- **Relations**: Parent to `tenants` (1-to-Many).

#### 2. `tenants`

The main tenant company profile. All tables are isolated by referencing this record.

- **Schema fields**:
  - `id` (`BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY`): Unique tenant ID.
  - `company_name` (`VARCHAR(200)`): Registered gas corporation name.
  - `contact_person` (`VARCHAR(150)`): Primary contact agent.
  - `phone` / `email` (`VARCHAR`): Contact details.
  - `subscription_plan_id` (`BIGINT`): Linked plan.
  - `status` (`VARCHAR(20)`): State of the tenant (*ACTIVE*, *SUSPENDED*).
  - `created_at` (`TIMESTAMP`): Registration date.
- **Relations**:
  - Mapped to `subscription_plans` via `subscription_plan_id` FK.
  - Parent to all multi-tenant operational tables via `tenant_id` FK.

---

### 2.2 Security, Role-Based Access (RBAC) & Overrides

```mermaid
erDiagram
    roles ||--o{ role_permissions : "contains"
    permissions ||--o{ role_permissions : "mapped"
    users ||--|| roles : "has_role"
    users ||--o{ audit_logs : "records_actions"
```

#### 3. `roles`

Role templates created by the Tenant Admin to manage permissions in groups.

- **Schema fields**:
  - `id` (`BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY`): Unique role ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `name` (`VARCHAR(50)`): Role label (e.g., *Operator*, *Technician*, *Reader*).
  - `created_at` (`TIMESTAMP`): Creation date.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 4. `permissions`

Global dictionary of functional system permissions.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique permission ID.
  - `code` (`VARCHAR(50)`): Key used in guard validation (e.g., `can_read_meter`).
  - `name` (`VARCHAR(100)`): Explanation.
  - `created_at` (`TIMESTAMP`): Creation timestamp.
- **Relations**: Mapped to `role_permissions` (1-to-Many).

#### 5. `role_permissions`

Join table mapping standard permissions to roles.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique mapping ID.
  - `tenant_id` (`BIGINT`): Tenant key.
  - `role_id` (`BIGINT`): Mapped role.
  - `permission_id` (`BIGINT`): Mapped permission.
  - `created_at` (`TIMESTAMP`): Mapping timestamp.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `roles` via `role_id` FK.
  - Mapped to `permissions` via `permission_id` FK.

#### 6. `users`

Logins and authentication profiles for tenant administrators and staff.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique user ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `email` (`VARCHAR(150)`): Login email (unique per tenant).
  - `password_hash` (`VARCHAR(255)`): Hashed credentials.
  - `name` (`VARCHAR(100)`): Display name.
  - `role_id` (`BIGINT`): Linked role.
  - `permissions_json` (`JSONB`): Consolidated copy of user-specific permissions.
  - `is_active` (`BOOLEAN`): Account access control flag.
  - `last_login` (`TIMESTAMP`): Last successful login timestamp.
  - `created_at` / `updated_at` (`TIMESTAMP`): Creation and modification logs.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `roles` via `role_id` FK.
- **Role Override Mechanism**: When a user is assigned a role, the system copies the corresponding permissions into `permissions_json`. This enables Tenant Admins to add or remove individual permissions for a specific employee without modifying the standard role template.

---

### 2.3 Physical Property Hierarchy

#### 7. `branches`

Optional geographic branches for tenants managing multiple regions.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique branch ID.
  - `tenant_id` (`BIGINT`): Tenant key.
  - `name` (`VARCHAR(150)`): Branch label.
  - `address` / `phone` (`TEXT`): Contact details.
  - `status` (`VARCHAR(20)`): Operating status.
  - `is_deleted` (`BOOLEAN`): Soft delete flag.
  - `created_at` (`TIMESTAMP`): Creation timestamp.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 8. `buildings`

Physical property buildings or real estate properties.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique building ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `branch_id` (`BIGINT`): Reference to parent branch (nullable).
  - `name` (`VARCHAR(200)`): Building name.
  - `area` (`TEXT`): Neighborhood or sector.
  - `address` (`TEXT`): Physical street address.
  - `total_floors` / `total_flats` (`INT`): Property size details.
  - `is_deleted` / `status` (`VARCHAR/BOOLEAN`): State parameters.
  - `created_at` (`TIMESTAMP`): Creation timestamp.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `branches` via `branch_id` FK (nullable).

#### 9. `flats`

Individual apartment flats or customer units inside a building.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique flat ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `branch_id` (`BIGINT`): Branch key (nullable).
  - `building_id` (`BIGINT`): Parent building key.
  - `unit_number` (`VARCHAR(50)`): Flat code (e.g. *Flat 4C*).
  - `floor` (`INT`): Floor number.
  - `size_sqft` (`INT`): Square footage (used for fixed-area charges).
  - `is_deleted` / `status` (`VARCHAR/BOOLEAN`): State parameters.
  - `created_at` (`TIMESTAMP`): Creation timestamp.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `branches` via `branch_id` FK (nullable).
  - Mapped to `buildings` via `building_id` FK.

---

### 2.4 Customers & Occupancy Mappings

#### 10. `customers`

Profiles of consumers who receive utility invoices and complete payments.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique customer ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `name` (`VARCHAR(150)`): Full name.
  - `phone` (`VARCHAR(50)`): Mobile number (used for SMS alerts).
  - `email` (`VARCHAR(150)`): Contact email.
  - `status` (`VARCHAR(20)`): Customer status (*ACTIVE*, *INACTIVE*).
  - `is_deleted` (`BOOLEAN`): Soft delete flag.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 11. `customer_flat_mappings`

Joins customer records to respective flats to track occupancy.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique mapping ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `customer_id` (`BIGINT`): Linked customer.
  - `flat_id` (`BIGINT`): Linked flat.
  - `ownership_type` (`VARCHAR(20)`): Occupancy status (*OWNER*, *RENTER*, *FAMILY*).
  - `start_date` (`DATE`): Lease start date.
  - `end_date` (`DATE`): Lease end date (nullable).
  - `is_primary` (`BOOLEAN`): Flag indicating if this customer is the primary billing contact.
  - `is_active` (`BOOLEAN`): Active lease mapping indicator.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `customers` via `customer_id` FK.
  - Mapped to `flats` via `flat_id` FK.
- **Business Rule**: **A flat can have only one active customer mapping at a time.** Vacating a flat requires setting `is_active = false` and recording an `end_date` on the mapping before a new customer can be assigned to the flat.

---

### 2.5 Employees & Responsibilities

#### 12. `employees`

Profiles of company technicians, readers, and operators.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique employee ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `branch_id` (`BIGINT`): Linked branch (nullable).
  - `building_id` (`BIGINT`): Linked building (nullable).
  - `name` (`VARCHAR(150)`): Full name.
  - `phone` / `email` (`VARCHAR`): Contact details.
  - `designation` (`VARCHAR(50)`): Job title (e.g. *Technician*, *Reader*).
  - `notes` (`VARCHAR(150)`): Internal notes.
  - `is_active` / `is_deleted` (`BOOLEAN`): State parameters.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `branches` via `branch_id` FK (nullable).
  - Mapped to `buildings` via `building_id` FK (nullable).

#### 13. `employee_unit_mappings`

Maps employees to operational units they are responsible for (such as a branch, a building, or a central supply store).

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique mapping ID.
  - `employee_id` (`BIGINT`): Reference to employee.
  - `unit_type` (`VARCHAR(30)`): Target unit category (*BRANCH*, *BUILDING*, *STORE*).
  - `unit_id` (`BIGINT`): Polymorphic reference ID corresponding to the dynamic `unit_type` entity.
  - `created_at` (`TIMESTAMP`): Audit time.
- **Relations**: Mapped to `employees` via `employee_id` FK.

---

### 2.6 Meters & Volumetric Readings

#### 14. `meters`

Physical gas hardware meters mapped to individual flats.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique meter ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `flat_id` (`BIGINT`): Linked flat.
  - `meter_number` (`VARCHAR(100)`): Hardware serial number.
  - `installation_date` (`DATE`): Date of installation.
  - `status` (`VARCHAR(20)`): Meter operational status (*ACTIVE*, *DAMAGED*, *SUSPENDED*).
  - `is_deleted` (`BOOLEAN`): Soft delete flag.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `flats` via `flat_id` FK.

#### 15. `meter_readings`

Records monthly meter index values to track gas consumption.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique reading ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `meter_id` (`BIGINT`): Linked meter.
  - `reading_month` (`DATE`): Target billing month (e.g. `2026-05-01`).
  - `previous_reading` / `current_reading` (`DECIMAL`): Index readings.
  - `consumption` (`DECIMAL`): Net gas consumption (calculated as `current_reading` - `previous_reading`).
  - `reading_date` (`DATE`): Date of reading entry.
  - `status` (`VARCHAR(20)`): Reading classification flag (*AVERAGE*, *HIGH*, *COMPLAIN*).
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `meters` via `meter_id` FK.
- **Business Rules**:
  - `current_reading` must be greater than or equal to `previous_reading`.
  - Only one meter reading can exist per meter for a given calendar month.

---

### 2.7 The Pricing & Charging Engine

#### 16. `charge_types`

The system dictionary defining categories of service fees and taxes.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique charge type ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `name` (`VARCHAR(100)`): Charge label (e.g. *Gas Usage*, *VAT*, *Maintenance*).
  - `charge_mode` (`VARCHAR(20)`): Math mode of the charge (*UNIT*, *FIXED*, *PERCENTAGE*).
  - `apply_on` (`VARCHAR(30)`): Reference amount the rate applies to (*CONSUMPTION*, *SUBTOTAL*, *TOTAL*).
  - `is_late_fee` / `is_slab` / `is_active` (`BOOLEAN`): Pricing behaviors.
  - `order_no` (`INT`): Execution priority order (critical for calculating percentages on subtotals).
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 17. `charge_configs`

Configures the rate details and active date range for a charge type.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique configuration ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `charge_type_id` (`BIGINT`): Reference to parent charge type.
  - `value` (`DECIMAL`): Base rate value (representing an amount or percentage depending on the mode).
  - `effective_from` / `effective_to` (`DATE`): Configuration date range.
  - `grace_days` (`INT`): Grace days before penalty (specifically for late fees).
  - `is_active` (`BOOLEAN`): Active configuration flag.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `charge_types` via `charge_type_id` FK.

#### 18. `charge_config_mappings`

Binds charge configurations to specific physical buildings.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique mapping ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `charge_config_id` (`BIGINT`): Reference to active config.
  - `building_id` (`BIGINT`): Target building.
  - `status` (`VARCHAR(20)`): Status.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `charge_configs` via `charge_config_id` FK.
  - Mapped to `buildings` via `building_id` FK.
- **Business Rule**: **A building can only have one active configuration of the same charge type at a time** (preventing overlapping date ranges or duplicate charges).

#### 19. `charge_slabs`

Configures tiered/slab pricing rates for volume-based pricing models.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique slab ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `charge_config_id` (`BIGINT`): Parent configuration.
  - `slab_order` (`INT`): Sequence of slab (e.g. *1, 2, 3*).
  - `from_unit` (`DECIMAL`): Starting volume threshold (inclusive).
  - `to_unit` (`DECIMAL`): Ending volume threshold (nullable for the final unlimited slab).
  - `rate` (`DECIMAL`): Rate per unit within this volume window.
  - `is_active` (`BOOLEAN`): Active status.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `charge_configs` via `charge_config_id` FK.

---

### 2.8 Invoicing, Payments & Bookkeeping Ledger

#### 20. `bills`

The master record for monthly generated customer invoices.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique bill ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `flat_id` (`BIGINT`): Target flat.
  - `bill_number` (`VARCHAR(50)`): Sequenced invoice number unique to the tenant (e.g. `GAS-2026-000145`).
  - `billing_month` (`DATE`): The calendar month billed.
  - `total_usage` (`DECIMAL`): Total gas volume consumed.
  - `total_amount` (`DECIMAL`): Total current charges.
  - `previous_due` (`DECIMAL`): Unpaid balance carried forward.
  - `due_amount` (`DECIMAL`): Total payable balance (calculated as `total_amount` + `previous_due` - `applied_credits`).
  - `late_fee_enabled` (`BOOLEAN`): Flag indicating if late fees can be applied if past the due date.
  - `late_fee_applied` (`BOOLEAN`): Flag indicating if a late fee penalty has already been applied.
  - `late_fee_applied_at` (`TIMESTAMP`): Penalty timestamp.
  - `grace_days` (`INT`): Billed grace period.
  - `due_date` (`DATE`): Payment deadline.
  - `status` (`VARCHAR(20)`): Payment status (*UNPAID*, *PARTIALLY_PAID*, *PAID*).
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `flats` via `flat_id` FK.

#### 21. `bill_items`

Itemized line rows detailing each charge computed on an invoice.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique item ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `bill_id` (`BIGINT`): Parent bill.
  - `charge_config_id` (`BIGINT`): Reference charge config for auditing.
  - `charge_name` (`VARCHAR(50)`): Display name (e.g. *Service Tax*, *Gas slab 1*).
  - `charge_type` (`VARCHAR(20)`): Math mode of the charge (*UNIT*, *FIXED*, *PERCENTAGE*).
  - `base_on` (`VARCHAR(30)`): Application base (*CONSUMPTION*, *SUBTOTAL*, *TOTAL*).
  - `value` (`DECIMAL`): Rate configured.
  - `calculated_amount` (`DECIMAL`): Calculated total for this item.
  - `created_at` (`TIMESTAMP`): Mapping date.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `bills` via `bill_id` FK.
  - Mapped to `charge_configs` via `charge_config_id` FK.

#### 22. `payments`

Records financial payments made by customers.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique payment ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `bill_id` (`BIGINT`): Linked bill.
  - `flat_id` (`BIGINT`): Linked flat (for query optimization).
  - `amount` (`DECIMAL`): Amount paid.
  - `payment_method` (`VARCHAR(30)`): Payment channel (*CASH*, *BKASH*, *BANK*, *CARD*).
  - `payment_date` (`DATE`): Date of transaction.
  - `reference_no` (`VARCHAR(100)`): Reference number (e.g. bkash transaction ID or check number).
  - `notes` (`TEXT`): Transaction memo notes.
  - `created_at` (`TIMESTAMP`): Record entry timestamp.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `bills` via `bill_id` FK.
  - Mapped to `flats` via `flat_id` FK.

#### 23. `transactions`

Audited general ledger entries for general accounting and transaction tracking.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique transaction ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `reference_id` (`BIGINT`): Mapped record ID (corresponds to a payments table index).
  - `payment_method` (`VARCHAR(50)`): Payment mode used.
  - `amount` (`DECIMAL`): Financial amount.
  - `transaction_date` (`DATE`): Ledger date.
  - `reference_no` (`VARCHAR(100)`): Transaction reference number.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

---

### 2.9 Materials Procurement & Warehouses

#### 24. `suppliers`

Supplier directory for hardware parts and equipment.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique supplier ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `name` (`VARCHAR(150)`): Vendor name.
  - `contact_person` (`VARCHAR(100)`): Primary representative.
  - `phone` / `email` / `address` (`TEXT`): Supplier contact details.
  - `supplier_type` (`VARCHAR(50)`): Category (e.g. *Gas supplier*, *Meter vendor*).
  - `is_active` (`BOOLEAN`): Active vendor flag.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 25. `materials`

Dictionary of inventory items that can be purchased.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique material ID.
  - `name` (`VARCHAR(150)`): Name of item (e.g. *Regulator type A*, *Copper pipes*).
  - `type` (`VARCHAR(30)`): Category.
  - `unit` (`VARCHAR(20)`): Base unit of measurement (e.g. *kg*, *pcs*, *liters*).
  - `is_active` (`BOOLEAN`): Available status.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Parent to `material_purchases`.

#### 26. `material_purchases`

Purchase ledger logging bulk hardware procurement.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique purchase ID.
  - `material_id` (`BIGINT`): Reference to catalog material.
  - `supplier_id` (`BIGINT`): Supplier reference.
  - `purchase_reference` (`VARCHAR(100)`): Supplier invoice number.
  - `quantity` (`NUMERIC`): Volume purchased.
  - `unit_price` (`NUMERIC`): Base cost per item.
  - `additional_charges` (`NUMERIC`): Logistics, shipping, or custom taxes.
  - `total_price` (`NUMERIC`): Auto-calculated total invoice (calculated as `quantity` $\times$ `unit_price` + `additional_charges`).
  - `effective_unit_cost` (`NUMERIC`): Auto-calculated true cost per item (calculated as `total_price` / `quantity`).
  - `purchase_date` (`DATE`): Date of purchase.
  - `note` (`TEXT`): Custom notes.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `materials` via `material_id` FK.
  - Mapped to `suppliers` via `supplier_id` FK.

#### 27. `stores`

Physical warehouse storage spaces (e.g. *Central Godown*).

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique store ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `branch_id` (`BIGINT`): Optional controlling branch.
  - `name` (`VARCHAR(150)`): Warehouse store name.
  - `address` (`TEXT`): Address of facility.
  - `contact_phone` (`VARCHAR(50)`): Storekeeper phone.
  - `is_active` (`BOOLEAN`): Operational status.
  - `created_at` (`TIMESTAMP`): Creation time.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `branches` via `branch_id` FK (nullable).

---

### 2.10 Inventory Tracking & Movements

#### 28. `inventory_items`

Master list of catalog products tracked in inventory stock (e.g. *45kg Gas Cylinder*, *Gas Meter M1*).

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique product ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `name` (`VARCHAR(100)`): Item catalog name.
  - `category` (`VARCHAR(50)`): Item category (*Cylinder*, *Meter*, *Pipe*).
  - `unit` (`VARCHAR(20)`): Measurement unit (*pcs*, *kg*).
  - `is_active` (`BOOLEAN`): Availability status.
  - `created_at` (`TIMESTAMP`): Creation time.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 29. `inventory_stock`

Tracks current stock balances per item per polymorphic location (Branch, Building, or Store).

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique stock ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `item_id` (`BIGINT`): Mapped catalog product.
  - `location_id` (`BIGINT`): Target entity ID.
  - `location_type` (`VARCHAR(20)`): Entity target category (*STORE*, *BRANCH*, *BUILDING*).
  - `current_quantity` (`DECIMAL`): Available stock count.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `inventory_items` via `item_id` FK.

#### 30. `inventory_transactions`

Audits every item movement transaction (procurements, issues, transfers, or customer returns).

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique transaction ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `item_id` (`BIGINT`): Mapped product.
  - `from_location_id` (`BIGINT`): Origin location reference.
  - `to_location_id` (`BIGINT`): Destination location reference.
  - `transaction_type` (`VARCHAR(20)`): Mode of transfer (*IN*, *OUT*, *TRANSFER*, *RETURN*).
  - `quantity` (`DECIMAL`): Quantity of items moved.
  - `reference_id` (`BIGINT`): Reference ID linking the transfer to an invoice or purchase order (nullable).
  - `transaction_date` (`DATE`): Processing date.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `inventory_items` via `item_id` FK.

---

### 2.11 Settings & Configurations

#### 31. `billing_settings`

Custom monthly billing cycles configured per building or tenant.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique settings ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `building_id` (`BIGINT`): Target building.
  - `billing_day` (`INT`): Day of the month bills are generated (e.g. *25th*).
  - `due_days` (`INT`): Number of days allowed to pay the bill (e.g. *15 days*).
  - `grace_days` (`INT`): Extra grace days allowed before late fees are applied.
  - `round_off_to` (`INT`): Rounding factor for billing totals.
  - `currency` (`VARCHAR(10)`): Currency code (e.g., *USD*, *BDT*).
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `buildings` via `building_id` FK.

#### 32. `sms_settings`

API gateway credentials for sending SMS notifications.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique settings ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `provider` (`VARCHAR(50)`): SMS Gateway provider name.
  - `api_key` (`VARCHAR(255)`): Security API key.
  - `sender_id` (`VARCHAR(50)`): Registered outbound Sender ID.
  - `is_active` (`BOOLEAN`): Gateway availability toggle.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 33. `sms_templates`

Custom message templates for billing updates and payment alerts.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique template ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `template_name` (`VARCHAR(100)`): Template label.
  - `message_body` (`JSONB`): Outbound message content with placeholder injection fields.
  - `type` (`VARCHAR(50)`): Target event category (*BillGenerate*, *PaymentConfirmation*, *OverdueAlert*).
  - `is_active` (`BOOLEAN`): Active status.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 34. `notification_settings`

Preferences for enabling or disabling different notification channels.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique configuration ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `is_email` / `is_sms` / `is_inapp` (`BOOLEAN`): Channel status flags.
  - `low_stock_alert` (`BOOLEAN`): Flag indicating if low stock alerts are active.
  - `overdue_alert_days` (`INT`): Number of days past the due date to send an alert.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 35. `email_settings`

SMTP gateway details for email routing.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique settings ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `smtp_host` (`VARCHAR(100)`): SMTP mail server host.
  - `smtp_port` (`INT`): SMTP connection port.
  - `smtp_user` (`VARCHAR(100)`): SMTP username.
  - `smtp_password` (`VARCHAR(255)`): SMTP password.
  - `from_email` (`VARCHAR(150)`): Outbound sender email.
  - `from_name` (`VARCHAR(100)`): Outbound sender name.
  - `is_active` (`BOOLEAN`): Gateway availability toggle.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

#### 36. `general_settings`

Dynamic key-value settings store for custom business configurations.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique settings ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `setting_key` (`VARCHAR(100)`): Config key.
  - `setting_value` (`TEXT`): Config value.
  - `created_at` / `updated_at` (`TIMESTAMP`): Audit times.
- **Relations**: Mapped to `tenants` via `tenant_id` FK.

---

### 2.12 Audit & Asynchronous Dispatch Queues

#### 37. `audit_logs`

Tracks all operations performed on database records.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique audit log ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `user_id` (`BIGINT`): Employee user who performed the operation.
  - `action` (`VARCHAR(50)`): Database action performed (*Create*, *Update*, *Delete*).
  - `entity_type` (`VARCHAR(50)`): Target table name (e.g. *bills*, *charge_configs*).
  - `entity_id` (`BIGINT`): Record ID affected.
  - `description` (`TEXT`): Custom explanation.
  - `old_values` (`JSONB`): State of the row before modification (nullable).
  - `new_values` (`JSONB`): State of the row after modification (nullable).
  - `ip_address` (`VARCHAR(50)`): Client IP address.
  - `created_at` (`TIMESTAMP`): Action timestamp.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `users` via `user_id` FK.

#### 38. `notification_queue`

Central queue table managing outbound SMS and email deliveries.

- **Schema fields**:
  - `id` (`BIGINT PRIMARY KEY`): Unique queue ID.
  - `tenant_id` (`BIGINT`): Reference to tenant.
  - `user_id` (`BIGINT`): Recipient user references (nullable).
  - `notification_type` (`VARCHAR(30)`): Communication channel (*SMS*, *EMAIL*, *INAPP*).
  - `recipient` (`VARCHAR(100)`): Mobile number or email address.
  - `subject` (`VARCHAR(150)`): Email subject line (nullable for SMS).
  - `message_body` (`TEXT`): Content dispatched.
  - `status` (`VARCHAR(20)`): Queue delivery status (*PENDING*, *SENT*, *FAILED*).
  - `retry_count` (`INT`): Number of delivery attempts.
  - `scheduled_at` (`TIMESTAMP`): Target delivery schedule time.
  - `sent_at` (`TIMESTAMP`): Successful delivery timestamp.
  - `created_at` (`TIMESTAMP`): Queue registration timestamp.
- **Relations**:
  - Mapped to `tenants` via `tenant_id` FK.
  - Mapped to `users` via `user_id` FK (nullable).

---

## 3. Module-by-Module API Endpoint Specifications (Postman Reference Manual)

Below is the complete API reference manual for GasPay. The API follows standard RESTful practices. Subdomains are dynamically intercepted to identify the active `tenant_id`. Payload schemas are formatted in standard camelCase, which maps directly to the snake_case database schema using Prisma ORM mapping.

---

### Module 3.1: Security, Access Control (RBAC), & Tenant Profile

#### 3.1.1 `GET /api/v1/tenant/profile`

Retrieve active tenant company profile, subscription limits, and current resource usage.

- **Request Header**: `Cookies: accessToken=<token>`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "tenantId": "14",
    "companyName": "Metropolitan Gas Co.",
    "subdomain": "metropolitan-gas-co",
    "subscriptionPlan": {
      "name": "Professional Tier",
      "priceMonthly": "250.00",
      "maxBranches": 5,
      "maxBuildings": 50,
      "maxFlats": 1000
    },
    "currentUsage": {
      "branches": 2,
      "buildings": 18,
      "flats": 580
    }
  }
}
```

#### 3.1.2 `PATCH /api/v1/tenant/profile`

Update tenant company contact information.

- **Request Payload**:

```json
{
  "companyName": "Metropolitan Gas Corporation Ltd.",
  "contactPerson": "Zahid Hasan Chowdhury",
  "phone": "+8801712345679"
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Tenant profile updated successfully.",
  "data": {
    "tenantId": "14",
    "companyName": "Metropolitan Gas Corporation Ltd.",
    "contactPerson": "Zahid Hasan Chowdhury"
  }
}
```

#### 3.1.3 `POST /api/v1/auth/register`

Registers a new Tenant organization and sets up their Tenant Admin account.

- **Request Payload**:

```json
{
  "companyName": "Metropolitan Gas Co.",
  "contactPerson": "Zahid Hasan",
  "phone": "+8801712345678",
  "email": "admin@metrogas.cc",
  "password": "SecurePassword123",
  "subscriptionPlanId": 2
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Tenant registered successfully.",
  "data": {
    "tenantId": "14",
    "companyName": "Metropolitan Gas Co.",
    "subdomain": "metropolitan-gas-co",
    "adminUser": {
      "id": "205",
      "name": "Zahid Hasan",
      "email": "admin@metrogas.cc",
      "role": "Tenant Admin"
    }
  }
}
```

#### 3.1.4 `POST /api/v1/auth/login`

Authenticates a user and returns their JWT token and authorization permissions.

- **Request Payload**:

```json
{
  "email": "admin@metrogas.cc",
  "password": "SecurePassword123"
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Authentication successful.",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.refresh...",
    "user": {
      "id": "205",
      "name": "Zahid Hasan",
      "email": "admin@metrogas.cc",
      "role": {
        "id": "2",
        "name": "Tenant Admin"
      },
      "permissions": [
        "can_read_meter",
        "can_generate_bill",
        "can_edit_charges",
        "can_record_payment",
        "can_view_reports"
      ]
    }
  }
}
```

#### 3.1.5 `POST /api/v1/auth/refresh`

Refresh active expired JWT access token.

- **Request Payload**:

```json
{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.refresh..."
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.newAccess..."
  }
}
```

#### 3.1.6 `GET /api/v1/users?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Tariqul&roleId=3&isActive=true`

List all tenant staff login profiles.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 2
  },
  "data": [
    {
      "id": "205",
      "name": "Zahid Hasan",
      "email": "admin@metrogas.cc",
      "isActive": true,
      "role": "Tenant Admin"
    },
    {
      "id": "206",
      "name": "Tariqul Islam",
      "email": "tariq@metrogas.cc",
      "isActive": true,
      "role": "Meter Reader"
    }
  ]
}
```

#### 3.1.7 `PATCH /api/v1/users/:id/permissions`

Overrides permissions for a specific user.

- **Request Payload**:

```json
{
  "permissionsOverride": {
    "add": ["can_correct_bill"],
    "remove": ["can_view_reports"]
  }
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "User permissions updated successfully.",
  "data": {
    "userId": "205",
    "consolidatedPermissions": [
      "can_read_meter",
      "can_generate_bill",
      "can_edit_charges",
      "can_record_payment",
      "can_correct_bill"
    ]
  }
}
```

#### 3.1.8 `GET /api/v1/roles?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Reader`

List roles configured under the tenant workspace.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 3
  },
  "data": [
    {
      "id": "1",
      "name": "Tenant Admin"
    },
    {
      "id": "2",
      "name": "Operator"
    },
    {
      "id": "3",
      "name": "Meter Reader"
    }
  ]
}
```

#### 3.1.9 `POST /api/v1/roles`

Create a custom role template.

- **Request Payload**:

```json
{
  "name": "Supervisor"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Resource created successfully",
  "data": {
    "id": "4",
    "name": "Supervisor"
  }
}
```

#### 3.1.10 `GET /api/v1/permissions`

Fetch all global platform permissions managed by platform.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 3
  },
  "data": [
    {
      "id": "1",
      "code": "can_read_meter",
      "name": "Log Meter readings"
    },
    {
      "id": "2",
      "code": "can_generate_bill",
      "name": "Process monthly bills"
    },
    {
      "id": "3",
      "code": "can_correct_bill",
      "name": "Recalculate invoices"
    }
  ]
}
```

---

### Module 3.2: Structural Property Hierarchy

#### 3.2.1 `GET /api/v1/branches?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Uttara&status=ACTIVE`

List geographic branches.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "5",
      "name": "Dhaka East Sub-division",
      "status": "ACTIVE"
    }
  ]
}
```

#### 3.2.2 `POST /api/v1/branches`

Creates a new geographic branch.

- **Request Payload**:

```json
{
  "name": "Dhaka East Sub-division",
  "address": "House 12, Road 4, Sector 3, Uttara, Dhaka",
  "phone": "+8801711122233"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Branch created successfully.",
  "data": {
    "id": "5",
    "name": "Dhaka East Sub-division",
    "status": "ACTIVE"
  }
}
```

#### 3.2.3 `GET /api/v1/buildings?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Rose&status=ACTIVE&branchId=5`

List property buildings. Supports area or branch query filters.

- **Request Parameters**: `/api/v1/buildings?branchId=5&area=Uttara`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "18",
      "name": "Rose Valley Heights",
      "area": "Uttara Sector 3",
      "totalFlats": 40
    }
  ]
}
```

#### 3.2.4 `POST /api/v1/buildings`

Creates a new property building.

- **Request Payload**:

```json
{
  "branchId": 5,
  "name": "Rose Valley Heights",
  "area": "Uttara Sector 3",
  "address": "Plot 45, Road 2, Sector 3, Uttara, Dhaka",
  "totalFloors": 10,
  "totalFlats": 40
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Building created successfully.",
  "data": {
    "id": "18",
    "branchId": "5",
    "name": "Rose Valley Heights",
    "status": "ACTIVE"
  }
}
```

#### 3.2.5 `GET /api/v1/flats?page=1&limit=9&sortBy=unitNumber&sortOrder=asc&searchTerm=Flat%204A&status=OCCUPIED&buildingId=18`

List flats. Supports buildingId and occupied status query parameters.

- **Request Parameters**: `/api/v1/flats?buildingId=18&status=VACANT`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "104",
      "unitNumber": "Flat 4A",
      "floor": 4,
      "status": "VACANT"
    }
  ]
}
```

#### 3.2.6 `POST /api/v1/flats`

Creates individual flats/units inside a building.

- **Request Payload**:

```json
{
  "buildingId": 18,
  "unitNumber": "Flat 4A",
  "floor": 4,
  "sizeSqft": 1450
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Flat created successfully.",
  "data": {
    "id": "104",
    "buildingId": "18",
    "unitNumber": "Flat 4A",
    "status": "VACANT"
  }
}
```

---

### Module 3.3: Customers CRM & Occupancy Mappings

#### 3.3.1 `GET /api/v1/customers?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Mahbubur&status=ACTIVE&startDate=2026-05-01&endDate=2026-05-19`

Search and list registered customer profiles.

- **Request Parameters**: `/api/v1/customers?search=Mahbubur`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "89",
      "name": "Mahbubur Rahman",
      "phone": "+8801911223344",
      "status": "ACTIVE"
    }
  ]
}
```

#### 3.3.2 `POST /api/v1/customers`

Registers a customer profile.

- **Request Payload**:

```json
{
  "name": "Mahbubur Rahman",
  "phone": "+8801911223344",
  "email": "mahbub@gmail.com"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Customer profile registered.",
  "data": {
    "id": "89",
    "name": "Mahbubur Rahman",
    "status": "ACTIVE"
  }
}
```

#### 3.3.3 `POST /api/v1/customers/flat-mappings`

Maps a customer to a flat.

- **Request Payload**:

```json
{
  "customerId": 89,
  "flatId": 104,
  "ownershipType": "RENTER",
  "startDate": "2026-05-01",
  "isPrimary": true
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Customer successfully assigned to unit.",
  "data": {
    "id": "312",
    "customerId": "89",
    "flatId": "104",
    "ownershipType": "RENTER",
    "isPrimary": true,
    "isActive": true
  }
}
```

#### 3.3.4 `PATCH /api/v1/customers/flat-mappings/:id/deactivate`

Deactivates a customer mapping when they vacate the flat.

- **Request Payload**:

```json
{
  "endDate": "2026-05-31"
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Customer mapping deactivated.",
  "data": {
    "id": "312",
    "isActive": false,
    "endDate": "2026-05-31"
  }
}
```

---

### Module 3.4: Employee Management & Deployments

#### 3.4.1 `GET /api/v1/employees?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Selim&designation=Reader&branchId=5`

List company employee personnel.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "12",
      "name": "Mohammad Selim",
      "phone": "+8801733445566",
      "designation": "Reader",
      "isActive": true
    }
  ]
}
```

#### 3.4.2 `POST /api/v1/employees`

Add employee details.

- **Request Payload**:

```json
{
  "name": "Mohammad Selim",
  "phone": "+8801733445566",
  "email": "selim@metrogas.cc",
  "designation": "Reader",
  "notes": "Assigned route for Uttara sector 3"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Employee registered.",
  "data": {
    "id": "12",
    "name": "Mohammad Selim",
    "designation": "Reader"
  }
}
```

#### 3.4.3 `GET /api/v1/employees/:id/unit-mappings`

Fetch unit deployment mapping for the employee.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "890",
      "unitType": "BUILDING",
      "unitId": "18",
      "unitName": "Rose Valley Heights"
    }
  ]
}
```

#### 3.4.4 `POST /api/v1/employees/:id/unit-mappings`

Deploys an employee to a property building or store route.

- **Request Payload**:

```json
{
  "unitType": "BUILDING",
  "unitId": 18
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Employee responsibility mapped successfully.",
  "data": {
    "id": "890",
    "employeeId": "12",
    "unitType": "BUILDING",
    "unitId": "18"
  }
}
```

---

### Module 3.5: Gas Meters & readings Configuration

#### 3.5.1 `GET /api/v1/meters?page=1&limit=9&sortBy=meterNumber&sortOrder=asc&searchTerm=MET-77&status=ACTIVE&buildingId=18`

List gas meters.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "76",
      "flatId": "104",
      "meterNumber": "MET-77890",
      "status": "ACTIVE"
    }
  ]
}
```

#### 3.5.2 `POST /api/v1/meters`

Installs and registers a physical gas meter in a flat.

- **Request Payload**:

```json
{
  "flatId": 104,
  "meterNumber": "MET-77890",
  "installationDate": "2026-05-01",
  "status": "ACTIVE"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Gas meter registered and mapped to flat.",
  "data": {
    "id": "76",
    "flatId": "104",
    "meterNumber": "MET-77890",
    "status": "ACTIVE"
  }
}
```

#### 3.5.3 `GET /api/v1/meter-readings?page=1&limit=9&sortBy=readingMonth&sortOrder=desc&status=AVERAGE&meterId=76&startDate=2026-05-01&endDate=2026-05-19`

List monthly meter index readings.

- **Request Parameters**: `/api/v1/meter-readings?readingMonth=2026-05-01`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "1045",
      "meterNumber": "MET-77890",
      "previousReading": "100.00",
      "currentReading": "180.50",
      "consumption": "80.50"
    }
  ]
}
```

#### 3.5.4 `POST /api/v1/meter-readings`

Records monthly meter index readings and calculates net consumption.

- **Request Payload**:

```json
{
  "meterId": 76,
  "readingMonth": "2026-05-01",
  "currentReading": 180.50,
  "readingDate": "2026-05-19"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Reading saved successfully.",
  "data": {
    "id": "1045",
    "meterId": "76",
    "previousReading": "100.00",
    "currentReading": "180.50",
    "consumption": "80.50",
    "status": "AVERAGE"
  }
}
```

---

### Module 3.6: Pricing Configurator Engine

#### 3.6.1 `GET /api/v1/charge-types?page=1&limit=9&sortBy=orderNo&sortOrder=asc&searchTerm=Slabs&isActive=true`

List active charge types.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "3",
      "name": "Gas Usage Slabs",
      "chargeMode": "UNIT",
      "applyOn": "CONSUMPTION"
    }
  ]
}
```

#### 3.6.2 `POST /api/v1/charge-types`

Registers a new charge type.

- **Request Payload**:

```json
{
  "name": "Gas Usage Slabs",
  "chargeMode": "UNIT",
  "applyOn": "CONSUMPTION",
  "isLateFee": false,
  "isSlab": true,
  "orderNo": 1
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Charge type registered.",
  "data": {
    "id": "3",
    "name": "Gas Usage Slabs",
    "isSlab": true,
    "orderNo": 1
  }
}
```

#### 3.6.3 `GET /api/v1/charge-configs?page=1&limit=9&sortBy=effectiveFrom&sortOrder=desc&chargeTypeId=3&buildingId=18`

List current configurations.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "14",
      "chargeTypeId": "3",
      "value": "120.00",
      "isActive": true
    }
  ]
}
```

#### 3.6.4 `POST /api/v1/charge-configs`

Creates a configuration rate for a charge type.

- **Request Payload**:

```json
{
  "chargeTypeId": 3,
  "value": 120.00,
  "effectiveFrom": "2026-05-01",
  "effectiveTo": null,
  "graceDays": 5
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Rate configuration saved successfully.",
  "data": {
    "id": "14",
    "chargeTypeId": "3",
    "value": "120.00",
    "effectiveFrom": "2026-05-01"
  }
}
```

#### 3.6.5 `GET /api/v1/charge-configs/:id/slabs`

Fetch active pricing slabs.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 2
  },
  "data": [
    {
      "id": "1",
      "slabOrder": 1,
      "fromUnit": "0.00",
      "toUnit": "100.00",
      "rate": "120.00"
    },
    {
      "id": "2",
      "slabOrder": 2,
      "fromUnit": "101.00",
      "toUnit": null,
      "rate": "180.00"
    }
  ]
}
```

#### 3.6.6 `POST /api/v1/charge-configs/:id/slabs`

Configures volume slabs for tiered volumetric pricing.

- **Request Payload**:

```json
{
  "slabs": [
    {
      "slabOrder": 1,
      "fromUnit": 0,
      "toUnit": 100,
      "rate": 120.00
    },
    {
      "slabOrder": 2,
      "fromUnit": 101,
      "toUnit": null,
      "rate": 180.00
    }
  ]
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Pricing slabs configured successfully.",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 2
  },
  "data": [
    {
      "id": "1",
      "slabOrder": 1,
      "fromUnit": "0.00",
      "toUnit": "100.00",
      "rate": "120.00"
    },
    {
      "id": "2",
      "slabOrder": 2,
      "fromUnit": "101.00",
      "toUnit": null,
      "rate": "180.00"
    }
  ]
}
```

#### 3.6.7 `GET /api/v1/charge-configs/mappings`

View active configuration mappings.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "89",
      "chargeConfigId": "14",
      "buildingId": "18",
      "status": "ACTIVE"
    }
  ]
}
```

#### 3.6.8 `POST /api/v1/charge-configs/mappings`

Binds a charge configuration to a specific building.

- **Request Payload**:

```json
{
  "chargeConfigId": 14,
  "buildingId": 18
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Charge configuration successfully bound to building.",
  "data": {
    "id": "89",
    "chargeConfigId": "14",
    "buildingId": "18",
    "status": "ACTIVE"
  }
}
```

---

### Module 3.7: Monthly Invoicing & Billing Calculators

#### 3.7.1 `POST /api/v1/billing/preview`

Previews calculations and itemized line items before a bill is finalized.

- **Request Payload**:

```json
{
  "flatId": 104,
  "billingMonth": "2026-05-01"
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "flatId": "104",
    "billingMonth": "2026-05-01",
    "totalUsage": "80.00",
    "subTotal": "10100.00",
    "carriedDue": "1200.00",
    "previewItems": [
      {
        "chargeName": "Gas Usage Slabs",
        "chargeType": "UNIT",
        "value": "120.00",
        "calculatedAmount": "9600.00"
      },
      {
        "chargeName": "Maintenance Charge",
        "chargeType": "FIXED",
        "value": "500.00",
        "calculatedAmount": "500.00"
      },
      {
        "chargeName": "VAT",
        "chargeType": "PERCENTAGE",
        "value": "5.00",
        "calculatedAmount": "505.00"
      }
    ],
    "totalDuePayable": "11805.00"
  }
}
```

#### 3.7.2 `POST /api/v1/billing/generate`

Generates bills in bulk for all occupied flats in a building.

- **Request Payload**:

```json
{
  "buildingId": 18,
  "billingMonth": "2026-05-01"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Billing run completed successfully.",
  "data": {
    "totalProcessed": 35,
    "totalSucceeded": 35,
    "totalInvoicedAmount": "385675.00",
    "billingRunId": "9982"
  }
}
```

#### 3.7.3 `GET /api/v1/billing/bills`

Search and list generated invoices.

- **Request Parameters**: `/api/v1/billing/bills?buildingId=18&billingMonth=2026-05-01&status=UNPAID`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "88701",
      "billNumber": "GAS-2026-000145",
      "flatUnit": "Flat 4A",
      "customerName": "Mahbubur Rahman",
      "totalUsage": "80.00",
      "dueAmount": "11805.00",
      "status": "UNPAID"
    }
  ]
}
```

#### 3.7.4 `GET /api/v1/billing/bills/:id`

Retrieve detailed itemized invoice calculation and breakdown.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "88701",
    "billNumber": "GAS-2026-000145",
    "billingMonth": "2026-05-01",
    "totalUsage": "80.00",
    "previousDue": "1200.00",
    "dueAmount": "11805.00",
    "status": "UNPAID",
    "billItems": [
      {
        "id": "1",
        "chargeName": "Gas Usage Slabs",
        "calculatedAmount": "9600.00"
      },
      {
        "id": "2",
        "chargeName": "Maintenance Charge",
        "calculatedAmount": "500.00"
      },
      {
        "id": "3",
        "chargeName": "VAT",
        "calculatedAmount": "505.00"
      }
    ]
  }
}
```

#### 3.7.5 `PATCH /api/v1/billing/bills/:id/correct`

Allows Super Admins to edit and recalculate a bill.

- **Request Payload**:

```json
{
  "overrideUsage": 75.00,
  "notes": "Corrected reading error"
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Bill recalculation completed successfully.",
  "data": {
    "billId": "88701",
    "billNumber": "GAS-2026-000145",
    "totalUsage": "75.00",
    "dueAmount": "11205.00"
  }
}
```

#### 3.7.6 `DELETE /api/v1/billing/bills/:id`

Admin override to soft delete/reverse an invoice.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Bill soft-deleted successfully.",
  "data": {
    "success": true,
    "message": "Bill soft-deleted successfully."
  }
}
```

---

### Module 3.8: Payments collection & accounting Ledger

#### 3.8.1 `GET /api/v1/payments?page=1&limit=9&sortBy=paymentDate&sortOrder=desc&paymentMethod=BKASH&startDate=2026-05-01&endDate=2026-05-19&buildingId=18`

Search list of collections.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "5054",
      "billNumber": "GAS-2026-000145",
      "amount": "11205.00",
      "paymentMethod": "CASH",
      "referenceNo": "REC-9988231"
    }
  ]
}
```

#### 3.8.2 `POST /api/v1/payments/manual`

Records a cash or bank transfer payment.

- **Request Payload**:

```json
{
  "billId": 88701,
  "amount": 11205.00,
  "paymentMethod": "CASH",
  "paymentDate": "2026-05-19",
  "referenceNo": "REC-9988231",
  "notes": "Handed cash to building manager"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Payment recorded successfully.",
  "data": {
    "paymentId": "5054",
    "billId": "88701",
    "amount": "11205.00",
    "paymentStatus": "PAID"
  }
}
```

#### 3.8.3 `POST /api/v1/payments/bkash/initiate`

Initializes a bKash gateway checkout session for a customer.

- **Request Payload**:

```json
{
  "billId": 88701
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "paymentID": "BK-CHECKOUT-88902",
    "bkashURL": "https://sandbox.payment.bkash.com/redirect/token/BK-CHECKOUT-88902",
    "amount": "11205.00"
  }
}
```

#### 3.8.4 `POST /api/v1/payments/bkash/callback`

Webhook endpoint called by the payment gateway upon transaction completion.

- **Request Payload**:

```json
{
  "paymentID": "BK-CHECKOUT-88902",
  "trxID": "TRX-BKASH-889021",
  "transactionStatus": "Completed",
  "amount": "11205.00",
  "paymentDate": "2026-05-19T12:00:00Z"
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "bkash callback captured. Bill marked as PAID.",
  "data": {
    "paymentId": "5055",
    "billId": "88701",
    "trxID": "TRX-BKASH-889021",
    "ledgerUpdated": true
  }
}
```

#### 3.8.5 `GET /api/v1/ledger/transactions`

Fetch double-entry audit general ledger history.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "9901",
      "paymentMethod": "CASH",
      "amount": "11205.00",
      "transactionDate": "2026-05-19",
      "referenceNo": "REC-9988231"
    }
  ]
}
```

---

### Module 3.9: Supply Chain & Godowns Procurement

#### 3.9.1 `GET /api/v1/inventory/suppliers?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Technoworth&supplierType=METER&isActive=true`

List suppliers.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "18",
      "name": "Technoworth Meters Ltd",
      "supplierType": "METER",
      "isActive": true
    }
  ]
}
```

#### 3.9.2 `POST /api/v1/inventory/suppliers`

Registers a supply vendor.

- **Request Payload**:

```json
{
  "name": "Technoworth Meters Ltd",
  "contactPerson": "Rashedul Bari",
  "phone": "+8801755566677",
  "email": "sales@technoworth.com",
  "address": "45/A, Dilkusha C/A, Dhaka-1000",
  "supplierType": "METER"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Supplier registered successfully.",
  "data": {
    "id": "18",
    "name": "Technoworth Meters Ltd",
    "status": "ACTIVE"
  }
}
```

#### 3.9.3 `GET /api/v1/inventory/materials?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Smart&isActive=true`

List materials dictionary.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "4",
      "name": "Smart Gas Meter G1.6",
      "type": "METER",
      "unit": "pcs",
      "isActive": true
    }
  ]
}
```

#### 3.9.4 `POST /api/v1/inventory/materials`

Add raw material definitions.

- **Request Payload**:

```json
{
  "name": "Smart Gas Meter G1.6",
  "type": "METER",
  "unit": "pcs"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Resource created successfully",
  "data": {
    "id": "4",
    "name": "Smart Gas Meter G1.6",
    "type": "METER"
  }
}
```

#### 3.9.5 `GET /api/v1/inventory/materials/purchases`

List material procurement logs.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "89",
      "purchaseReference": "INV-2026-00445",
      "quantity": "100.00",
      "totalPrice": "125000.00",
      "purchaseDate": "2026-05-15"
    }
  ]
}
```

#### 3.9.6 `POST /api/v1/inventory/materials/purchases`

Logs a procurement invoice and updates stock levels (auto-calculates effective unit costs).

- **Request Payload**:

```json
{
  "materialId": 4,
  "supplierId": 18,
  "purchaseReference": "INV-2026-00445",
  "quantity": 100.00,
  "unitPrice": 1200.00,
  "additionalCharges": 5000.00,
  "purchaseDate": "2026-05-15",
  "note": "Initial installation meters purchase"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Purchase invoice recorded. Stock updated.",
  "data": {
    "id": "89",
    "totalPrice": "125000.00",
    "effectiveUnitCost": "1250.0000"
  }
}
```

---

### Module 3.10: Warehouses Stock & Movements

#### 3.10.1 `GET /api/v1/inventory/stores?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Tejgaon&isActive=true&branchId=5`

List godown stores.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "2",
      "name": "Central Dhaka Godown",
      "address": "12/A Tejgaon Industrial Area",
      "isActive": true
    }
  ]
}
```

#### 3.10.2 `POST /api/v1/inventory/stores`

Registers a godown warehouse.

- **Request Payload**:

```json
{
  "branchId": 5,
  "name": "Central Dhaka Godown",
  "address": "12/A Tejgaon Industrial Area, Dhaka",
  "contactPhone": "+8801799887766"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Godown registered successfully.",
  "data": {
    "id": "2",
    "name": "Central Dhaka Godown",
    "status": "ACTIVE"
  }
}
```

#### 3.10.3 `GET /api/v1/inventory/items?page=1&limit=9&sortBy=name&sortOrder=asc&searchTerm=Cylinder&category=Cylinder&isActive=true`

List catalog inventory items.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "8",
      "name": "45kg Liquefied Cylinder",
      "category": "Cylinder",
      "unit": "pcs",
      "isActive": true
    }
  ]
}
```

#### 3.10.4 `POST /api/v1/inventory/items`

Register a catalog inventory product.

- **Request Payload**:

```json
{
  "name": "45kg Liquefied Cylinder",
  "category": "Cylinder",
  "unit": "pcs"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Resource created successfully",
  "data": {
    "id": "8",
    "name": "45kg Liquefied Cylinder"
  }
}
```

#### 3.10.5 `GET /api/v1/inventory/stock`

Fetch current stock counts per item per location (supports filtering by `locationType` STORE/BRANCH/BUILDING).

- **Request Parameters**: `/api/v1/inventory/stock?locationType=STORE&locationId=2`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "45",
      "itemName": "45kg Liquefied Cylinder",
      "locationName": "Central Dhaka Godown",
      "currentQuantity": "150.00"
    }
  ]
}
```

#### 3.10.6 `POST /api/v1/inventory/stock/transfer`

Logs stock transfers or allocations.

- **Request Payload**:

```json
{
  "itemId": 8,
  "fromLocationId": 2,
  "toLocationId": 18,
  "transactionType": "TRANSFER",
  "quantity": 30.00,
  "referenceId": 89,
  "transactionDate": "2026-05-19"
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Stock transfer transaction recorded successfully.",
  "data": {
    "id": "1882",
    "itemId": "8",
    "quantity": "30.00",
    "transactionType": "TRANSFER"
  }
}
```

---

### Module 3.11: Settings & Alerts Dispatch

#### 3.11.1 `GET /api/v1/settings/gateways`

Fetch email and SMS gateway active details.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "smsGateway": {
      "provider": "Twilio",
      "senderId": "GASPAY",
      "isActive": true
    },
    "emailGateway": {
      "smtpHost": "smtp.gmail.com",
      "fromEmail": "billing@gaspay.cc",
      "isActive": true
    }
  }
}
```

#### 3.11.2 `POST /api/v1/settings/sms-gateway`

Configures SMS gateway settings.

- **Request Payload**:

```json
{
  "provider": "Twilio",
  "apiKey": "SK_API_KEY_SECRET_7789211",
  "senderId": "GASPAY",
  "isActive": true
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "SMS gateway configured successfully.",
  "data": {
    "success": true,
    "message": "SMS gateway configured successfully."
  }
}
```

#### 3.11.3 `POST /api/v1/settings/email-gateway`

Configures email gateway SMTP details.

- **Request Payload**:

```json
{
  "smtpHost": "smtp.gmail.com",
  "smtpPort": 587,
  "smtpUser": "billing@gaspay.cc",
  "smtpPassword": "smtpPasswordSecretKey123",
  "fromEmail": "billing@gaspay.cc",
  "fromName": "GasPay Invoice System",
  "isActive": true
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Email SMTP gateway configured successfully.",
  "data": {
    "success": true,
    "message": "Email SMTP gateway configured successfully."
  }
}
```

#### 3.11.4 `GET /api/v1/settings/sms-templates`

List reusable templates.

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "4",
      "templateName": "Invoice Generated",
      "type": "BillGenerate",
      "isActive": true
    }
  ]
}
```

#### 3.11.5 `POST /api/v1/settings/sms-templates`

Creates reusable text templates for automated alerts.

- **Request Payload**:

```json
{
  "templateName": "Invoice Generated",
  "messageBody": {
    "text": "Dear {customerName}, your bill {billNumber} has been generated. Amount: BDT {dueAmount}. Due: {dueDate}. View bill: {invoiceLink}"
  },
  "type": "BillGenerate",
  "isActive": true
}
```

- **Response Payload (201 Created)**:

```json
{
  "statusCode": 201,
  "success": true,
  "message": "Message template saved successfully.",
  "data": {
    "id": "4",
    "templateName": "Invoice Generated",
    "type": "BillGenerate"
  }
}
```

#### 3.11.6 `GET /api/v1/notifications/queue`

Inspect dispatch statuses of the outbound queue.

- **Request Parameters**: `/api/v1/notifications/queue?status=FAILED`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "99018",
      "recipient": "+8801911223344",
      "messageBody": "Dear Mahbubur...",
      "status": "FAILED",
      "retryCount": 3
    }
  ]
}
```

#### 3.11.7 `POST /api/v1/notifications/queue/retry`

Manually trigger retry attempts for failed alerts in the queue.

- **Request Payload**:

```json
{
  "queueIds": [99018]
}
```

- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Retry trigger dispatched for 1 queued notification.",
  "data": {
    "success": true,
    "message": "Retry trigger dispatched for 1 queued notification."
  }
}
```

---

### Module 3.12: Security Auditing logs

#### 3.12.1 `GET /api/v1/audit-logs`

Fetches the audit log list (supports pagination and filtering).

- **Request Query Parameters**:
  - `page=1`
  - `limit=10`
  - `entityType=bills`
- **Response Payload (200 OK)**:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 10,
    "totalCount": 1
  },
  "data": [
    {
      "id": "890212",
      "action": "UPDATE",
      "entityType": "bills",
      "entityId": "88701",
      "user": {
        "id": "205",
        "name": "Zahid Hasan"
      },
      "description": "Recalculated bill due to incorrect initial index",
      "oldValues": {
        "totalUsage": "80.00",
        "dueAmount": "11805.00"
      },
      "newValues": {
        "totalUsage": "75.00",
        "dueAmount": "11205.00"
      },
      "ipAddress": "192.168.10.45",
      "createdAt": "2026-05-19T12:05:00Z"
    }
  ]
}
```

---

## 4. Key Architectural Patterns & Best Practices

1. **Strict Multi-Tenancy**: The application intercepts subdomains via routing middleware to identify the active tenant. A global filter applies `tenant_id` to all database operations, ensuring that tenants cannot access or modify each other's data.
2. **Dynamic Role Permitting Overrides**: Inherited permissions are copied to the user's `permissions_json` field upon role assignment. This balances rapid JWT token checks with custom user overrides.
3. **Double-Entry Financial Transactions**: Every payment transaction updates a cash account and registers an matching record in both the `payments` ledger and double-entry style `transactions` table.
4. **Soft Delete**: Deleting critical records (such as branches, buildings, flats, or meters) does not execute a hard SQL delete. Instead, the application sets `is_deleted = true`, preserving historical data integrity for monthly invoicing and payments.
5. **Nightly Late Fee Workers**: Cron jobs check outstanding invoices against active `grace_days` and `due_date` settings to automatically apply penalties without requiring manual administrative tasks.

---

## 5. Complete Table Schema Reference (All 38 Tables)

Every table, every column, every constraint � documented for pre-implementation clarity.

### Domain 1: Tenant & Subscription

#### Table: subscription_plans

Defines SaaS pricing tiers and resource limits.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Unique plan ID |
| name | VARCHAR(100) | Tier name (Basic/Pro/Enterprise) |
| price_monthly | DECIMAL | Monthly cost |
| price_yearly | DECIMAL | Yearly cost |
| features | JSONB | Feature flags list |
| max_branch | INT | Max branches allowed |
| max_building | INT | Max buildings allowed |
| max_flats | INT | Max flats allowed |
| is_active | BOOLEAN | Plan available for signup |
| created_at | TIMESTAMP | Creation time |

#### Table: tenants

Core company record � all data isolates here via tenant_id FK.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Unique tenant ID |
| company_name | VARCHAR(200) | Gas company name |
| contact_person | VARCHAR(150) | Primary contact |
| phone | VARCHAR(50) | Contact phone |
| email | VARCHAR(150) | Contact email |
| subscription_plan_id | BIGINT FK | Links to subscription_plans.id |
| status | VARCHAR(20) | ACTIVE or SUSPENDED |
| created_at | TIMESTAMP | Registration date |

### Domain 2: RBAC & Users

#### Table: roles

Permission group templates per tenant.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Role ID |
| tenant_id | BIGINT FK | Owner tenant |
| name | VARCHAR(50) | Role label (Admin/Operator/Reader) |
| created_at | TIMESTAMP | Creation time |

#### Table: permissions

Global dictionary of all system permission codes.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Permission ID |
| code | VARCHAR(50) | e.g. can_read_meter, can_generate_bill |
| name | VARCHAR(100) | Human readable description |
| created_at | TIMESTAMP | Creation time |

#### Table: role_permissions

Junction: maps permissions to roles.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Mapping ID |
| tenant_id | BIGINT FK | Tenant isolation |
| role_id | BIGINT FK | Links to roles.id |
| permission_id | BIGINT FK | Links to permissions.id |
| created_at | TIMESTAMP | Mapping time |

**Unique index**: (role_id, permission_id) � prevents duplicate mappings.

#### Table: users

Login accounts for all tenant staff.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | User ID |
| tenant_id | BIGINT FK | Owner tenant |
| email | VARCHAR(150) | Login email (unique per tenant) |
| password_hash | VARCHAR(255) | bcrypt hash |
| name | VARCHAR(100) | Display name |
| role_id | BIGINT FK | Linked role template |
| permissions_json | JSONB | Copied + custom permissions array |
| is_active | BOOLEAN | Account enabled flag |
| last_login | TIMESTAMP | Last auth timestamp |
| created_at | TIMESTAMP | Account creation time |
| updated_at | TIMESTAMP | Last update time |

> **KEY MECHANISM**: On user creation, the system copies all permissions from role_permissions into permissions_json. Guards check this JSON � NOT the DB at runtime. Admins can add/remove individual codes without changing the role.

### Domain 3: Property Hierarchy

#### Table: branches

Optional geographic operational divisions.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Branch ID |
| tenant_id | BIGINT FK | Owner tenant |
| name | VARCHAR(150) | Branch label |
| address | TEXT | Physical address |
| phone | VARCHAR(50) | Contact number |
| status | VARCHAR(20) | ACTIVE / INACTIVE |
| is_deleted | BOOLEAN | Soft delete flag |
| created_at | TIMESTAMP | Creation time |

#### Table: buildings

Physical property estates where gas pipelines are installed.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Building ID |
| tenant_id | BIGINT FK | Owner tenant |
| branch_id | BIGINT FK NULL | Optional parent branch |
| name | VARCHAR(200) | Building name |
| area | TEXT | Neighbourhood/sector for filtering |
| address | TEXT | Street address |
| total_floors | INT | Number of floors |
| total_flats | INT | Number of flats |
| is_deleted | BOOLEAN | Soft delete flag |
| status | VARCHAR(20) | ACTIVE / INACTIVE |
| created_at | TIMESTAMP | Creation time |

#### Table: flats

Individual residential/commercial units inside a building.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Flat ID |
| tenant_id | BIGINT FK | Owner tenant |
| branch_id | BIGINT FK NULL | Optional branch ref |
| building_id | BIGINT FK | Parent building |
| unit_number | VARCHAR(50) | Flat code e.g. Flat 4A |
| floor | INT | Floor number |
| size_sqft | INT | Area in sqft |
| is_deleted | BOOLEAN | Soft delete flag |
| status | VARCHAR(20) | VACANT / OCCUPIED |
| created_at | TIMESTAMP | Creation time |

### Domain 4: Customers & Occupancy

#### Table: customers

Resident consumer profiles.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Customer ID |
| tenant_id | BIGINT FK | Owner tenant |
| name | VARCHAR(150) | Full name |
| phone | VARCHAR(50) | Mobile (used for SMS alerts) |
| email | VARCHAR(150) | Email address |
| status | VARCHAR(20) | ACTIVE / INACTIVE |
| is_deleted | BOOLEAN | Soft delete flag |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: customer_flat_mappings

Links customers to flats as occupants.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Mapping ID |
| tenant_id | BIGINT FK | Owner tenant |
| customer_id | BIGINT FK | Links to customers.id |
| flat_id | BIGINT FK | Links to flats.id |
| ownership_type | VARCHAR(20) | OWNER / RENTER / FAMILY |
| start_date | DATE | Lease start date |
| end_date | DATE NULL | Lease end date |
| is_primary | BOOLEAN | Primary billing contact flag |
| is_active | BOOLEAN | Active occupancy flag |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

> **BUSINESS RULE**: Unique partial index on (flat_id) WHERE is_active = true. Only ONE active customer per flat at any time. One customer CAN map to multiple flats.

### Domain 5: Employees

#### Table: employees

Field staff � meter readers, technicians, operators.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Employee ID |
| tenant_id | BIGINT FK | Owner tenant |
| branch_id | BIGINT FK NULL | Optional branch link |
| building_id | BIGINT FK NULL | Optional building link |
| name | VARCHAR(150) | Full name |
| phone | VARCHAR(50) | Contact phone |
| email | VARCHAR(150) | Email address |
| designation | VARCHAR(50) | Reader / Technician / Supervisor |
| notes | VARCHAR(150) | Internal notes |
| is_active | BOOLEAN | Active status |
| is_deleted | BOOLEAN | Soft delete flag |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: employee_unit_mappings

Assigns employees to operational units polymorphically.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Mapping ID |
| employee_id | BIGINT FK | Links to employees.id |
| unit_type | VARCHAR(30) | BRANCH / BUILDING / STORE |
| unit_id | BIGINT | ID of the target entity |
| created_at | TIMESTAMP | Mapping time |

> **POLYMORPHIC**: unit_id refers to branches.id, buildings.id, or stores.id depending on unit_type.

### Domain 6: Meters & Readings

#### Table: meters

Physical gas hardware installed in flats.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Meter ID |
| tenant_id | BIGINT FK | Owner tenant |
| flat_id | BIGINT FK | Installed flat |
| meter_number | VARCHAR(100) | Hardware serial number |
| installation_date | DATE | Install date |
| status | VARCHAR(20) | ACTIVE / DAMAGED / SUSPENDED |
| is_deleted | BOOLEAN | Soft delete flag |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

> **UNIQUE**: One meter per flat. Unique index on (flat_id). Cannot delete flat with active meter (RESTRICT).

#### Table: meter_readings

Monthly gas consumption index logs.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Reading ID |
| tenant_id | BIGINT FK | Owner tenant |
| meter_id | BIGINT FK | Links to meters.id |
| reading_month | DATE | Billing month e.g. 2026-05-01 |
| previous_reading | DECIMAL | Auto-pulled from last reading |
| current_reading | DECIMAL | Manually entered by operator |
| consumption | DECIMAL | Auto-calculated: current - previous |
| reading_date | DATE | Date entry was logged |
| status | VARCHAR(20) | AVERAGE / HIGH / COMPLAIN |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

> **UNIQUE**: Composite index on (meter_id, reading_month). Only ONE reading per meter per month.
> **VALIDATION**: current_reading must be >= previous_reading. If negative, set consumption=0 and flag as COMPLAIN.

### Domain 7: Pricing Engine

#### Table: charge_types

Dictionary of service charge categories.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Charge type ID |
| tenant_id | BIGINT FK | Owner tenant |
| name | VARCHAR(100) | Label e.g. Gas Usage / VAT / Maintenance |
| charge_mode | VARCHAR(20) | UNIT / FIXED / PERCENTAGE |
| apply_on | VARCHAR(30) | CONSUMPTION / SUBTOTAL / TOTAL |
| is_late_fee | BOOLEAN | Is this a late penalty charge? |
| is_slab | BOOLEAN | Uses volumetric slab ranges? |
| order_no | INT | Execution priority (1=first) |
| is_active | BOOLEAN | Active status |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

> **order_no is CRITICAL**: Percentage charges on SUBTOTAL must run AFTER unit and fixed charges. Order controls the calculation sequence.

#### Table: charge_configs

Active rate values and date ranges per charge type.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Config ID |
| tenant_id | BIGINT FK | Owner tenant |
| charge_type_id | BIGINT FK | Links to charge_types.id |
| value | DECIMAL(12,4) | Rate (amount or percentage) |
| effective_from | DATE | Config start date |
| effective_to | DATE NULL | Config end date (NULL = ongoing) |
| grace_days | INT | Days before late fee applies |
| is_active | BOOLEAN | Active config flag |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: charge_config_mappings

Binds charge configs to specific buildings.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Mapping ID |
| tenant_id | BIGINT FK | Owner tenant |
| charge_config_id | BIGINT FK | Links to charge_configs.id |
| building_id | BIGINT FK | Target building |
| status | VARCHAR(20) | ACTIVE / INACTIVE |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

> **RULE**: One building cannot have two active configs of the same charge_type at the same time.

#### Table: charge_slabs

Tiered volumetric pricing ranges.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Slab ID |
| tenant_id | BIGINT FK | Owner tenant |
| charge_config_id | BIGINT FK | Parent config |
| slab_order | INT | Slab sequence (1, 2, 3...) |
| from_unit | DECIMAL | Range start (inclusive) |
| to_unit | DECIMAL NULL | Range end (NULL = unlimited) |
| rate | DECIMAL | Price per unit in this range |
| is_active | BOOLEAN | Active flag |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

### Domain 8: Billing & Invoicing

#### Table: bills

Master monthly invoice records.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Bill ID |
| tenant_id | BIGINT FK | Owner tenant |
| flat_id | BIGINT FK | Target flat |
| bill_number | VARCHAR(50) | Sequence e.g. GAS-2026-000145 |
| billing_month | DATE | Month being billed e.g. 2026-05-01 |
| total_usage | DECIMAL | Gas consumed (units) |
| total_amount | DECIMAL | Current month charges total |
| previous_due | DECIMAL | Unpaid balance carried from last month |
| due_amount | DECIMAL | Final payable = total_amount + previous_due - credits |
| late_fee_enabled | BOOLEAN | Allow late penalty on this bill |
| late_fee_applied | BOOLEAN | Has penalty been applied? |
| late_fee_applied_at | TIMESTAMP | When penalty was applied |
| grace_days | INT | Grace days before penalty |
| due_date | DATE | Payment deadline |
| status | VARCHAR(20) | UNPAID / PARTIALLY_PAID / PAID |
| created_at | TIMESTAMP | Generation time |
| updated_at | TIMESTAMP | Last update time |

> **UNIQUE**: Composite index on (flat_id, billing_month). One bill per flat per month.

#### Table: bill_items

Itemized audit lines for each charge on a bill.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Item ID |
| tenant_id | BIGINT FK | Owner tenant |
| bill_id | BIGINT FK | Parent bill |
| charge_config_id | BIGINT FK | Config reference for audit trail |
| charge_name | VARCHAR(50) | Label e.g. Gas Slab 1 / VAT |
| charge_type | VARCHAR(20) | UNIT / FIXED / PERCENTAGE |
| base_on | VARCHAR(30) | CONSUMPTION / SUBTOTAL / TOTAL |
| value | DECIMAL(12,4) | Rate configured |
| calculated_amount | DECIMAL(12,2) | Final computed charge |
| created_at | TIMESTAMP | Line creation time |

### Domain 9: Payments & Ledger

#### Table: payments

Customer payment records.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Payment ID |
| tenant_id | BIGINT FK | Owner tenant |
| bill_id | BIGINT FK | Linked bill |
| flat_id | BIGINT FK | Flat ref for fast queries |
| amount | DECIMAL | Amount paid |
| payment_method | VARCHAR(30) | CASH / BKASH / BANK / CARD |
| payment_date | DATE | Transaction date |
| reference_no | VARCHAR(100) | bKash TrxID or receipt number |
| notes | TEXT | Optional memo |
| created_at | TIMESTAMP | Entry time |

#### Table: transactions

Double-entry general ledger entries.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Transaction ID |
| tenant_id | BIGINT FK | Owner tenant |
| reference_id | BIGINT FK | Links to payments.id |
| payment_method | VARCHAR(50) | Payment mode |
| amount | DECIMAL | Amount |
| transaction_date | DATE | Ledger date |
| reference_no | VARCHAR(100) | Reference number |
| created_at | TIMESTAMP | Entry time |
| updated_at | TIMESTAMP | Last update time |

> Every payment auto-creates one transaction record. This is the double-entry bookkeeping pair.

### Domain 10: Inventory & Warehouse

#### Table: suppliers

Hardware and material vendors.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Supplier ID |
| tenant_id | BIGINT FK | Owner tenant |
| name | VARCHAR(150) | Vendor company name |
| contact_person | VARCHAR(100) | Rep name |
| phone | VARCHAR(50) | Phone |
| email | VARCHAR(150) | Email |
| address | TEXT | Address |
| supplier_type | VARCHAR(50) | Gas / Cylinder / Meter / Regulator |
| is_active | BOOLEAN | Active vendor |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: materials

Catalog of purchasable raw materials.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Material ID |
| name | VARCHAR(150) | Item name |
| type | VARCHAR(30) | Category type |
| unit | VARCHAR(20) | kg / pcs / liters |
| is_active | BOOLEAN | Available flag |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: material_purchases

Procurement invoices from suppliers.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Purchase ID |
| material_id | BIGINT FK | Links to materials.id |
| supplier_id | BIGINT FK | Links to suppliers.id |
| purchase_reference | VARCHAR(100) | Supplier invoice number |
| quantity | NUMERIC(10,3) | Units purchased |
| unit_price | NUMERIC(10,2) | Cost per unit |
| additional_charges | NUMERIC(10,2) | Shipping/customs/misc |
| total_price | NUMERIC(10,2) | AUTO: (qty * unit_price) + additional |
| effective_unit_cost | NUMERIC(10,4) | AUTO: total_price / quantity |
| purchase_date | DATE | Purchase date |
| note | TEXT | Optional notes |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: stores

Physical warehouse godowns.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Store ID |
| tenant_id | BIGINT FK | Owner tenant |
| branch_id | BIGINT FK NULL | Optional branch link |
| name | VARCHAR(150) | Godown name |
| address | TEXT | Physical address |
| contact_phone | VARCHAR(50) | Storekeeper phone |
| is_active | BOOLEAN | Operational status |
| created_at | TIMESTAMP | Creation time |

#### Table: inventory_items

Master product catalog for tracked stock.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Item ID |
| tenant_id | BIGINT FK | Owner tenant |
| name | VARCHAR(100) | Product name e.g. 45kg Cylinder |
| category | VARCHAR(50) | Cylinder / Meter / Pipe |
| unit | VARCHAR(20) | pcs / kg |
| is_active | BOOLEAN | Available flag |
| created_at | TIMESTAMP | Creation time |

#### Table: inventory_stock

Real-time stock balance per item per location (polymorphic).

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Stock record ID |
| tenant_id | BIGINT FK | Owner tenant |
| item_id | BIGINT FK | Links to inventory_items.id |
| location_id | BIGINT | Target entity ID (polymorphic) |
| location_type | VARCHAR(20) | STORE / BRANCH / BUILDING |
| current_quantity | DECIMAL(12,3) | Available stock count |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

> **POLYMORPHIC**: location_id references stores.id OR branches.id OR buildings.id based on location_type. Always filter by BOTH location_id AND location_type together.

#### Table: inventory_transactions

Audit log of every stock movement.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Transaction ID |
| tenant_id | BIGINT FK | Owner tenant |
| item_id | BIGINT FK | Which item moved |
| from_location_id | BIGINT | Origin location (polymorphic) |
| to_location_id | BIGINT | Destination location (polymorphic) |
| transaction_type | VARCHAR(20) | IN / OUT / TRANSFER / RETURN |
| quantity | DECIMAL(12,3) | How much moved |
| reference_id | BIGINT NULL | Linked purchase or bill ID |
| transaction_date | DATE | Movement date |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

### Domain 11: Settings

#### Table: billing_settings

Per-building billing cycle configuration.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Settings ID |
| tenant_id | BIGINT FK | Owner tenant |
| building_id | BIGINT FK | Target building |
| billing_day | INT | Day of month bills generate (e.g. 25) |
| due_days | INT | Days allowed to pay (e.g. 15) |
| grace_days | INT | Extra days before late fee |
| round_off_to | INT | Rounding factor for totals |
| currency | VARCHAR(10) | Currency code e.g. BDT |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: sms_settings

SMS gateway credentials.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Settings ID |
| tenant_id | BIGINT FK | Owner tenant |
| provider | VARCHAR(50) | Gateway provider name |
| api_key | VARCHAR(255) | API authentication key |
| sender_id | VARCHAR(50) | Registered sender ID |
| is_active | BOOLEAN | Gateway enabled |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: sms_templates

Reusable message templates with placeholder injection.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Template ID |
| tenant_id | BIGINT FK | Owner tenant |
| template_name | VARCHAR(100) | Template label |
| message_body | JSONB | Message with {placeholders} |
| type | VARCHAR(50) | BillGenerate / Payment / Overdue / MaintenanceAlert |
| is_active | BOOLEAN | Active flag |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: email_settings

SMTP gateway configuration.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Settings ID |
| tenant_id | BIGINT FK | Owner tenant |
| smtp_host | VARCHAR(100) | SMTP server host |
| smtp_port | INT | Connection port |
| smtp_user | VARCHAR(100) | SMTP username |
| smtp_password | VARCHAR(255) | Encrypted SMTP password |
| from_email | VARCHAR(150) | Sender email address |
| from_name | VARCHAR(100) | Sender display name |
| is_active | BOOLEAN | Gateway enabled |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: notification_settings

Channel enable/disable preferences.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Settings ID |
| tenant_id | BIGINT FK | Owner tenant |
| is_email | BOOLEAN | Email alerts enabled |
| is_sms | BOOLEAN | SMS alerts enabled |
| is_inapp | BOOLEAN | In-app alerts enabled |
| low_stock_alert | BOOLEAN | Low inventory alerts enabled |
| overdue_alert_days | INT | Days past due to trigger overdue alert |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

#### Table: general_settings

Key-value store for custom configuration.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Settings ID |
| tenant_id | BIGINT FK | Owner tenant |
| setting_key | VARCHAR(100) | Config key name |
| setting_value | TEXT | Config value |
| created_at | TIMESTAMP | Creation time |
| updated_at | TIMESTAMP | Last update time |

### Domain 12: Audit & Queue

#### Table: audit_logs

Immutable history of all database operations.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Log ID |
| tenant_id | BIGINT FK | Owner tenant |
| user_id | BIGINT FK | Staff who did the action |
| action | VARCHAR(50) | CREATE / UPDATE / DELETE |
| entity_type | VARCHAR(50) | Table name e.g. bills |
| entity_id | BIGINT | Record ID affected |
| description | TEXT | Human-readable summary |
| old_values | JSONB | Snapshot before change |
| new_values | JSONB | Snapshot after change |
| ip_address | VARCHAR(50) | Client IP |
| created_at | TIMESTAMP | Action timestamp |

#### Table: notification_queue

Asynchronous outbound message dispatch queue.

| Column | Type | Purpose |
|---|---|---|
| id | BIGSERIAL PK | Queue entry ID |
| tenant_id | BIGINT FK | Owner tenant |
| user_id | BIGINT FK NULL | Optional recipient user |
| notification_type | VARCHAR(30) | SMS / EMAIL / INAPP |
| recipient | VARCHAR(100) | Phone number or email |
| subject | VARCHAR(150) | Email subject (nullable for SMS) |
| message_body | TEXT | Full message content |
| status | VARCHAR(20) | PENDING / SENT / FAILED / UNREAD / SEEN |
| retry_count | INT | Delivery attempt count |
| scheduled_at | TIMESTAMP | Scheduled send time |
| sent_at | TIMESTAMP | Successful delivery time |
| created_at | TIMESTAMP | Queue entry time |

---

## 6. Charging Engine � Test Cases Reference

From the official test matrix � these are the exact scenarios your billing engine must handle:

| Test ID | Type | Scenario | Logic | Expected Result |
|---|---|---|---|---|
| CHG-001 | UNIT (no slab) | Normal rate | consumption x value | 80 x 150 = 12,000 |
| CHG-002 | UNIT (no slab) | Zero consumption | 0 x value | GasBill = 0 |
| CHG-003 | UNIT (no slab) | Negative consumption | System handles | Set to 0 or COMPLAIN flag |
| CHG-004 | UNIT (slab) | Inside first slab | Apply slab 1 rate | 80 units at slab 1 rate |
| CHG-005 | UNIT (slab) | Spans multiple slabs | Sum all applicable slabs | 180 units = (100x120) + (80x180) = 26,400 |
| CHG-006 | UNIT (slab) | Last unlimited slab | Apply last rate | 350 units at last slab rate |
| CHG-007 | FIXED | Monthly fixed fee | Value as-is | ServiceCharge = 500 |
| CHG-008 | FIXED | Zero consumption | Still charge | Fixed amount always applies |
| CHG-009 | PERCENTAGE on Consumption | % of gas bill | consumption x unitPrice x % | 80 x 150 x 0.08 = 960 |
| CHG-010 | PERCENTAGE on Subtotal | % of unit+service | (GasBill + Service) x % | (12000 + 960) x 0.05 = 648 |
| CHG-011 | PERCENTAGE on Total | % of full total | TotalBeforeVAT x % | 15000 x 0.05 = 750 |
| CHG-012 | Late Fee (%) on Total | Penalty on bill | PreviousTotal x % | 12000 x 0.10 = 1,200 |
| CHG-013 | Late Fee (%) on Subtotal | Penalty on charges | (GasBill+Service) x % | 12960 x 0.10 = 1,296 |
| CHG-014 | Late Fee (Fixed) | Fixed penalty | Fixed value | LateFee = 500 |
| CHG-015 | Minimum Charge | Bill below minimum | Apply minimum | Calculated 450 becomes 800 |
| CHG-016 | Combination | Unit+Service%+VAT | Step-by-step | Full bill calculation |
| CHG-017 | Combination | Slab+Service%+VAT | Slab first then % | Complex tiered + percentage |
| CHG-018 | Edge Case | No charge config | No active configs found | Bill skipped for that flat |
| CHG-019 | Edge Case | Expired config | effective_to passed | Charge not applied |
| CHG-020 | Building-wise | Different rates | Building A vs B | Different values per building |
| CHG-021 | Uniqueness | Duplicate charge type | Same type twice in 1 building | REJECTED � 1 active per type per building |
| CHG-022 | Date Range | Ongoing config | effective_to = NULL | Continues until changed |

---

## 7. Role & Permission Test Cases

| Test ID | Scenario | Expected Result |
|---|---|---|
| RP-001 | Create new role for tenant | Role created successfully |
| RP-002 | Duplicate role name in same tenant | Error: Role name already exists |
| RP-003 | Assign 3 permissions to role | Role shows 3 permissions |
| RP-004 | Remove permission from role | Role updated, permission removed |
| RP-005 | Create user with role | User inherits all role permissions |
| RP-006 | Check permissions_json after role assign | Exact copy of role permissions |
| RP-007 | Add extra permission to user | User has original + extra |
| RP-008 | Remove inherited permission from user | User no longer has that permission |
| RP-009 | Change role permissions after users assigned | Existing users NOT affected |
| RP-010 | Re-assign role to user | User gets new role permissions |
| RP-011 | Tenant A role visible to Tenant B | BLOCKED - tenant isolation enforced |
| RP-012 | Super Admin adds new global permission | Visible to all tenants |
| RP-013 | User without permission tries action | 403 Forbidden |
| RP-015 | Deactivate a role | Users keep their copied permissions |
| RP-016 | User with no role | No permissions - blocked |
| RP-017 | Empty permissions_json | No access except login |

---

## 8. Tech Stack

| Category | Technology | Reason |
|---|---|---|
| Frontend | Next.js 16 App Router | Performance, SEO, mobile |
| Backend | NestJS TypeScript | Modular SaaS architecture |
| Database | PostgreSQL | Reliable, scalable |
| ORM | Prisma | TypeScript support + migrations |
| Auth | JWT + Refresh Token | Stateless, secure |
| Multi-tenancy | Subdomain-based | tenant1.gaspay.cc isolation |
| Validation | class-validator + DTOs | Strict input validation |
| Error Handling | Global Exception Filter | Consistent error responses |
| Response Format | Standard JSON | { success, message, data, meta } |
| Soft Delete | is_deleted flag | All major tables |

---

## 9. NestJS Module List (Implementation Priority)

| Module | Priority | Tables Covered |
|---|---|---|
| AuthModule | HIGH | users, roles, permissions, role_permissions |
| TenantModule | HIGH | tenants, subscription_plans |
| LocationsModule | HIGH | branches, buildings, flats, stores |
| AssociatesModule | HIGH | customers, customer_flat_mappings, employees, employee_unit_mappings |
| MeterReadingsModule | HIGH | meters, meter_readings |
| ChargesModule | HIGH | charge_types, charge_configs, charge_slabs, charge_config_mappings |
| BillingModule | HIGH | bills, bill_items, billing_settings |
| PaymentsModule | HIGH | payments, transactions |
| InventoryModule | MEDIUM | suppliers, materials, material_purchases, inventory_items, inventory_stock, inventory_transactions |
| SMSModule | HIGH | sms_settings, sms_templates |
| NotificationsModule | MEDIUM | notification_settings, notification_queue |
| SettingsModule | HIGH | email_settings, general_settings |
| AuditModule | MEDIUM | audit_logs |

---

## 10. Additional Postman Routes (Missing from Section 4)

### Billing Settings

#### GET /api/v1/settings/billing

Fetch billing settings for a building.

- Query: ?buildingId=18
- Response: { billingDay, dueDays, graceDays, roundOffTo, currency }

#### POST /api/v1/settings/billing

Configure billing cycle for a building.

- Body: { buildingId, billingDay, dueDays, graceDays, roundOffTo, currency }

#### PATCH /api/v1/settings/billing/:id

Update existing billing settings.

### General Settings

#### GET /api/v1/settings/general

List all key-value general settings.

#### POST /api/v1/settings/general

Create a new setting entry.

- Body: { settingKey, settingValue }

#### PATCH /api/v1/settings/general/:id

Update a general setting.

### Notification Settings

#### GET /api/v1/settings/notifications

Fetch current notification channel preferences.

- Response: { isEmail, isSms, isInapp, lowStockAlert, overdueAlertDays }

#### PATCH /api/v1/settings/notifications

Update notification preferences.

- Body: { isEmail, isSms, isInapp, lowStockAlert, overdueAlertDays }

### Branches - Full CRUD

#### GET /api/v1/branches/:id

Get a single branch by ID.

#### PATCH /api/v1/branches/:id

Update branch details.

- Body: { name, address, phone, status }

#### DELETE /api/v1/branches/:id

Soft delete a branch (sets is_deleted=true).

### Buildings - Full CRUD

#### GET /api/v1/buildings/:id

Get building details including flat count.

#### PATCH /api/v1/buildings/:id

Update building info.

- Body: { name, area, address, totalFloors, totalFlats }

#### DELETE /api/v1/buildings/:id

Soft delete a building.

### Flats - Full CRUD

#### GET /api/v1/flats/:id

Get flat details including current occupant.

#### PATCH /api/v1/flats/:id

Update flat information.

- Body: { unitNumber, floor, sizeSqft, status }

#### DELETE /api/v1/flats/:id

Soft delete a flat.

### Customers - Full CRUD

#### GET /api/v1/customers/:id

Get customer profile with flat history.

#### PATCH /api/v1/customers/:id

Update customer details.

- Body: { name, phone, email, status }

#### DELETE /api/v1/customers/:id

Soft delete customer profile.

#### GET /api/v1/customers/:id/bills

Fetch all bills associated with a customer.

- Response: list of bills across all flats

#### GET /api/v1/customers/flat-mappings

List all active customer-flat mappings.

- Query: ?flatId=104 OR ?customerId=89

### Users - Full CRUD

#### POST /api/v1/users

Create a new staff user account.

- Body: { name, email, password, roleId }

#### GET /api/v1/users/:id

Get a specific user profile.

#### PATCH /api/v1/users/:id

Update user details.

- Body: { name, email, isActive }

#### DELETE /api/v1/users/:id

Deactivate a user account.

#### PATCH /api/v1/users/:id/role

Change a user role (re-copies permissions).

- Body: { roleId }

### Roles - Full CRUD

#### GET /api/v1/roles/:id

Get role with its permission list.

#### PATCH /api/v1/roles/:id

Update role name.

- Body: { name }

#### DELETE /api/v1/roles/:id

Delete a role (only if no active users assigned).

#### POST /api/v1/roles/:id/permissions

Assign permissions to a role.

- Body: { permissionIds: [1, 2, 3] }

#### DELETE /api/v1/roles/:id/permissions/:permissionId

Remove a specific permission from a role.

### Meters - Full CRUD

#### GET /api/v1/meters/:id

Get meter details with installation history.

#### PATCH /api/v1/meters/:id

Update meter status or details.

- Body: { status, meterNumber }

#### DELETE /api/v1/meters/:id

Soft delete / decommission a meter.

### Employees - Full CRUD

#### GET /api/v1/employees/:id

Get employee with unit assignments.

#### PATCH /api/v1/employees/:id

Update employee details.

- Body: { name, phone, email, designation, isActive }

#### DELETE /api/v1/employees/:id

Soft delete employee record.

#### DELETE /api/v1/employees/:id/unit-mappings/:mappingId

Remove an employee unit assignment.

### Charge Types & Configs - Full CRUD

#### GET /api/v1/charge-types/:id

Get charge type details.

#### PATCH /api/v1/charge-types/:id

Update charge type.

- Body: { name, chargeMode, applyOn, orderNo, isActive }

#### DELETE /api/v1/charge-types/:id

Delete a charge type.

#### GET /api/v1/charge-configs/:id

Get charge configuration details.

#### PATCH /api/v1/charge-configs/:id

Update a configuration.

- Body: { value, effectiveFrom, effectiveTo, graceDays, isActive }

#### DELETE /api/v1/charge-configs/:id

Remove a charge configuration.

#### DELETE /api/v1/charge-configs/mappings/:id

Remove a building-config binding.

#### PATCH /api/v1/charge-configs/:id/slabs

Replace all slabs for a configuration.

- Body: { slabs: [{ slabOrder, fromUnit, toUnit, rate }] }

### Suppliers - Full CRUD

#### GET /api/v1/inventory/suppliers/:id

Get supplier details.

#### PATCH /api/v1/inventory/suppliers/:id

Update supplier.

- Body: { name, contactPerson, phone, email, address, supplierType, isActive }

#### DELETE /api/v1/inventory/suppliers/:id

Deactivate supplier.

### Materials - Full CRUD

#### GET /api/v1/inventory/materials/:id

Get material details.

#### PATCH /api/v1/inventory/materials/:id

Update material.

- Body: { name, type, unit, isActive }

#### GET /api/v1/inventory/materials/purchases/:id

Get purchase invoice details.

### Inventory Items - Full CRUD

#### GET /api/v1/inventory/items/:id

Get item details.

#### PATCH /api/v1/inventory/items/:id

Update item.

- Body: { name, category, unit, isActive }

#### DELETE /api/v1/inventory/items/:id

Deactivate item.

### Stores - Full CRUD

#### GET /api/v1/inventory/stores/:id

Get store details.

#### PATCH /api/v1/inventory/stores/:id

Update store.

- Body: { name, address, contactPhone, isActive }

#### GET /api/v1/inventory/transactions?page=1&limit=9&sortBy=transactionDate&sortOrder=desc&itemId=8&transactionType=TRANSFER&startDate=2026-05-01&endDate=2026-05-19

List all stock movement history.

- Query: ?itemId=8&transactionType=TRANSFER

### Subscription Plans

#### GET /api/v1/subscription-plans

List all available plans.

#### POST /api/v1/subscription-plans

Create a new plan (Super Admin only).

- Body: { name, priceMonthly, priceYearly, features, maxBranch, maxBuilding, maxFlats }

---

## 11. Complete Flow-Wise Implementation Sequence

This is the exact build order. Each step depends on the previous.

### PHASE 1: Foundation (Week 1)

**Step 1.1 � NestJS Project Bootstrap**

- Run: npx @nestjs/cli new gaspay-backend
- Install: prisma, @nestjs/jwt, @nestjs/passport, class-validator, class-transformer
- Configure: .env with DATABASE_URL, JWT_SECRET, JWT_REFRESH_SECRET

**Step 1.2 � Prisma Schema & Migrations**

- Convert Gaspay.sql to schema.prisma
- Define all 38 models with proper relations and @@index annotations
- Run: npx prisma migrate dev --name init
- Run: npx prisma generate
- Tables to create first: subscription_plans, tenants

**Step 1.3 � Subdomain Multi-Tenant Middleware**

- Create TenantMiddleware that extracts subdomain from request host
- Query tenants table for matching subdomain
- Inject tenantId into request context (req.tenantId)
- Apply globally in AppModule
- Test: Host header dhaka-gas.gaspay.cc resolves tenant correctly

**Step 1.4 � Auth Module (JWT + RBAC)**

- Create tables: roles, permissions, role_permissions, users
- Implement: POST /auth/register (creates tenant + admin user)
- Implement: POST /auth/login (returns accessToken + refreshToken)
- Implement: POST /auth/refresh (rotates access token)
- Implement JwtAuthGuard and PermissionsGuard
- On user creation: copy role permissions into permissions_json
- On login: include permissions_json in JWT payload
- Test: Login returns token, token guards routes, 403 on missing permission

---

### PHASE 2: Property & People (Week 2)

**Step 2.1 � Locations Module**

- Tables: branches, buildings, flats
- CRUD for all three
- On flat/building creation: check subscription plan limits (max_branch, max_building, max_flats)
- On flat creation: default status = VACANT
- Soft delete pattern: set is_deleted=true, filter in all queries
- Test: Create branch > building > flat hierarchy

**Step 2.2 � Customers Module**

- Tables: customers, customer_flat_mappings
- POST /customers � register customer
- POST /customers/flat-mappings � assign to flat
  - Validate: flat has no active mapping (is_active=true)
  - Set flat.status = OCCUPIED inside DB transaction
  - Set is_primary=true if first mapping
- PATCH /customers/flat-mappings/:id/deactivate
  - Set is_active=false, set end_date
  - Set flat.status = VACANT
- Test: Cannot assign two active customers to same flat

**Step 2.3 � Employees Module**

- Tables: employees, employee_unit_mappings
- CRUD for employees
- POST /employees/:id/unit-mappings � assign to BRANCH/BUILDING/STORE
- Test: Employee mapped to building, only sees that building's flats

---

### PHASE 3: Hardware (Week 3)

**Step 3.1 � Meters Module**

- Table: meters
- POST /meters � install meter in flat
  - Validate: flat has no active meter already
  - Unique index on flat_id
- Test: Cannot install second meter in same flat

**Step 3.2 � Meter Readings Module**

- Table: meter_readings
- POST /meter-readings � log monthly reading
  - Auto-populate previous_reading from last reading for this meter
  - Validate: current_reading >= previous_reading
  - Calculate: consumption = current_reading - previous_reading
  - Set status: AVERAGE / HIGH / COMPLAIN based on thresholds
  - Unique index on (meter_id, reading_month) � one per month
- GET /meter-readings?buildingId=18&readingMonth=2026-05-01
- Test: Duplicate month reading rejected, lower current rejected

---

### PHASE 4: Pricing Engine (Week 4)

**Step 4.1 � Charge Types**

- Table: charge_types
- CRUD with order_no control
- order_no determines calculation sequence

**Step 4.2 � Charge Configs**

- Table: charge_configs
- CRUD with effective date range
- Validation: effective_to > effective_from
- is_active management

**Step 4.3 � Charge Slabs**

- Table: charge_slabs
- POST /charge-configs/:id/slabs � bulk upsert slab ranges
- Validate: slabs cover contiguous ranges, no gaps
- Last slab to_unit = NULL (unlimited)

**Step 4.4 � Charge Config Mappings**

- Table: charge_config_mappings
- POST /charge-configs/mappings � bind config to building
- Validation: building cannot have two active configs of same charge_type
- Test: Overlapping date ranges for same charge type on same building rejected

---

### PHASE 5: Billing Engine (Week 5)

**Step 5.1 � Billing Preview (in-memory)**

- No DB writes
- POST /billing/preview � { flatId, billingMonth }
- Fetch: active meter reading for month, active charge configs for building
- Calculate in order_no sequence:
  1. UNIT charges (with or without slabs)
  2. FIXED charges
  3. PERCENTAGE charges (applied on CONSUMPTION, SUBTOTAL, or TOTAL)
- Fetch previous_due from last unpaid bill
- Fetch customer credit balance
- Return itemized preview

**Step 5.2 � Billing Settings**

- Table: billing_settings
- Configure per-building: billing_day, due_days, grace_days, currency

**Step 5.3 � Bulk Bill Generation**

- Tables: bills, bill_items
- POST /billing/generate � { buildingId, billingMonth }
- For each occupied flat in building:
  - Run preview calculation
  - Check for previous unpaid bill � add to previous_due
  - Apply customer credit balance � deduct from due_amount
  - Generate sequential bill_number (GAS-YYYY-XXXXXX)
  - Create bill record + bill_items in single DB transaction
  - Add notification_queue entry for SMS alert
- Test: Bills generated, previous dues carried, credits applied

**Step 5.4 � Bill Management**

- GET /billing/bills � list with filters
- GET /billing/bills/:id � full detail with bill_items
- PATCH /billing/bills/:id/correct � Super Admin recalculate
- DELETE /billing/bills/:id � soft delete

---

### PHASE 6: Payments (Week 6)

**Step 6.1 � Manual Payments**

- Tables: payments, transactions
- POST /payments/manual � { billId, amount, paymentMethod, paymentDate }
- Logic:
  - if amount == due_amount: bill.status = PAID, due_amount = 0
  - if amount < due_amount: bill.status = PARTIALLY_PAID, reduce due_amount
  - if amount > due_amount: bill.status = PAID, excess = customer credit balance
- Auto-create transaction record (double-entry)
- Test: Overpayment creates credit balance

**Step 6.2 � bKash Gateway**

- POST /payments/bkash/initiate � call bKash API, return redirect URL
- POST /payments/bkash/callback � verify webhook, execute payment logic
- Wrap in DB transaction for atomicity

---

### PHASE 7: Inventory (Week 7)

**Step 7.1 � Suppliers & Materials**

- CRUD for suppliers and materials
- POST /inventory/materials/purchases
  - Auto-calculate: total_price = (qty * unit_price) + additional_charges
  - Auto-calculate: effective_unit_cost = total_price / qty

**Step 7.2 � Stores & Stock**

- CRUD for stores and inventory_items
- POST /inventory/stock/transfer
  - Validate: from_location has sufficient stock
  - In single DB transaction: decrease from_location, increase to_location
  - Create inventory_transaction record
- GET /inventory/stock?locationType=STORE&locationId=2

---

### PHASE 8: Notifications & Workers (Week 8)

**Step 8.1 � Gateway Settings**

- CRUD for sms_settings, email_settings, sms_templates
- Template placeholder system: {customerName}, {billNumber}, {dueAmount}

**Step 8.2 � Notification Queue Worker**

- Background worker polls notification_queue WHERE status=PENDING
- Process template variables with actual values
- Call SMS/Email gateway API
- Update status: SENT or FAILED (increment retry_count)
- POST /notifications/queue/retry � manual retry

**Step 8.3 � Late Fee Cron Job**

- @nestjs/schedule daily cron
- Query bills WHERE status=UNPAID AND due_date < TODAY - grace_days AND late_fee_applied=false
- Apply late fee charge (FIXED or PERCENTAGE based on charge_type config)
- Set late_fee_applied=true, late_fee_applied_at=NOW()

---

### PHASE 9: Audit & Polish (Week 9)

**Step 9.1 � Audit Logging**

- Prisma middleware or NestJS interceptor
- Auto-capture: action, entity_type, entity_id, old_values, new_values, user_id, ip_address
- GET /audit-logs with filters and pagination

**Step 9.2 � Reporting Routes**

- GET /reports/collections?buildingId=18&month=2026-05
- GET /reports/outstanding � list all unpaid bills
- GET /reports/inventory-summary � stock levels across all locations

---

## 12. Complete Route Quick-Reference Index

### Auth & Tenants

- POST /api/v1/auth/register
- POST /api/v1/auth/login
- POST /api/v1/auth/refresh
- GET  /api/v1/tenant/profile
- PATCH  /api/v1/tenant/profile

### Users, Roles, Permissions

- GET  /api/v1/users
- POST /api/v1/users
- GET  /api/v1/users/:id
- PATCH  /api/v1/users/:id
- DELETE /api/v1/users/:id
- PATCH  /api/v1/users/:id/role
- PATCH  /api/v1/users/:id/permissions
- GET  /api/v1/roles
- POST /api/v1/roles
- GET  /api/v1/roles/:id
- PATCH  /api/v1/roles/:id
- DELETE /api/v1/roles/:id
- POST /api/v1/roles/:id/permissions
- DELETE /api/v1/roles/:id/permissions/:permissionId
- GET  /api/v1/permissions

### Property Hierarchy

- GET  /api/v1/branches
- POST /api/v1/branches
- GET  /api/v1/branches/:id
- PATCH  /api/v1/branches/:id
- DELETE /api/v1/branches/:id
- GET  /api/v1/buildings
- POST /api/v1/buildings
- GET  /api/v1/buildings/:id
- PATCH  /api/v1/buildings/:id
- DELETE /api/v1/buildings/:id
- GET  /api/v1/flats
- POST /api/v1/flats
- GET  /api/v1/flats/:id
- PATCH  /api/v1/flats/:id
- DELETE /api/v1/flats/:id

### Customers

- GET  /api/v1/customers
- POST /api/v1/customers
- GET  /api/v1/customers/:id
- PATCH  /api/v1/customers/:id
- DELETE /api/v1/customers/:id
- GET  /api/v1/customers/:id/bills
- GET  /api/v1/customers/flat-mappings
- POST /api/v1/customers/flat-mappings
- PATCH  /api/v1/customers/flat-mappings/:id/deactivate

### Employees

- GET  /api/v1/employees
- POST /api/v1/employees
- GET  /api/v1/employees/:id
- PATCH  /api/v1/employees/:id
- DELETE /api/v1/employees/:id
- GET  /api/v1/employees/:id/unit-mappings
- POST /api/v1/employees/:id/unit-mappings
- DELETE /api/v1/employees/:id/unit-mappings/:mappingId

### Meters & Readings

- GET  /api/v1/meters
- POST /api/v1/meters
- GET  /api/v1/meters/:id
- PATCH  /api/v1/meters/:id
- DELETE /api/v1/meters/:id
- GET  /api/v1/meter-readings
- POST /api/v1/meter-readings
- GET  /api/v1/meter-readings/:id
- PATCH  /api/v1/meter-readings/:id

### Charges & Pricing

- GET  /api/v1/charge-types
- POST /api/v1/charge-types
- GET  /api/v1/charge-types/:id
- PATCH  /api/v1/charge-types/:id
- DELETE /api/v1/charge-types/:id
- GET  /api/v1/charge-configs
- POST /api/v1/charge-configs
- GET  /api/v1/charge-configs/:id
- PATCH  /api/v1/charge-configs/:id
- DELETE /api/v1/charge-configs/:id
- GET  /api/v1/charge-configs/:id/slabs
- POST /api/v1/charge-configs/:id/slabs
- PATCH  /api/v1/charge-configs/:id/slabs
- GET  /api/v1/charge-configs/mappings
- POST /api/v1/charge-configs/mappings
- DELETE /api/v1/charge-configs/mappings/:id

### Billing

- POST /api/v1/billing/preview
- POST /api/v1/billing/generate
- GET  /api/v1/billing/bills
- GET  /api/v1/billing/bills/:id
- PATCH  /api/v1/billing/bills/:id/correct
- DELETE /api/v1/billing/bills/:id

### Payments & Ledger

- GET  /api/v1/payments
- POST /api/v1/payments/manual
- POST /api/v1/payments/bkash/initiate
- POST /api/v1/payments/bkash/callback
- GET  /api/v1/ledger/transactions

### Inventory

- GET  /api/v1/inventory/suppliers
- POST /api/v1/inventory/suppliers
- GET  /api/v1/inventory/suppliers/:id
- PATCH  /api/v1/inventory/suppliers/:id
- DELETE /api/v1/inventory/suppliers/:id
- GET  /api/v1/inventory/materials
- POST /api/v1/inventory/materials
- GET  /api/v1/inventory/materials/:id
- PATCH  /api/v1/inventory/materials/:id
- GET  /api/v1/inventory/materials/purchases
- POST /api/v1/inventory/materials/purchases
- GET  /api/v1/inventory/materials/purchases/:id
- GET  /api/v1/inventory/stores
- POST /api/v1/inventory/stores
- GET  /api/v1/inventory/stores/:id
- PATCH  /api/v1/inventory/stores/:id
- GET  /api/v1/inventory/items
- POST /api/v1/inventory/items
- GET  /api/v1/inventory/items/:id
- PATCH  /api/v1/inventory/items/:id
- GET  /api/v1/inventory/stock
- POST /api/v1/inventory/stock/transfer
- GET  /api/v1/inventory/transactions

### Settings

- GET  /api/v1/settings/billing
- POST /api/v1/settings/billing
- PATCH  /api/v1/settings/billing/:id
- GET  /api/v1/settings/sms-gateway
- POST /api/v1/settings/sms-gateway
- GET  /api/v1/settings/email-gateway
- POST /api/v1/settings/email-gateway
- GET  /api/v1/settings/sms-templates
- POST /api/v1/settings/sms-templates
- PATCH  /api/v1/settings/sms-templates/:id
- DELETE /api/v1/settings/sms-templates/:id
- GET  /api/v1/settings/notifications
- PATCH  /api/v1/settings/notifications
- GET  /api/v1/settings/general
- POST /api/v1/settings/general
- PATCH  /api/v1/settings/general/:id

### Notifications & Audit

- GET  /api/v1/notifications/queue
- POST /api/v1/notifications/queue/retry
- GET  /api/v1/audit-logs

### Subscription Plans

- GET  /api/v1/subscription-plans
- POST /api/v1/subscription-plans

---

## 13. Detailed Business Scenario Walkthroughs

### Scenario A: New Tenant Onboarding (End-to-End)

**Goal**: A gas company "Dhaka Metro Gas Ltd" signs up and becomes operational in 10 steps.

**Step 1 � Tenant Registration**

```
POST /api/v1/auth/register
Body: { companyName, contactPerson, phone, email, password, subscriptionPlanId }
Result: Tenant record created. Admin user created. permissions_json populated from role.
DB writes: subscription_plans (read) -> tenants (insert) -> roles (insert Admin role) -> users (insert)
```

**Step 2 � Login & Get Token**

```
POST /api/v1/auth/login
Body: { email, password }
Result: JWT token containing tenantId + permissions_json. Store token for all future requests.
```

**Step 3 � Create Branch**

```
POST /api/v1/branches
Cookies: accessToken=<token> (HTTP-only, secure)
Body: { name: "Uttara Zone", address: "...", phone: "..." }
Result: Branch created. Check: tenant.subscription_plan.max_branch not exceeded.
DB writes: branches (insert)
```

**Step 4 � Create Building under Branch**

```
POST /api/v1/buildings
Body: { branchId: 1, name: "Rose Valley Heights", area: "Uttara Sector 3", address: "...", totalFloors: 10, totalFlats: 40 }
Result: Building created. Check: max_building limit.
DB writes: buildings (insert)
```

**Step 5 � Create Flats inside Building**

```
POST /api/v1/flats (repeat per flat)
Body: { buildingId: 1, unitNumber: "Flat 4A", floor: 4, sizeSqft: 1450 }
Result: Flat created with status=VACANT. Check: max_flats limit.
DB writes: flats (insert, status=VACANT)
```

**Step 6 � Configure Charges for Building**

```
POST /api/v1/charge-types   -> { name: "Gas Slabs", chargeMode: "UNIT", applyOn: "CONSUMPTION", isSlab: true, orderNo: 1 }
POST /api/v1/charge-configs -> { chargeTypeId: 1, value: 0, effectiveFrom: "2026-05-01" }
POST /api/v1/charge-configs/1/slabs -> { slabs: [{ slabOrder:1, fromUnit:0, toUnit:100, rate:120 }, { slabOrder:2, fromUnit:101, toUnit:null, rate:180 }] }
POST /api/v1/charge-types   -> { name: "Service Fee", chargeMode: "FIXED", applyOn: "CONSUMPTION", orderNo: 2 }
POST /api/v1/charge-configs -> { chargeTypeId: 2, value: 500, effectiveFrom: "2026-05-01" }
POST /api/v1/charge-types   -> { name: "VAT", chargeMode: "PERCENTAGE", applyOn: "SUBTOTAL", orderNo: 3 }
POST /api/v1/charge-configs -> { chargeTypeId: 3, value: 5, effectiveFrom: "2026-05-01" }
POST /api/v1/charge-configs/mappings -> { chargeConfigId: 1, buildingId: 1 } (repeat for config 2 and 3)
```

**Step 7 � Register Customer & Assign to Flat**

```
POST /api/v1/customers -> { name: "Mahbubur Rahman", phone: "+8801911223344", email: "..." }
POST /api/v1/customers/flat-mappings -> { customerId: 1, flatId: 1, ownershipType: "RENTER", startDate: "2026-05-01", isPrimary: true }
Result: Flat status updated VACANT -> OCCUPIED inside DB transaction.
```

**Step 8 � Install Meter**

```
POST /api/v1/meters -> { flatId: 1, meterNumber: "MET-77890", installationDate: "2026-05-01", status: "ACTIVE" }
Validation: Flat cannot have two active meters (unique index on flat_id).
```

**Step 9 � Log Monthly Reading**

```
POST /api/v1/meter-readings -> { meterId: 1, readingMonth: "2026-05-01", currentReading: 180.50, readingDate: "2026-05-19" }
Result: previous_reading auto-fetched from last reading. consumption = 180.50 - 100.00 = 80.50.
Validation: currentReading >= previousReading. Unique per (meterId, readingMonth).
```

**Step 10 � Generate Bill & Collect Payment**

```
POST /api/v1/billing/preview -> { flatId: 1, billingMonth: "2026-05-01" }   (verify calculation first)
POST /api/v1/billing/generate -> { buildingId: 1, billingMonth: "2026-05-01" }   (generate for all flats)
POST /api/v1/payments/manual -> { billId: 1, amount: 11805, paymentMethod: "CASH", paymentDate: "2026-05-19" }
Result: Bill status UNPAID -> PAID. Transaction ledger record auto-created.
```

---

### Scenario B: Slab Calculation Walkthrough (80 units consumed)

**Active Slabs**: Slab 1: 0-100 units @ BDT 120/unit | Slab 2: 101+ units @ BDT 180/unit
**Consumption**: 80 units (falls entirely in Slab 1)

```
Gas Fee    = 80 units * 120 = BDT 9,600
Service    = BDT 500 (FIXED)
Subtotal   = 9,600 + 500 = BDT 10,100
VAT (5%)   = 10,100 * 0.05 = BDT 505
Total      = 10,100 + 505 = BDT 10,605
Prev Due   = BDT 1,200 (unpaid last month)
DUE AMOUNT = 10,605 + 1,200 = BDT 11,805
```

**DB writes**: bills { total_usage:80, total_amount:10605, previous_due:1200, due_amount:11805, status:UNPAID }
**bill_items**: 3 rows � one per charge type

---

### Scenario C: Multi-Slab Calculation (180 units consumed)

**Active Slabs**: Slab 1: 0-100 @ 120 | Slab 2: 101+ @ 180

```
Slab 1 = 100 units * 120 = BDT 12,000
Slab 2 = 80 units * 180  = BDT 14,400
Gas Fee = BDT 26,400
Service = BDT 500
Subtotal = 26,900
VAT 5%  = 1,345
Total   = BDT 28,245
```

---

### Scenario D: Overpayment & Credit Balance

**Bill due**: BDT 11,805 | **Customer pays**: BDT 12,000

```
Payment  = 12,000
Due      = 11,805
Excess   = 12,000 - 11,805 = BDT 195

DB writes:
  payments { amount: 12000, payment_method: CASH }
  transactions { amount: 12000 }  (double-entry ledger)
  bills { status: PAID, due_amount: 0 }
  customers { credit_balance += 195 }  (applied to next month bill)
```

---

### Scenario E: Late Fee Application (Nightly Cron)

**Config**: Late fee = 10% of TOTAL | Grace days = 5

```
Bill due_date = 2026-05-25
Today        = 2026-06-01 (past due_date + grace_days)
Cron queries: WHERE status=UNPAID AND due_date < (NOW() - grace_days) AND late_fee_applied=false

Late Fee = 11,805 * 0.10 = BDT 1,180.50
due_amount += 1,180.50
bills { late_fee_applied: true, late_fee_applied_at: NOW() }
```

---

### Scenario F: Stock Transfer Flow

**Goal**: Transfer 30 cylinders from Central Godown to Rose Valley Building

```
Step 1: Check stock at Central Godown
GET /api/v1/inventory/stock?locationType=STORE&locationId=1
Result: { currentQuantity: 150 }  -> sufficient

Step 2: Execute transfer (single DB transaction)
POST /api/v1/inventory/stock/transfer
Body: { itemId: 1, fromLocationId: 1, toLocationId: 1, transactionType: "TRANSFER", quantity: 30 }

DB writes (atomic transaction block):
  inventory_stock (Central Godown) { current_quantity: 150 -> 120 }
  inventory_stock (Rose Valley Bldg) { current_quantity: 0 -> 30 }
  inventory_transactions { transaction_type: TRANSFER, quantity: 30 }
```

---

## 14. Complete Table Foreign Key Connection Map

Every FK relationship, direction, and cascade behavior.

```
subscription_plans (1) ------------------> (many) tenants
  tenants.subscription_plan_id -> subscription_plans.id  [RESTRICT on delete]

tenants (1) -----------------------------> (many) roles
  roles.tenant_id -> tenants.id  [CASCADE on delete]

tenants (1) -----------------------------> (many) role_permissions
  role_permissions.tenant_id -> tenants.id  [CASCADE]

roles (1) -------------------------------> (many) role_permissions
  role_permissions.role_id -> roles.id  [CASCADE]

permissions (1) -------------------------> (many) role_permissions
  role_permissions.permission_id -> permissions.id  [CASCADE]
  UNIQUE INDEX: (role_id, permission_id)

tenants (1) -----------------------------> (many) users
  users.tenant_id -> tenants.id  [CASCADE]

roles (1) -------------------------------> (many) users
  users.role_id -> roles.id  [SET NULL on delete]

users (1) -------------------------------> (many) audit_logs
  audit_logs.user_id -> users.id  [RESTRICT]

tenants (1) -----------------------------> (many) branches
  branches.tenant_id -> tenants.id  [CASCADE]

branches (1) ----------------------------> (many) buildings
  buildings.branch_id -> branches.id  [SET NULL � nullable]

tenants (1) -----------------------------> (many) buildings
  buildings.tenant_id -> tenants.id  [CASCADE]

buildings (1) ---------------------------> (many) flats
  flats.building_id -> buildings.id  [CASCADE]
  INDEX on flats.building_id

branches (1) ----------------------------> (many) flats
  flats.branch_id -> branches.id  [SET NULL � nullable]

tenants (1) -----------------------------> (many) customers
  customers.tenant_id -> tenants.id  [CASCADE]

customers (1) ---------------------------> (many) customer_flat_mappings
  customer_flat_mappings.customer_id -> customers.id  [CASCADE]

flats (1) -------------------------------> (many) customer_flat_mappings
  customer_flat_mappings.flat_id -> flats.id  [CASCADE]
  UNIQUE PARTIAL INDEX: (flat_id) WHERE is_active = true

tenants (1) -----------------------------> (many) employees
  employees.tenant_id -> tenants.id  [CASCADE]

branches (1) ----------------------------> (many) employees
  employees.branch_id -> branches.id  [SET NULL � nullable]

buildings (1) ---------------------------> (many) employees
  employees.building_id -> buildings.id  [SET NULL � nullable]

employees (1) ---------------------------> (many) employee_unit_mappings
  employee_unit_mappings.employee_id -> employees.id  [CASCADE]
  unit_id is POLYMORPHIC (branches/buildings/stores)

flats (1) -------------------------------> (many) meters
  meters.flat_id -> flats.id  [RESTRICT � cannot delete flat with meter]
  UNIQUE INDEX: (flat_id)  � one meter per flat

tenants (1) -----------------------------> (many) meters
  meters.tenant_id -> tenants.id  [CASCADE]

meters (1) ------------------------------> (many) meter_readings
  meter_readings.meter_id -> meters.id  [CASCADE]
  UNIQUE INDEX: (meter_id, reading_month)

tenants (1) -----------------------------> (many) charge_types
  charge_types.tenant_id -> tenants.id  [CASCADE]

charge_types (1) ------------------------> (many) charge_configs
  charge_configs.charge_type_id -> charge_types.id  [CASCADE]

charge_configs (1) ----------------------> (many) charge_slabs
  charge_slabs.charge_config_id -> charge_configs.id  [CASCADE]

charge_configs (1) ----------------------> (many) charge_config_mappings
  charge_config_mappings.charge_config_id -> charge_configs.id  [CASCADE]

buildings (1) ---------------------------> (many) charge_config_mappings
  charge_config_mappings.building_id -> buildings.id  [CASCADE]
  UNIQUE INDEX: (building_id, charge_type) WHERE status=ACTIVE

flats (1) -------------------------------> (many) bills
  bills.flat_id -> flats.id  [RESTRICT � cannot delete flat with bills]
  UNIQUE INDEX: (flat_id, billing_month)

tenants (1) -----------------------------> (many) bills
  bills.tenant_id -> tenants.id  [CASCADE]

bills (1) -------------------------------> (many) bill_items
  bill_items.bill_id -> bills.id  [CASCADE]

charge_configs (1) ----------------------> (many) bill_items
  bill_items.charge_config_id -> charge_configs.id  [RESTRICT � audit trail]

bills (1) -------------------------------> (many) payments
  payments.bill_id -> bills.id  [RESTRICT]

flats (1) -------------------------------> (many) payments
  payments.flat_id -> flats.id  [RESTRICT]

payments (1) ----------------------------> (many) transactions
  transactions.reference_id -> payments.id  [CASCADE]

tenants (1) -----------------------------> (many) suppliers
  suppliers.tenant_id -> tenants.id  [CASCADE]

materials (1) ---------------------------> (many) material_purchases
  material_purchases.material_id -> materials.id  [RESTRICT]

suppliers (1) ---------------------------> (many) material_purchases
  material_purchases.supplier_id -> suppliers.id  [RESTRICT]

tenants (1) -----------------------------> (many) stores
  stores.tenant_id -> tenants.id  [CASCADE]

branches (1) ----------------------------> (many) stores
  stores.branch_id -> branches.id  [SET NULL � nullable]

tenants (1) -----------------------------> (many) inventory_items
  inventory_items.tenant_id -> tenants.id  [CASCADE]

inventory_items (1) ---------------------> (many) inventory_stock
  inventory_stock.item_id -> inventory_items.id  [RESTRICT]
  location_id POLYMORPHIC: stores.id | branches.id | buildings.id

inventory_items (1) ---------------------> (many) inventory_transactions
  inventory_transactions.item_id -> inventory_items.id  [RESTRICT]

tenants (1) -----------------------------> (many) billing_settings
  billing_settings.tenant_id -> tenants.id  [CASCADE]

buildings (1) ---------------------------> (many) billing_settings
  billing_settings.building_id -> buildings.id  [CASCADE]
  UNIQUE INDEX: (building_id) � one settings per building

tenants (1) -----------------------------> (many) sms_settings, email_settings,
                                            notification_settings, general_settings,
                                            sms_templates, audit_logs, notification_queue
  All: tenant_id -> tenants.id  [CASCADE]
```

---

## 15. Full Postman Route Payloads (Section 10 Routes - Missing Payloads)

### 15.1 Billing Settings

#### GET /api/v1/settings/billing?buildingId=18

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "1",
    "buildingId": "18",
    "billingDay": 25,
    "dueDays": 15,
    "graceDays": 5,
    "roundOffTo": 2,
    "currency": "BDT"
  }
}
```

#### POST /api/v1/settings/billing

```json
Request:
{ "buildingId": 18, "billingDay": 25, "dueDays": 15, "graceDays": 5, "roundOffTo": 2, "currency": "BDT" }

Response 201:
{ "success": true, "message": "Billing settings saved.", "data": { "id": "1", "buildingId": "18" } }
```

#### PATCH /api/v1/settings/billing/:id

```json
Request:  { "billingDay": 28, "dueDays": 20, "graceDays": 7 }
Response: { "success": true, "message": "Billing settings updated.", "data": { "id": "1" } }
```

### 15.2 General Settings

#### GET /api/v1/settings/general

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 3
  },
  "data": [
    {
      "id": "1",
      "settingKey": "company_logo_url",
      "settingValue": "https://cdn.gaspay.cc/logo.png"
    },
    {
      "id": "2",
      "settingKey": "default_currency",
      "settingValue": "BDT"
    },
    {
      "id": "3",
      "settingKey": "invoice_prefix",
      "settingValue": "GAS"
    }
  ]
}
```

#### POST /api/v1/settings/general

```json
Request:  { "settingKey": "invoice_prefix", "settingValue": "GAS" }
Response: { "success": true, "data": { "id": "3", "settingKey": "invoice_prefix" } }
```

#### PATCH /api/v1/settings/general/:id

```json
Request:  { "settingValue": "METRO-GAS" }
Response: { "success": true, "message": "Setting updated." }
```

### 15.3 Notification Settings

#### GET /api/v1/settings/notifications

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "1",
    "isEmail": true,
    "isSms": true,
    "isInapp": false,
    "lowStockAlert": true,
    "overdueAlertDays": 3
  }
}
```

#### PATCH /api/v1/settings/notifications

```json
Request:
{ "isEmail": true, "isSms": true, "isInapp": true, "lowStockAlert": true, "overdueAlertDays": 5 }

Response: { "success": true, "message": "Notification preferences updated." }
```

### 15.4 Branches Full CRUD

#### GET /api/v1/branches/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "5",
    "name": "Uttara Zone",
    "address": "...",
    "phone": "...",
    "status": "ACTIVE",
    "buildingCount": 4
  }
}
```

#### PATCH /api/v1/branches/:id

```json
Request:  { "name": "Uttara North Zone", "address": "Updated address", "status": "ACTIVE" }
Response: { "success": true, "message": "Branch updated.", "data": { "id": "5", "name": "Uttara North Zone" } }
```

#### DELETE /api/v1/branches/:id

```json
Response: { "success": true, "message": "Branch soft-deleted." }
NOTE: Sets is_deleted=true. Branch stays in DB for historical reference.
```

### 15.5 Buildings Full CRUD

#### GET /api/v1/buildings/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "18",
    "name": "Rose Valley Heights",
    "area": "Uttara Sector 3",
    "totalFloors": 10,
    "totalFlats": 40,
    "occupiedFlats": 35,
    "branch": {
      "id": "5",
      "name": "Uttara Zone"
    }
  }
}
```

#### PATCH /api/v1/buildings/:id

```json
Request:  { "name": "Rose Valley Tower", "area": "Uttara Sector 3", "totalFloors": 12 }
Response: { "success": true, "message": "Building updated.", "data": { "id": "18" } }
```

#### DELETE /api/v1/buildings/:id

```json
Response: { "success": true, "message": "Building soft-deleted." }
NOTE: Sets is_deleted=true. Cannot delete if active meters/bills exist.
```

### 15.6 Flats Full CRUD

#### GET /api/v1/flats/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "104",
    "unitNumber": "Flat 4A",
    "floor": 4,
    "sizeSqft": 1450,
    "status": "OCCUPIED",
    "building": {
      "id": "18",
      "name": "Rose Valley Heights"
    },
    "currentOccupant": {
      "customerId": "89",
      "name": "Mahbubur Rahman",
      "ownershipType": "RENTER"
    },
    "meter": {
      "id": "76",
      "meterNumber": "MET-77890",
      "status": "ACTIVE"
    }
  }
}
```

#### PATCH /api/v1/flats/:id

```json
Request:  { "unitNumber": "Flat 4A", "floor": 4, "sizeSqft": 1500, "status": "OCCUPIED" }
Response: { "success": true, "message": "Flat updated.", "data": { "id": "104" } }
```

#### DELETE /api/v1/flats/:id

```json
Response: { "success": true, "message": "Flat soft-deleted." }
NOTE: BLOCKED if active meter installed (RESTRICT). BLOCKED if unpaid bills exist.
```

### 15.7 Customers Full CRUD

#### GET /api/v1/customers/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "89",
    "name": "Mahbubur Rahman",
    "phone": "+8801911223344",
    "status": "ACTIVE",
    "currentFlat": {
      "flatId": "104",
      "unitNumber": "Flat 4A",
      "buildingName": "Rose Valley Heights"
    },
    "creditBalance": "195.00",
    "totalUnpaidBills": 1,
    "totalOutstanding": "11805.00"
  }
}
```

#### PATCH /api/v1/customers/:id

```json
Request:  { "name": "Mahbubur Rahman Chowdhury", "phone": "+8801911223344", "status": "ACTIVE" }
Response: { "success": true, "message": "Customer updated.", "data": { "id": "89" } }
```

#### DELETE /api/v1/customers/:id

```json
Response: { "success": true, "message": "Customer soft-deleted." }
NOTE: Sets is_deleted=true. All flat mappings set is_active=false.
```

#### GET /api/v1/customers/:id/bills

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "total": 2,
    "unpaid": 1,
    "totalOutstanding": "11805.00"
  },
  "data": [
    {
      "id": "88701",
      "billNumber": "GAS-2026-000145",
      "billingMonth": "2026-05-01",
      "dueAmount": "11805.00",
      "status": "UNPAID"
    },
    {
      "id": "88650",
      "billNumber": "GAS-2026-000100",
      "billingMonth": "2026-04-01",
      "dueAmount": "0.00",
      "status": "PAID"
    }
  ]
}
```

#### GET /api/v1/customers/flat-mappings?flatId=104

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "312",
      "customerId": "89",
      "customerName": "Mahbubur Rahman",
      "ownershipType": "RENTER",
      "startDate": "2026-05-01",
      "isActive": true,
      "isPrimary": true
    }
  ]
}
```

### 15.8 Users Full CRUD

#### POST /api/v1/users

```json
Request:
{ "name": "Tariqul Islam", "email": "tariq@metrogas.cc", "password": "SecurePass123", "roleId": 3 }

Response 201:
{
  "success": true,
  "message": "User account created.",
  "data": { "id": "206", "name": "Tariqul Islam", "role": "Meter Reader", "permissionsCopied": 5 }
}
NOTE: permissions_json is auto-populated from role_id on creation.
```

#### GET /api/v1/users/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "206",
    "name": "Tariqul Islam",
    "email": "tariq@metrogas.cc",
    "role": {
      "id": "3",
      "name": "Meter Reader"
    },
    "permissionsJson": [
      "can_read_meter",
      "can_view_bills"
    ],
    "isActive": true,
    "lastLogin": "2026-05-19T08:00:00Z"
  }
}
```

#### PATCH /api/v1/users/:id

```json
Request:  { "name": "Tariqul Islam Sr.", "isActive": true }
Response: { "success": true, "message": "User updated.", "data": { "id": "206" } }
```

#### DELETE /api/v1/users/:id

```json
Response: { "success": true, "message": "User account deactivated." }
NOTE: Sets is_active=false. Hard delete blocked � audit_logs preserve history.
```

#### PATCH /api/v1/users/:id/role

```json
Request:  { "roleId": 2 }
Response: { "success": true, "message": "Role updated. Permissions re-copied.", "data": { "newRole": "Operator", "permissionsCopied": 8 } }
NOTE: Wipes old permissions_json and copies fresh from new role.
```

### 15.9 Roles Full CRUD

#### GET /api/v1/roles/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "3",
    "name": "Meter Reader",
    "permissions": [
      {
        "id": "1",
        "code": "can_read_meter",
        "name": "Log Meter readings"
      },
      {
        "id": "5",
        "code": "can_view_bills",
        "name": "View invoice list"
      }
    ],
    "userCount": 4
  }
}
```

#### PATCH /api/v1/roles/:id

```json
Request:  { "name": "Field Reader" }
Response: { "success": true, "message": "Role renamed.", "data": { "id": "3", "name": "Field Reader" } }
```

#### DELETE /api/v1/roles/:id

```json
Response: { "success": true, "message": "Role deleted." }
BLOCKED if: any users have this role_id assigned (returns 400 with userCount).
```

#### POST /api/v1/roles/:id/permissions

```json
Request:  { "permissionIds": [1, 2, 5] }
Response: { "success": true, "message": "3 permissions assigned to role." }
NOTE: Does NOT auto-update existing users. They keep their copied permissions_json.
```

#### DELETE /api/v1/roles/:id/permissions/:permissionId

```json
Response: { "success": true, "message": "Permission removed from role." }
NOTE: Does NOT affect existing users. Only changes future user assignments.
```

### 15.10 Meters Full CRUD

#### GET /api/v1/meters/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "76",
    "meterNumber": "MET-77890",
    "status": "ACTIVE",
    "flat": {
      "id": "104",
      "unitNumber": "Flat 4A",
      "buildingName": "Rose Valley Heights"
    },
    "lastReading": {
      "currentReading": "180.50",
      "readingMonth": "2026-05-01",
      "consumption": "80.50"
    }
  }
}
```

#### PATCH /api/v1/meters/:id

```json
Request:  { "status": "DAMAGED", "meterNumber": "MET-77890-REP" }
Response: { "success": true, "message": "Meter updated.", "data": { "id": "76", "status": "DAMAGED" } }
```

#### DELETE /api/v1/meters/:id

```json
Response: { "success": true, "message": "Meter decommissioned." }
NOTE: Sets is_deleted=true. Unique index on flat_id released � new meter can now be installed.
```

### 15.11 Employees Full CRUD

#### GET /api/v1/employees/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "12",
    "name": "Mohammad Selim",
    "designation": "Reader",
    "unitMappings": [
      {
        "unitType": "BUILDING",
        "unitId": "18",
        "unitName": "Rose Valley Heights"
      }
    ]
  }
}
```

#### PATCH /api/v1/employees/:id

```json
Request:  { "name": "Mohammad Selim Hossain", "designation": "Senior Reader", "isActive": true }
Response: { "success": true, "message": "Employee updated." }
```

#### DELETE /api/v1/employees/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Employee soft-deleted.",
  "data": {
    "success": true,
    "message": "Employee soft-deleted."
  }
}
```

#### DELETE /api/v1/employees/:id/unit-mappings/:mappingId

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Unit assignment removed.",
  "data": {
    "success": true,
    "message": "Unit assignment removed."
  }
}
```

### 15.12 Charge Types & Configs Full CRUD

#### GET /api/v1/charge-types/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "3",
    "name": "Gas Usage Slabs",
    "chargeMode": "UNIT",
    "applyOn": "CONSUMPTION",
    "isSlab": true,
    "orderNo": 1,
    "isActive": true
  }
}
```

#### PATCH /api/v1/charge-types/:id

```json
Request:  { "name": "Gas Consumption Slabs", "orderNo": 1, "isActive": true }
Response: { "success": true, "message": "Charge type updated." }
```

#### DELETE /api/v1/charge-types/:id

```json
Response: { "success": true, "message": "Charge type deleted." }
BLOCKED if: active charge_configs reference this type.
```

#### GET /api/v1/charge-configs/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "14",
    "chargeTypeId": "3",
    "value": "0.00",
    "effectiveFrom": "2026-05-01",
    "effectiveTo": null,
    "graceDays": 5,
    "isActive": true,
    "slabs": [
      {
        "slabOrder": 1,
        "fromUnit": "0.00",
        "toUnit": "100.00",
        "rate": "120.00"
      },
      {
        "slabOrder": 2,
        "fromUnit": "101.00",
        "toUnit": null,
        "rate": "180.00"
      }
    ],
    "mappedBuildings": [
      {
        "buildingId": "18",
        "name": "Rose Valley Heights"
      }
    ]
  }
}
```

#### PATCH /api/v1/charge-configs/:id

```json
Request:  { "value": 130.00, "effectiveFrom": "2026-06-01", "isActive": true }
Response: { "success": true, "message": "Charge config updated." }
```

#### DELETE /api/v1/charge-configs/:id

```json
Response: { "success": true, "message": "Charge config removed." }
BLOCKED if: any bill_items reference this config (RESTRICT � audit trail preserved).
```

#### DELETE /api/v1/charge-configs/mappings/:id

```json
Response: { "success": true, "message": "Building-config binding removed." }
NOTE: Building will no longer have this charge type applied in future billing runs.
```

#### PATCH /api/v1/charge-configs/:id/slabs  (replace all slabs)

```json
Request:
{
  "slabs": [
    { "slabOrder": 1, "fromUnit": 0, "toUnit": 50, "rate": 100 },
    { "slabOrder": 2, "fromUnit": 51, "toUnit": 150, "rate": 150 },
    { "slabOrder": 3, "fromUnit": 151, "toUnit": null, "rate": 200 }
  ]
}
Response: { "success": true, "message": "Slabs replaced. 3 slabs active.", "data": [...] }
NOTE: This DELETES all existing slabs for this config and INSERTS the new set.
```

### 15.13 Inventory - Full CRUD Payloads

#### GET /api/v1/inventory/suppliers/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "18",
    "name": "Technoworth Meters Ltd",
    "contactPerson": "Rashedul Bari",
    "supplierType": "METER",
    "isActive": true
  }
}
```

#### PATCH /api/v1/inventory/suppliers/:id

```json
Request:  { "contactPerson": "Jahangir Hossain", "phone": "+8801755566699", "isActive": true }
Response: { "success": true, "message": "Supplier updated." }
```

#### DELETE /api/v1/inventory/suppliers/:id

```json
Response: { "success": true, "message": "Supplier deactivated." }
NOTE: Sets is_active=false. BLOCKED if pending purchase orders reference this supplier.
```

#### GET /api/v1/inventory/materials/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "4",
    "name": "Smart Gas Meter G1.6",
    "type": "METER",
    "unit": "pcs",
    "isActive": true
  }
}
```

#### PATCH /api/v1/inventory/materials/:id

```json
Request:  { "name": "Smart Gas Meter G2.0", "isActive": true }
Response: { "success": true, "message": "Material updated." }
```

#### GET /api/v1/inventory/materials/purchases/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "89",
    "purchaseReference": "INV-2026-00445",
    "material": {
      "id": "4",
      "name": "Smart Gas Meter G1.6"
    },
    "supplier": {
      "id": "18",
      "name": "Technoworth Meters Ltd"
    },
    "quantity": "100.000",
    "unitPrice": "1200.00",
    "additionalCharges": "5000.00",
    "totalPrice": "125000.00",
    "effectiveUnitCost": "1250.0000",
    "purchaseDate": "2026-05-15"
  }
}
```

#### GET /api/v1/inventory/items/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "8",
    "name": "45kg Liquefied Cylinder",
    "category": "Cylinder",
    "unit": "pcs",
    "stockSummary": [
      {
        "locationType": "STORE",
        "locationName": "Central Dhaka Godown",
        "currentQuantity": "120.000"
      },
      {
        "locationType": "BUILDING",
        "locationName": "Rose Valley Heights",
        "currentQuantity": "30.000"
      }
    ]
  }
}
```

#### PATCH /api/v1/inventory/items/:id

```json
Request:  { "name": "45kg LPG Cylinder", "isActive": true }
Response: { "success": true, "message": "Inventory item updated." }
```

#### DELETE /api/v1/inventory/items/:id

```json
Response: { "success": true, "message": "Inventory item deactivated." }
BLOCKED if: active stock records exist for this item.
```

#### GET /api/v1/inventory/stores/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "2",
    "name": "Central Dhaka Godown",
    "address": "12/A Tejgaon",
    "contactPhone": "+8801799887766",
    "isActive": true,
    "branch": {
      "id": "5",
      "name": "Uttara Zone"
    },
    "stockItems": [
      {
        "itemName": "45kg Cylinder",
        "currentQuantity": "120.000"
      }
    ]
  }
}
```

#### PATCH /api/v1/inventory/stores/:id

```json
Request:  { "name": "Central Dhaka Godown � Expanded", "contactPhone": "+8801799887799" }
Response: { "success": true, "message": "Store updated." }
```

#### GET /api/v1/inventory/transactions?itemId=8&transactionType=TRANSFER

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 1
  },
  "data": [
    {
      "id": "1882",
      "itemName": "45kg Cylinder",
      "fromLocation": {
        "type": "STORE",
        "name": "Central Dhaka Godown"
      },
      "toLocation": {
        "type": "BUILDING",
        "name": "Rose Valley Heights"
      },
      "transactionType": "TRANSFER",
      "quantity": "30.000",
      "transactionDate": "2026-05-19"
    }
  ]
}
```

### 15.14 Subscription Plans

#### GET /api/v1/subscription-plans

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 3
  },
  "data": [
    {
      "id": "1",
      "name": "Starter",
      "priceMonthly": "99.00",
      "maxBranch": 1,
      "maxBuilding": 5,
      "maxFlats": 100,
      "isActive": true
    },
    {
      "id": "2",
      "name": "Professional",
      "priceMonthly": "250.00",
      "maxBranch": 5,
      "maxBuilding": 50,
      "maxFlats": 1000,
      "isActive": true
    },
    {
      "id": "3",
      "name": "Enterprise",
      "priceMonthly": "500.00",
      "maxBranch": 20,
      "maxBuilding": 200,
      "maxFlats": 5000,
      "isActive": true
    }
  ]
}
```

#### POST /api/v1/subscription-plans  (Super Admin only)

```json
Request:
{
  "name": "Enterprise Plus", "priceMonthly": 750.00, "priceYearly": 7500.00,
  "features": ["billing", "inventory", "sms", "bkash", "advanced_reports"],
  "maxBranch": 50, "maxBuilding": 500, "maxFlats": 10000
}

Response 201:
{ "success": true, "message": "Plan created.", "data": { "id": "4", "name": "Enterprise Plus" } }
```

### 15.15 Meter Readings Full CRUD

#### GET /api/v1/meter-readings/:id

Response 200:

```json
{
  "statusCode": 200,
  "success": true,
  "message": "Fetched successfully",
  "data": {
    "id": "1045",
    "meter": {
      "id": "76",
      "meterNumber": "MET-77890"
    },
    "readingMonth": "2026-05-01",
    "previousReading": "100.00",
    "currentReading": "180.50",
    "consumption": "80.50",
    "readingDate": "2026-05-19",
    "status": "AVERAGE"
  }
}
```

#### PATCH /api/v1/meter-readings/:id  (correction by admin)

```json
Request:  { "currentReading": 185.00, "notes": "Corrected manual entry error" }
Response: { "success": true, "message": "Reading corrected. consumption recalculated to 85.00." }
NOTE: Audit log auto-created showing old_values vs new_values.
```

---

## 16. Data Flow � What Happens In The Database Per Action

This section shows the exact DB write sequence for every major operation.

### 16.1 POST /auth/register

```
1. INSERT subscription_plans (read existing plan � no write)
2. INSERT tenants { company_name, subscription_plan_id, status: ACTIVE }
3. INSERT roles { tenant_id, name: "Tenant Admin" }
4. SELECT permissions WHERE code IN system defaults
5. INSERT role_permissions (bulk � maps all Admin permissions to role)
6. INSERT users { tenant_id, email, password_hash, role_id }
7. SELECT role_permissions WHERE role_id = newRoleId
8. UPDATE users SET permissions_json = [copied array]
```

### 16.2 POST /auth/login

```
1. SELECT users WHERE email = ? AND tenant_id = ? (from subdomain middleware)
2. bcrypt.compare(password, password_hash)
3. UPDATE users SET last_login = NOW()
4. RETURN JWT { userId, tenantId, permissionsJson }
```

### 16.3 POST /customers/flat-mappings (assign customer to flat)

```
BEGIN TRANSACTION
1. SELECT customer_flat_mappings WHERE flat_id = ? AND is_active = true   (check vacancy)
   -> If found: THROW 400 "Flat already occupied"
2. INSERT customer_flat_mappings { customer_id, flat_id, is_active: true, start_date }
3. UPDATE flats SET status = 'OCCUPIED' WHERE id = flat_id
COMMIT
```

### 16.4 POST /meters (install meter)

```
BEGIN TRANSACTION
1. SELECT meters WHERE flat_id = ? AND is_deleted = false   (check no existing meter)
   -> If found: THROW 400 "Meter already installed in this flat"
2. INSERT meters { flat_id, meter_number, status: ACTIVE }
COMMIT
```

### 16.5 POST /meter-readings (log monthly reading)

```
BEGIN TRANSACTION
1. SELECT meter_readings WHERE meter_id = ? AND reading_month = ?   (check duplicate)
   -> If found: THROW 409 "Reading already logged for this month"
2. SELECT meter_readings WHERE meter_id = ? ORDER BY reading_month DESC LIMIT 1
   -> previous_reading = last row's current_reading (or 0 if first reading)
3. IF currentReading < previousReading: THROW 400 "Current reading cannot be less than previous"
4. consumption = currentReading - previousReading
5. status = consumption > threshold ? "HIGH" : "AVERAGE"
6. INSERT meter_readings { meter_id, reading_month, previous_reading, current_reading, consumption, status }
COMMIT
```

### 16.6 POST /billing/generate (bulk bill generation)

```
BEGIN TRANSACTION (per flat)
1. SELECT flats WHERE building_id = ? AND status = OCCUPIED AND is_deleted = false
2. FOR EACH flat:
   a. SELECT meter WHERE flat_id = ? -> get meter_id
   b. SELECT meter_readings WHERE meter_id = ? AND reading_month = billingMonth -> get consumption
   c. SELECT charge_config_mappings WHERE building_id = ? AND status = ACTIVE -> get configs
   d. SELECT charge_configs + charge_slabs for each config
   e. CHECK: SELECT bills WHERE flat_id = ? AND billing_month = ? -> prevent duplicate
   f. CALCULATE (in-memory):
      - Run each charge in order_no sequence
      - UNIT charges: apply slabs if is_slab=true, else flat rate * consumption
      - FIXED charges: add value directly
      - PERCENTAGE charges: apply % on CONSUMPTION/SUBTOTAL/TOTAL depending on apply_on
   g. SELECT bills WHERE flat_id = ? AND status != PAID ORDER BY billing_month DESC LIMIT 1
      -> previous_due = sum of due_amount from unpaid bills
   h. due_amount = total_amount + previous_due - customer_credit_balance
   i. INSERT bills { flat_id, bill_number, billing_month, total_usage, total_amount, previous_due, due_amount, status: UNPAID }
   j. INSERT bill_items (bulk) � one row per charge type applied
   k. INSERT notification_queue { recipient: customer.phone, message_body, status: PENDING }
COMMIT (each flat is its own transaction)
```

### 16.7 POST /payments/manual

```
BEGIN TRANSACTION
1. SELECT bills WHERE id = bill_id FOR UPDATE  (lock the row)
2. IF bill.status = PAID: THROW 400 "Bill already paid"
3. INSERT payments { bill_id, flat_id, amount, payment_method, payment_date }
4. INSERT transactions { reference_id: payment.id, amount, transaction_date }  (double-entry)
5. IF amount >= bill.due_amount:
   - UPDATE bills SET status = PAID, due_amount = 0
   - IF amount > bill.due_amount: UPDATE customers SET credit_balance += (amount - due_amount)
6. IF amount < bill.due_amount:
   - UPDATE bills SET status = PARTIALLY_PAID, due_amount -= amount
COMMIT
```

### 16.8 POST /inventory/stock/transfer

```
BEGIN TRANSACTION
1. SELECT inventory_stock WHERE item_id = ? AND location_id = fromLocationId AND location_type = fromType FOR UPDATE
2. IF stock.current_quantity < quantity: THROW 400 "Insufficient stock"
3. UPDATE inventory_stock SET current_quantity -= quantity WHERE item_id=? AND location_id=fromLocationId
4. SELECT inventory_stock WHERE item_id = ? AND location_id = toLocationId AND location_type = toType
   -> IF exists: UPDATE current_quantity += quantity
   -> IF not exists: INSERT inventory_stock { item_id, location_id, location_type, current_quantity: quantity }
5. INSERT inventory_transactions { item_id, from_location_id, to_location_id, transaction_type: TRANSFER, quantity }
COMMIT
```

---

## 17. Common Validation Rules � Quick Reference

| Entity | Rule | Error Code |
|---|---|---|
| Flat occupancy | Only 1 active customer per flat | 400 |
| Meter per flat | Only 1 active meter per flat | 400 |
| Reading per month | Only 1 reading per meter per month | 409 |
| Reading value | current_reading >= previous_reading | 400 |
| Bill per flat/month | Only 1 bill per flat per billing_month | 409 |
| Charge per building | Only 1 active charge_type config per building | 400 |
| Subscription limits | Branches/Buildings/Flats count within plan max | 403 |
| Stock transfer | from_location must have sufficient stock | 400 |
| Delete meter | Cannot delete flat with active meter | 400 |
| Delete role | Cannot delete role with active users | 400 |
| Tenant isolation | All queries filter by tenant_id from subdomain | 403 |
| Payment on paid bill | Cannot pay an already PAID bill | 400 |
| Slab ranges | Last slab must have to_unit = NULL (unlimited) | 400 |

---

## 18. NestJS Guard & Middleware Flow

Every request goes through this chain:

```
HTTP Request
    |
    v
TenantMiddleware
  - Extract subdomain from Host header
  - Query: SELECT * FROM tenants WHERE subdomain = ?
  - If not found: throw NotFoundException
  - Inject tenantId into request context: req.tenantId = tenant.id
    |
    v
JwtAuthGuard  (@UseGuards(JwtAuthGuard))
  - Extract token from HTTP-only secure cookie: req.cookies.accessToken
  - Decode and validate JWT: { userId, tenantId, permissionsJson }
  - Verify tenantId in JWT matches req.tenantId (cross-tenant check)
  - Attach user to request: req.user = { id, tenantId, permissionsJson }
    |
    v
PermissionsGuard  (@RequirePermissions('can_generate_bill'))
  - Read required permission from @RequirePermissions() decorator
  - Check: req.user.permissionsJson.includes(requiredPermission)
  - If missing: throw ForbiddenException 403
    |
    v
Controller Method
  - All Prisma queries automatically include: WHERE tenant_id = req.user.tenantId
  - This is enforced via a global Prisma middleware/interceptor
```

---

## 19. Environment Variables Required (.env)

```
# Database
DATABASE_URL="postgresql://user:password@localhost:5432/gaspay_db"

# JWT
JWT_SECRET="your-super-secret-jwt-key-min-32-chars"
JWT_REFRESH_SECRET="your-refresh-secret-key-different-from-access"
JWT_EXPIRES_IN="15m"
JWT_REFRESH_EXPIRES_IN="7d"

# App
APP_PORT=3000
APP_BASE_DOMAIN="gaspay.cc"   # Used for subdomain parsing

# bKash Gateway
BKASH_APP_KEY="your-bkash-app-key"
BKASH_APP_SECRET="your-bkash-app-secret"
BKASH_USERNAME="your-bkash-username"
BKASH_PASSWORD="your-bkash-password"
BKASH_BASE_URL="https://tokenized.sandbox.bka.sh/v1.2.0-beta"
BKASH_CALLBACK_URL="https://api.gaspay.cc/api/v1/payments/bkash/callback"

# Redis (for queue workers)
REDIS_HOST="localhost"
REDIS_PORT=6379

# CORS
ALLOWED_ORIGINS="https://app.gaspay.cc,http://localhost:3001"
```

---

## 20. Comprehensive System Operations & Interactive UI/UX Workflows

This section maps out the full functional operations of the GasPay site, including role-specific dashboards, dynamic screen transitions, and detailed workflow diagrams.

### 20.1 End-to-End Operational Lifecycle Diagram

The following comprehensive system-wide flow maps every interaction across all 5 operational user roles (Super Admin, Tenant Admin, Field Reader/Operator, Customer, and Warehouse Manager).

```mermaid
graph TD
    %% Roles
    SA[Super Admin]
    TA[Tenant Admin]
    FR[Field Reader/Operator]
    CUST[Customer]
    WM[Warehouse Manager]

    %% Super Admin Flows
    SA -->|1. Manages| SP[Subscription Plans]
    SA -->|2. Audits| GL[Global Ledger & System Logs]

    %% Tenant Admin Setup Flows
    TA -->|3. Subscribes To| SP
    TA -->|4. Sets Up| BH[Property Hierarchy: Branch > Building > Flat]
    TA -->|5. Configures| CE[Charging Engine Slabs & Rules]
    TA -->|6. Provisions| EMP[Staff Accounts & Roles]

    %% Field Reader & Tech Flows
    FR -->|7. Visits Site & Reads| MET[Gas Meter]
    FR -->|8. Logs Reading| MR[Meter Readings Table]
    MR -->|9. Computes| CONS[Net Consumption]

    %% Billing Execution
    CONS -->|10. Triggers billing run| BE[Billing Engine]
    CE -->|11. Supplies active rates| BE
    BE -->|12. Generates| BL[Invoice Master & Bill Items]
    BL -->|13. Enqueues alert| NQ[Notification Queue]

    %% Alert Dispatch
    NQ -->|14. Sends SMS/Email| CUST

    %% Customer Actions
    CUST -->|15. Views Bill Online| CP[Customer Portal Dashboard]
    CP -->|16. Selects Payment| PG[bKash Payment Gateway]
    PG -->|17. Callback success| PAY[Payments Table]
    PAY -->|18. Updates ledger| TX[General Ledger Transactions]
    PAY -->|19. Updates bill status| BL
    PAY -->|20. Updates credit| CUST

    %% Warehouse Operations
    WM -->|21. Procures parts| SUPP[Suppliers & Materials]
    WM -->|22. Updates warehouse| WH[Stores Stock]
    WH -->|23. Allocates to technicians| TR[Stock Transfer]
    TR -->|24. Deploys hardware| MET
```

---

### 20.2 Dynamic UI/UX Screen Transition Map (The Frontend Site Structure)

This map shows how screens, pages, and modular modals connect and transition in the frontend Next.js 16 Web Application.

```mermaid
graph TD
    %% Main Routes
    Landing[Landing Page / Domain Root] -->|1. Click Login| Login[Auth Login Page]
    Landing -->|2. Click Signup| Register[Tenant Registration Page]

    %% Workspace Subdomain Routings
    Subdomain{Subdomain Checker Middleware}
    Login --> Subdomain
    Register --> Subdomain

    %% Main Dashboards
    Subdomain -->|If Admin| AdminDash[Tenant Admin Dashboard]
    Subdomain -->|If Staff/Reader| StaffDash[Field Staff Portal Dashboard]
    Subdomain -->|If Customer| CustPortal[Customer Portal Dashboard]

    %% Tenant Admin Sub-pages & Modals
    AdminDash -->|Menu| PropMgr[Property Manager View]
    PropMgr -->|Add| NewBuildingModal[New Building Modal]
    PropMgr -->|Add| NewFlatModal[New Flat Modal]

    AdminDash -->|Menu| BillingMgr[Billing & Invoicing Panel]
    BillingMgr -->|Trigger| RunBillingModal[Run Billing Run Modal]
    BillingMgr -->|View Calc| PreviewBillModal[In-Memory Bill Preview Modal]

    AdminDash -->|Menu| CRMPanel[Customer & CRM Mappings]
    CRMPanel -->|Assign Renter| OccupancyModal[Assign Occupant Modal]

    AdminDash -->|Menu| RateMgr[Rate Configurator Page]
    RateMgr -->|Add Slabs| SlabConfigModal[Slab Pricing Modal]

    AdminDash -->|Menu| InventMgr[Warehouse & Inventory Panel]
    InventMgr -->|New Transfer| StockTransferModal[Stock Transfer Modal]
    InventMgr -->|Procurement| PurchaseOrderModal[Log Purchase Modal]

    %% Field Staff Sub-pages
    StaffDash -->|Select Building| ReadingList[Meter Reading Sheet]
    ReadingList -->|Click Flat| EnterReadingModal[Log Index Modal]

    %% Customer Portal Sub-pages
    CustPortal -->|View Current| BillDetail[Bill Detail & Breakdown Screen]
    BillDetail -->|Pay Now| CheckoutScreen[bKash Checkout Redirect]
    CustPortal -->|History| PastBillsList[Payment History Sheet]
```

---

### 20.3 Detailed Multi-Step Operations Walkthrough of the Site

#### 1. Tenant Admin Site Workspace Initialization

- **Action**: The Tenant Admin logs into `tenant1.gaspay.cc/dashboard`.
- **First-Time Screen Transition**: The system detects zero properties and redirects to `Setup Wizard`.
- **Properties Definition**:
  1. Admin clicks **Add Branch** -> enters "Dhaka South".
  2. Admin clicks **Add Building** -> selects "Dhaka South" branch, enters name "Lily Heights", floor count 12, flat count 48.
  3. Admin click **Bulk Generate Flats** -> enters suffix "A,B,C,D" -> system auto-creates flats `1A`, `1B`, `1C`, `1D` through `12A` to `12D`.
  4. Database status for all generated flats defaults to `VACANT`.

#### 2. Hardware Allocation & Customer CRM Occupancy Mapping

- **Action**: A resident lease begins. The Admin registers occupant and sets up hardware.
- **Lease Assignment**:
  1. Admin navigates to **CRM Panel** -> clicks **Add Customer** -> registers "Sharif Ahmed" (+88018XXXXXXXX).
  2. Admin selects Flat `3B`, clicks **Assign Customer** -> selects "Sharif Ahmed" as renter, inputs lease start date -> system updates Flat `3B` database status from `VACANT` to `OCCUPIED`.
- **Hardware Integration**:
  1. Admin navigates to **Inventory Dashboard** -> selects a Gas Meter serial "MET-998811" from **Tejgaon Store** stock.
  2. Admin clicks **Install in Flat** -> selects Flat `3B` -> system binds the meter to the flat.
  3. The Gas Meter database status changes from `IN_STOCK` to `ACTIVE`.

#### 3. Field Meter Reading Operation

- **Action**: End of month meter logs.
- **Field Execution**:
  1. Field Reader logs into `staff.gaspay.cc` on a mobile tablet.
  2. Reader selects **Lily Heights** -> system displays lists of occupied flats lacking current month reading.
  3. Reader taps Flat `3B` -> system shows previous index reading (`150.00 units`).
  4. Reader reads physical dial, inputs current index (`230.00 units`), clicks **Submit**.
  5. The API validates `230.00 >= 150.00`, calculates consumption `80.00 units`, sets reading status to `AVERAGE`, and writes the record.

#### 4. Automatic Billing Run & Notification Dispatch

- **Action**: Admin compiles bills for the month.
- **Invoicing Execution**:
  1. Admin goes to **Billing Panel** -> selects month "2026-05" -> selects building **Lily Heights**.
  2. Admin clicks **Calculate Preview** -> system runs calculation engine for all occupant flats in-memory and shows totals.
  3. Admin clicks **Confirm & Generate Bills** -> system locks the building and processes bulk generation inside database transactions.
  4. **Bill Itemization**:
     - *Lily Heights Unit Rate Slab*: 0-100 units = BDT 120/unit.
     - Flat `3B` gas consumption BDT: $80 \times 120$ = **BDT 9,600**.
     - Flat `3B` service charge fixed: **BDT 500**.
     - Subtotal = **BDT 10,100**.
     - Flat `3B` VAT percentage (5%): **BDT 505**.
     - Previous unpaid due balance: **BDT 0**.
     - Final payable amount: **BDT 10,605**.
  5. The system generates unique invoice number `GAS-2026-10492` and adds a dispatch job to the outbound `notification_queue`.
  6. An asynchronous SMS gateway worker reads the queue and dispatches an SMS to Sharif Ahmed:
     *"Dear Sharif Ahmed, your GasPay invoice GAS-2026-10492 for BDT 10,605 is ready. Pay online before due date 2026-06-10 at tenant1.gaspay.cc/pay/GAS-2026-10492"*

#### 5. Customer Digital Checkout & Ledger Bookkeeping

- **Action**: Customer completes checkout online.
- **Payment Collection**:
  1. Sharif Ahmed clicks the SMS link, opening `tenant1.gaspay.cc/pay/GAS-2026-10492` directly on his smartphone.
  2. The screen shows an itemized billing breakdown. Customer clicks **Pay with bKash**.
  3. System initiates bKash checkout, displaying the payment PIN and OTP confirmation page.
  4. Upon successful payment verification, the bKash webhook calls the backend callback controller.
  5. The backend executes a database transaction:
     - Updates `bills.status` to `PAID` and `bills.due_amount` to `0.00`.
     - Inserts a record in `payments` containing bKash TrxID `BKX9988112233`.
     - Inserts double-entry accounting ledger entries in `transactions` (Debit Cash, Credit Accounts Receivable).
  6. The Customer Portal screen updates to show a "Payment Successful" receipt and a download PDF option.

#### 6. Stock Distribution Logistics

- **Action**: Warehouse manager transfers physical inventory.
- **Logistics Flow**:
  1. Warehouse Manager logs into `tenant1.gaspay.cc/inventory`.
  2. Manager registers a bulk supplier invoice: 100 meters at BDT 1,200/unit + BDT 5,000 logistics fees. System updates Central Store stock count of "Smart Gas Meter G1.6" by `+100` and sets effective cost to `BDT 1,250`.
  3. Manager logs a Stock Transfer: 20 meters from **Tejgaon Store** (Central) to **Uttara Store** (Branch).
  4. System verifies Tejgaon Store has enough stock, decrements Central Store stock by `20`, increments Uttara Store stock by `20`, and logs an audited inventory transaction record.

---

## 21. Core TypeScript Utility Implementations

These standard utilities ensure robust, type-safe, and consistent API request/response lifecycles (pagination, async handling, request body picking, and uniform API responses).

### 21.1 Pagination Helper (`paginationHelper.ts`)

Converts dynamic client query parameters (page, limit, sort) into standardized MongoDB/Prisma pagination options (`skip`, `take`, `orderBy`).

```typescript
type IOptions = {
    page?: number,
    limit?: number,
    sortOrder?: string,
    sortBy?: string
}

type IOptionsResult = {
    page: number,
    limit: number,
    skip: number,
    sortBy: string,
    sortOrder: string
}

const calculatePagination = (options: IOptions): IOptionsResult => {
    const page: number = Number(options.page) || 1;
    const limit: number = Number(options.limit) || 9;
    const skip: number = (Number(page) - 1) * limit;

    const sortBy: string = options.sortBy || 'createdAt';
    const sortOrder: string = options.sortOrder || 'desc';

    return {
        page,
        limit,
        skip,
        sortBy,
        sortOrder
    }
}

export const paginationHelper = {
    calculatePagination
}
```

### 21.2 Async Handler Wrapper (`catchAsync.ts`)

Express middleware utility that removes `try/catch` boilerplate from route controllers by catching rejected promises and forwarding them to the global error handler.

```typescript
import type { NextFunction, Request, Response } from "express";
import config from "../config";

type AsyncHandler = (req: Request, res: Response, next: NextFunction) => Promise<void>

export const catchAsync = (fn: AsyncHandler) => (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch((err: any) => {
        if (config.node_env === "development") {
            console.log(err);
        }
        next(err)
    })
}
```

### 21.3 Body Field Picker (`pick.ts`)

A helper that extracts specified keys from an object, which is useful for filtering allowed search query parameters from `req.query`.

```typescript
const pick = <T extends Record<string, unknown>, k extends keyof T>(obj: T, keys: k[]): Partial<T> => {
    const finalObject: Partial<T> = {};

    for (const key of keys) {
        if (obj && Object.hasOwnProperty.call(obj, key)) {
            finalObject[key] = obj[key]
        }
    }

    return finalObject;
}

export default pick;
```

### 21.4 Standardized Response Helper (`sendResponse.ts`)

Enforces that every route response matches the uniform `TResponse<T>` interface, containing the standard `statusCode`, `success`, `message`, `meta` (optional for lists), and `data` fields.

```typescript
import type { Response } from "express";

interface TMeta {
    page: number;
    limit: number;
    total: number
}

interface TResponse<T> {
    statusCode: number;
    success: boolean;
    message: string;
    data: T;
    meta?: TMeta
}

export const sendResponse = <T>(res: Response, data: TResponse<T>) => {
    res.status(data.statusCode).json({
        statusCode: data.statusCode,
        success: data.success,
        message: data.message,
        meta: data.meta || null || undefined,
        data: data.data || null || undefined
    })
}
```

---

## 22. Advanced Auth, Security & OTP Lifecycle Specifications

This section specifies the robust authentication lifecycle designed for frontend Next.js operations, including OTP verification via Redis, cookie-based token rotation (Access/Refresh), password resets, change-password flows, logout endpoints, and rate-limiting security guards.

### 22.1 Authentication & Security Architecture Overview

1. **Double-Cookie Security Pattern**:
   - `accessToken`: Short-lived (e.g., 15 minutes), HTTP-only, secure, SameSite=Lax cookie containing userId, tenantId, and permissions.
   - `refreshToken`: Long-lived (e.g., 7 days), HTTP-only, secure, SameSite=Lax cookie used solely to rotate expired access tokens.
2. **OTP Lifecycle using Redis**:
   - OTP request generates a cryptographically secure 6-digit number.
   - Saves OTP to Redis using key format `otp:tenantId:phone` with a 300-second (5-minute) TTL.
   - Limits OTP request rate using a key format `otp_limit:tenantId:phone` (e.g., max 1 SMS per 60 seconds).
   - Verifies OTP directly from Redis and invalidates the key immediately upon successful login (one-time use).
3. **API Rate Limiting Guard**:
   - Enforced globally via middleware: max 100 requests per 15 minutes per IP.
   - Auth routes (`/auth/login/*`, `/auth/forgot-password/*`) are strictly constrained: max 5 requests per 15 minutes per IP/Account to prevent brute-force attacks.

---

### 22.2 API Spec: Access/Refresh, OTP, Password Reset, and Logout

All responses conform exactly to the uniform `TResponse<T>` pattern.

#### 1. POST /api/v1/auth/login/credentials

Standard email/password authentication. Sets HTTP-only secure cookies (`accessToken`, `refreshToken`) on successful login.

```json
Request:
{
  "email": "admin@metrogas.cc",
  "password": "SecurePassword123"
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Login successful. Authentication cookies established.",
  "data": {
    "user": {
      "id": "201",
      "name": "Zahid Hasan",
      "email": "admin@metrogas.cc",
      "role": "Tenant Admin",
      "permissionsJson": ["can_read_meter", "can_generate_bill", "can_view_bills"]
    }
  }
}
```

#### 2. POST /api/v1/auth/login/otp-send

Sends a dynamic 6-digit OTP to the customer or staff mobile number and caches it in Redis.

```json
Request:
{
  "phone": "+8801911223344"
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "OTP sent successfully to mobile number. Valid for 5 minutes.",
  "data": {
    "phone": "+8801911223344",
    "retryAfterSeconds": 60
  }
}
```

#### 3. POST /api/v1/auth/login/otp-verify

Verifies the OTP against Redis. On success, logs user in and sets HTTP-only secure cookies.

```json
Request:
{
  "phone": "+8801911223344",
  "otp": "489201"
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "OTP verified successfully. Access granted.",
  "data": {
    "user": {
      "id": "89",
      "name": "Sharif Ahmed",
      "phone": "+8801911223344",
      "role": "Customer",
      "permissionsJson": ["can_view_bills", "can_pay_bill"]
    }
  }
}
```

#### 4. POST /api/v1/auth/refresh-token

Rotates expired `accessToken` using valid `refreshToken` read from the secure cookies. Sets a fresh `accessToken` cookie.

```json
Request:
{} (Automatically reads req.cookies.refreshToken)

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Access token refreshed successfully.",
  "data": {
    "accessTokenExpiresIn": "15m"
  }
}
```

#### 5. POST /api/v1/auth/logout

Clears both the `accessToken` and `refreshToken` secure cookies instantly.

```json
Request:
{}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Logged out successfully. Secure cookies cleared.",
  "data": null
}
```

#### 6. POST /api/v1/auth/forgot-password/request

Initiates forgotten password flow. Sends a secure reset link with token via SMS or Email (cached in Redis for 15 minutes).

```json
Request:
{
  "email": "zahid@metrogas.cc"
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Password reset token sent to registered email.",
  "data": {
    "resetTokenExpiryMinutes": 15
  }
}
```

#### 7. POST /api/v1/auth/forgot-password/reset

Validates the reset token and updates the user password.

```json
Request:
{
  "token": "rst_991823ab2",
  "newPassword": "BrandNewSuperSecurePass99!"
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Password updated successfully. Please login with new credentials.",
  "data": null
}
```

#### 8. POST /api/v1/auth/change-password

Authenticated endpoint for changing password while logged in.

```json
Request:
{
  "oldPassword": "SecurePassword123",
  "newPassword": "NewSuperPassword99!"
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Password updated successfully.",
  "data": null
}
```

#### 9. GET /api/v1/auth/me

Enables the frontend Next.js app to fetch the currently logged-in user profile on page refresh.

```json
Request:
{} (Automatically reads req.cookies.accessToken)

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Current user profile fetched successfully.",
  "data": {
    "user": {
      "id": "201",
      "name": "Zahid Hasan",
      "email": "admin@metrogas.cc",
      "role": "Tenant Admin",
      "permissionsJson": ["can_read_meter", "can_generate_bill", "can_view_bills"]
    }
  }
}
```

---

### 22.3 Redis Schema & OTP State Structure

```
Redis Key: otp:[tenantId]:[phone]
Value: "489201" (String)
TTL: 300 seconds

Redis Key: otp_limit:[tenantId]:[phone]
Value: "1" (String)
TTL: 60 seconds (prevents OTP SMS spam)

Redis Key: pw_reset:[resetToken]
Value: "userId_201" (String)
TTL: 900 seconds (15 minutes)
```

---

## 23. SaaS Dashboard Analytics, Bulk Operations & PDF Reports Specifications

This section specifies advanced functional endpoints required to power interactive charts, data grids, bulk uploads, audit ledgers, and PDF exports inside the frontend Next.js application.

### 23.1 Building & Branch Dashboard Analytics

#### 1. GET /api/v1/buildings/:id/analytics

Returns key summary cards and telemetry metrics for a specific building dashboard.

```json
Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Building analytics fetched successfully.",
  "data": {
    "buildingId": "18",
    "buildingName": "Rose Valley Heights",
    "metrics": {
      "totalFlats": 48,
      "occupiedFlats": 35,
      "vacantFlats": 13,
      "activeMeters": 35,
      "damagedMetersCount": 2,
      "occupancyRate": 72.92
    },
    "billingSummary": {
      "currentMonthGenerated": "125400.00",
      "currentMonthCollected": "98500.00",
      "collectionRate": 78.55,
      "totalOutstandingDues": "26900.00"
    }
  }
}
```

#### 2. GET /api/v1/branches/:id/analytics

Returns aggregated metrics for an entire territorial branch.

```json
Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Branch analytics fetched successfully.",
  "data": {
    "branchId": "5",
    "branchName": "Uttara Zone",
    "metrics": {
      "totalBuildings": 8,
      "totalFlats": 384,
      "occupiedFlats": 310,
      "activeMeters": 310,
      "collectionRateYTD": 92.45
    }
  }
}
```

---

### 23.2 CRM Customer Detailed Ledger

#### 1. GET /api/v1/customers/:id/ledger?page=1&limit=9

Returns a chronological credit/debit transaction log showing every invoice billed and every payment credited.

```json
Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Customer ledger logs fetched.",
  "meta": {
    "page": 1,
    "limit": 9,
    "total": 3
  },
  "data": {
    "customerId": "89",
    "customerName": "Sharif Ahmed",
    "creditBalance": "195.00",
    "ledgerLines": [
      { "date": "2026-05-19", "type": "PAYMENT", "reference": "Pay #8890-BKASH", "debit": "0.00", "credit": "12000.00", "balance": "195.00" },
      { "date": "2026-05-01", "type": "BILL", "reference": "GAS-2026-10492", "debit": "11805.00", "credit": "0.00", "balance": "11805.00" },
      { "date": "2026-04-19", "type": "PAYMENT", "reference": "Pay #8500-CASH", "debit": "0.00", "credit": "1200.00", "balance": "0.00" }
    ]
  }
}
```

---

### 23.3 Bulk Import Operations

#### 1. POST /api/v1/meter-readings/bulk-import

Allows upload of Excel sheet containing multiple readings for a building to speed up monthly logs.

```json
Request: Multi-part form-data
- File: [ readings_lily_heights_may2026.xlsx ]
- buildingId: 18
- readingMonth: "2026-05-01"

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Bulk import completed. 48 readings processed successfully.",
  "data": {
    "rowsProcessed": 48,
    "insertedCount": 46,
    "updatedCount": 2,
    "failedCount": 0
  }
}
```

#### 2. POST /api/v1/flats/bulk-import

Imports property flat layouts directly from a CSV/Excel layout blueprint during tenant setup.

```json
Request: Multi-part form-data
- File: [ flats_rose_valley.xlsx ]
- buildingId: 18

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Bulk flat layout imported successfully.",
  "data": {
    "flatsCreated": 40
  }
}
```

---

### 23.4 Billing Special Operations (Manual Adjustments)

#### 1. POST /api/v1/bills/:id/waive-late-fee

Waives late fee for a disputed bill (requires authorization).

```json
Request:
{
  "reason": "Customer was out of country; delay pre-notified to branch manager."
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Late fee waived successfully. Invoice updated.",
  "data": {
    "billId": "88701",
    "waivedAmount": "1180.50",
    "newDueAmount": "10624.50"
  }
}
```

#### 2. POST /api/v1/bills/:id/adjust-balance

Applies standard credit or debit adjustments to clear invoice discrepancies.

```json
Request:
{
  "adjustmentType": "CREDIT",
  "amount": 250.00,
  "reason": "Adjustment for over-average calculation correction."
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Adjustment credit applied successfully.",
  "data": {
    "billId": "88701",
    "adjustmentId": "adj_44091",
    "adjustedDue": "10374.50"
  }
}
```

---

### 23.5 Inventory Stock Ledger & Alarm Settings

#### 1. GET /api/v1/inventory/items/:id/ledger?page=1&limit=9

Chronological stock movement record for individual components.

```json
Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Stock movements ledger loaded.",
  "meta": { "page": 1, "limit": 9, "total": 2 },
  "data": {
    "itemId": "8",
    "itemName": "45kg LPG Cylinder",
    "ledger": [
      { "date": "2026-05-19", "from": "Central Dhaka Godown", "to": "Rose Valley Heights", "qty": "30.00", "type": "TRANSFER" },
      { "date": "2026-05-15", "from": "Technoworth Meters Ltd", "to": "Central Dhaka Godown", "qty": "150.00", "type": "PURCHASE" }
    ]
  }
}
```

#### 2. PATCH /api/v1/inventory/items/:id/alert-thresholds

Sets low-stock alert parameters to trigger email/SMS warnings to managers.

```json
Request:
{
  "lowStockThreshold": 15.00,
  "lowStockAlertEnabled": true
}

Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Low stock alert configurations updated.",
  "data": null
}
```

---

### 23.6 SaaS Global Executive & Financial Reports

#### 1. GET /api/v1/reports/revenue?startDate=2026-05-01&endDate=2026-05-19

Revenue and payment performance chart dataset.

```json
Response 200:
{
  "statusCode": 200,
  "success": true,
  "message": "Financial performance report fetched successfully.",
  "data": {
    "totalInvoiced": "1450200.00",
    "totalCollected": "1198500.00",
    "totalOutstanding": "251700.00",
    "splits": {
      "bkashOnline": "890500.00",
      "cashManual": "308000.00"
    },
    "collectionPercentage": 82.64
  }
}
```

#### 2. GET /api/v1/reports/export?type=REVENUE&format=PDF&startDate=2026-05-01&endDate=2026-05-19

Triggers standard binary stream of generated PDF reports for board review.

```
Headers:
- Content-Type: application/pdf
- Content-Disposition: attachment; filename="Revenue_Report_May_2026.pdf"

Response 200: (Binary stream of PDF file contents)
```

---

## 24. Detailed Entity Relation & Line-by-Line Database Schema Explanation

Here is the complete, line-by-line, table-by-table explanation of the GasPay database schema defined in the Gas Pay.xlsx spreadsheet and compiled into the PRD.

To make this super easy to understand, the 38 tables are grouped into 9 logical Business Domains.

### Domain 1: SaaS Multi-Tenancy & Security Core

This domain is the foundational layer. It governs your platform’s commercial subscription billing and user authorization system. Every tenant has completely isolated workspaces.

#### 1. SubscriptionPlans

- **Purpose**: Stores the commercial packages that gas companies (Tenants) subscribe to (e.g. Starter, Pro, Enterprise).
- **Key Columns**:
  - `price_monthly` & `price_yearly`: Plan pricing models.
  - `max_branch`, `max_building`, `max_flats`: Hard capacity limits enforced in the backend.
  - `features` (`JSONB`): A list of enabled modules (e.g., `["billing", "inventory", "sms"]`).
- **Relations**:
  - `SubscriptionPlans` (1) $\rightarrow$ (Many) `Tenants`. A subscription plan is assigned to many tenants.

#### 2. Tenants

- **Purpose**: Stores the companies operating gas utilities (e.g., Metropolitan Gas Co.). Every data point in downstream tables is partitioned by `tenant_id` to ensure complete privacy.
- **Key Columns**:
  - `company_name`: The name of the client firm.
  - `subscription_plan_id` (`FK`): Points to their current subscription level.
  - `status`: Workspace activation state (`ACTIVE`, `SUSPENDED`).
- **Relations**:
  - Connects as a parent to almost all database tables via `tenant_id`.

#### 3. Roles

- **Purpose**: Stores customizable permission groups (e.g., Tenant Admin, Meter Reader, Store Keeper).
- **Relations**:
  - `Tenants` (1) $\rightarrow$ (Many) `Roles`. Roles are created by and isolated within a specific Tenant.
  - `Roles` (1) $\rightarrow$ (Many) `RolePermissions`.
  - `Roles` (1) $\rightarrow$ (Many) `Users`.

#### 4. Permissions

- **Purpose**: The global, platform-wide static registry of actions a user can execute (e.g., `can_read_meter`, `can_generate_bill`, `can_view_reports`).
- **Relations**:
  - `Permissions` (1) $\rightarrow$ (Many) `RolePermissions`.

#### 5. RolePermissions

- **Purpose**: A junction/mapping table matching `Roles` to their standard assigned `Permissions`.
- **Relations**:
  - Binds `role_id` $\rightarrow$ `roles.id`
  - Binds `permission_id` $\rightarrow$ `permissions.id`

#### 6. Users

- **Purpose**: Central credential registry for platform login.
- **Key Columns**:
  - `email`: Login credential (unique per tenant).
  - `password_hash`: BCRYPT encrypted password.
  - `role_id` (`FK`): Inherited permission group.
  - `permissions_json` (`JSONB`): **Highly Important!** On creation, permissions are copied from `RolePermissions` into this array. This allows custom overrides (adding or taking away a specific permission from one single user without altering the entire group role).
- **Relations**:
  - Connects `role_id` $\rightarrow$ `roles.id`.

### Domain 2: Property Structure Hierarchy

This is the geographical layout of the property assets owned by a Tenant.

```
Branches (Optional) ──> Buildings ──> Flats
```

#### 7. Branches

- **Purpose**: Optional regional zones or territorial branch offices under a tenant (e.g., Uttara Branch, Mirpur Branch).
- **Relations**:
  - `Branches` (1) $\rightarrow$ (Many) `Buildings`.
  - `Branches` (1) $\rightarrow$ (Many) `Flats`.

#### 8. Buildings

- **Purpose**: Represents physical structures, apartment complexes, or housing projects where gas pipelines are installed.
- **Key Columns**:
  - `total_floors` & `total_flats`: Telemetry metrics.
  - `area` & `address`: Geographical descriptions for search filters.
- **Relations**:
  - Connects `branch_id` $\rightarrow$ `branches.id`.
  - `Buildings` (1) $\rightarrow$ (Many) `Flats`.

#### 9. Flats

- **Purpose**: The specific residential/commercial units in a building.
- **Key Columns**:
  - `unit_number`: The label (e.g., Flat 4A, Flat 3B).
  - `status`: State of occupancy (`VACANT`, `OCCUPIED`).
- **Relations**:
  - Connects `building_id` $\rightarrow$ `buildings.id`.
  - `Flats` (1) $\rightarrow$ (1) `Meters` (via `meters.flat_id`).
  - `Flats` (1) $\rightarrow$ (Many) `CustomerFlatMappings`.

### Domain 3: Customer Management & CRM Mappings

Governs the occupants residing in the property hierarchy.

#### 10. Customers

- **Purpose**: Stores occupant profiles (e.g., Sharif Ahmed, their phone and email).
- **Key Columns**:
  - `credit_balance`: Stores any overpaid monetary values to offset their next invoice.
- **Relations**:
  - `Customers` (1) $\rightarrow$ (Many) `CustomerFlatMappings`.
  - `Customers` (1) $\rightarrow$ (Many) `Bills` (indirectly linked via `flat_id`).

#### 11. CustomerFlatMappings

- **Purpose**: Connects `Customers` to `Flats` to specify who resides where and who gets billed.
- **Key Columns**:
  - `ownership_type`: Identifies if they are an `OWNER`, `RENTER`, or `FAMILY`.
  - `is_primary`: Boolean designating who is the primary contact to receive monthly SMS alerts.
  - `is_active`: Designates active occupancy.
- **Business Constraints**:
  - A flat can have only ONE active mapping at a time (unique partial index). Vacating the flat sets `is_active = false`, updates the flat back to `VACANT`, and records an `end_date`.
  - One customer can map to MULTIPLE flats (e.g., owning 3 apartments in a building).
- **Relations**:
  - Connects `customer_id` $\rightarrow$ `customers.id`
  - Connects `flat_id` $\rightarrow$ `flats.id`

### Domain 4: Employee Deployments

Manages field technicians and meter readers.

#### 12. Employees

- **Purpose**: Profiles of company staff members.
- **Relations**:
  - Connects `branch_id` $\rightarrow$ `branches.id`.
  - `Employees` (1) $\rightarrow$ (Many) `EmployeeUnitMappings`.

#### 13. EmployeeUnitMappings

- **Purpose**: Assigns employees to specific geographic properties or warehouses they are responsible for managing.
- **Key Columns**:
  - `unit_type` & `unit_id`: Polymorphic fields specifying target (e.g., `unit_type = 'BUILDING'`, `unit_id = 18` or `unit_type = 'STORE'`, `unit_id = 2`).
- **Relations**:
  - Connects `employee_id` $\rightarrow$ `employees.id`.

### Domain 5: Gas Meters & Readings

Telemetry logging of actual resource consumption.

#### 14. Meters

- **Purpose**: Physical gas meters installed at properties.
- **Relations**:
  - Connects `flat_id` $\rightarrow$ `flats.id`. Enforces a unique index: One active meter per flat.
  - `Meters` (1) $\rightarrow$ (Many) `MeterReadings`.

#### 15. MeterReadings

- **Purpose**: Logs monthly dial measurements.
- **Key Columns**:
  - `previous_reading` & `current_reading`: The dial logs.
  - `consumption`: System-calculated usage (`current_reading - previous_reading`).
  - `status`: Flags to trigger analytics (e.g., `AVERAGE`, `HIGH` usage, or `COMPLAIN`).
- **Relations**:
  - Connects `meter_id` $\rightarrow$ `meters.id`. Enforces unique index `(meter_id, reading_month)` to prevent logging two readings for a meter in the same month.

### Domain 6: Pricing Engine & Charging Configurations

This is your platform’s complex billing core. It defines pricing slabs, taxes, and service fees.

#### 16. ChargeTypes

- **Purpose**: Defines standard pricing templates.
- **Key Columns**:
  - `charge_mode`: Math category (`UNIT` based, `FIXED` fee, or `PERCENTAGE` tax).
  - `apply_on`: Target of calculation (`CONSUMPTION` gas fee, `SUBTOTAL` of charges, or global `TOTAL`).
  - `is_slab`: Triggers lookup of tiered volumetric slabs.
  - `is_late_fee`: Designates if this type acts as the late penalty.
  - `order_no`: Execution priority. Percentage charges on SUBTOTAL must run after unit and fixed charges.
- **Relations**:
  - `ChargeTypes` (1) $\rightarrow$ (Many) `ChargeConfigs`.

#### 17. ChargeConfigs

- **Purpose**: Stores active rate prices (e.g., BDT 120 per unit or 5% VAT) along with an active date span.
- **Key Columns**:
  - `value`: Numerical rate.
  - `effective_from` & `effective_to`: Date boundaries. If `effective_to` is `NULL`, it runs indefinitely.
  - `grace_days`: Days before late fee penalty triggers.
- **Relations**:
  - Connects `charge_type_id` $\rightarrow$ `charge_types.id`.
  - `ChargeConfigs` (1) $\rightarrow$ (Many) `ChargeConfigMappings`.
  - `ChargeConfigs` (1) $\rightarrow$ (Many) `ChargeSlabs`.
  - `ChargeConfigs` (1) $\rightarrow$ (Many) `BillItems` (preserves historical audit logs).

#### 18. ChargeConfigMappings

- **Purpose**: Restricts configurations to target properties (e.g., Building A uses Tariff Rate A, Building B uses Tariff Rate B).
- **Relations**:
  - Connects `charge_config_id` $\rightarrow$ `charge_configs.id`.
  - Connects `building_id` $\rightarrow$ `buildings.id`. Enforces unique partial index: Only ONE active config of the same charge type per building at any time.

#### 19. ChargeSlabs

- **Purpose**: Tiered billing configurations (volumetric brackets).
- **Key Columns**:
  - `from_unit` & `to_unit`: The slab interval limits. The final slab must have `to_unit = NULL` (meaning unlimited).
  - `rate`: The cost per unit in this slab.
- **Relations**:
  - Connects `charge_config_id` $\rightarrow$ `charge_configs.id`.

### Domain 7: Billing, Payments & Double-Entry Financial Ledger

Manages monetary transactions and accounting records.

#### 20. Bills

- **Purpose**: Monthly invoices generated for a flat.
- **Key Columns**:
  - `bill_number`: Unique sequential format (e.g., `GAS-2026-000145`).
  - `total_usage`: Aggregated consumption.
  - `total_amount`: Pure monthly cost.
  - `previous_due`: Outstanding unpaid amount brought forward from last month.
  - `due_amount`: The payable invoice total (`total_amount + previous_due - customer.credit_balance`).
  - `status`: Payment condition (`UNPAID`, `PARTIALLY_PAID`, `PAID`).
- **Relations**:
  - Connects `flat_id` $\rightarrow$ `flats.id`. Enforces unique index `(flat_id, billing_month)` to prevent duplicate bill generation.
  - `Bills` (1) $\rightarrow$ (Many) `BillItems`.
  - `Bills` (1) $\rightarrow$ (Many) `Payments`.

#### 21. BillItems

- **Purpose**: Stores the itemized list of costs included in an invoice (e.g., Gas slab cost, Service Charge BDT 500, VAT BDT 505) for financial audit trails.
- **Relations**:
  - Connects `bill_id` $\rightarrow$ `bills.id`.
  - Connects `charge_config_id` $\rightarrow$ `charge_configs.id`.

#### 22. Payments

- **Purpose**: Records cash, bank transfers, or bKash collections.
- **Key Columns**:
  - `payment_method` & `reference_no`: Payment verification logs.
- **Relations**:
  - Connects `bill_id` $\rightarrow$ `bills.id`.
  - Connects `flat_id` $\rightarrow$ `flats.id`.
  - `Payments` (1) $\rightarrow$ (Many) `Transactions` (accounting).

#### 23. Transactions

- **Purpose**: The double-entry general financial ledger. Every cash transaction creates a bookkeeping log here.
- **Relations**:
  - Connects `reference_id` $\rightarrow$ `payments.id`.

### Domain 8: Supply Chain & Godowns Inventory

Manages hardware inventory (purchase and deployment of physical cylinders, pipes, regulators, and meters).

#### 24. Suppliers

- **Purpose**: B2B vendors of hardware parts.
- **Relations**:
  - `Suppliers` (1) $\rightarrow$ (Many) `MaterialPurchases`.

#### 25. Materials

- **Purpose**: Master product template list (e.g., `Gas Meter G1.6`, `Cylinder 45kg`).
- **Relations**:
  - `Materials` (1) $\rightarrow$ (Many) `MaterialPurchases`.

#### 26. MaterialPurchases

- **Purpose**: Bulk purchase logs (Stock In).
- **Key Columns**:
  - `quantity` & `unit_price`: Cost dynamics.
  - `additional_charges`: Customs or delivery fees.
  - `total_price`: Auto calculated.
  - `effective_unit_cost`: System-calculated average cost including shipping: `(quantity * unit_price + additional_charges) / quantity`.
- **Relations**:
  - Connects `material_id` $\rightarrow$ `materials.id`
  - Connects `supplier_id` $\rightarrow$ `suppliers.id`

#### 27. Stores

- **Purpose**: Central godowns or warehouse stores.
- **Relations**:
  - Connects `branch_id` $\rightarrow$ `branches.id`.
  - `Stores` (1) $\rightarrow$ (Many) `InventoryStock`.

#### 28. InventoryItems

- **Purpose**: The stock item master catalog.
- **Relations**:
  - `InventoryItems` (1) $\rightarrow$ (Many) `InventoryStock`.
  - `InventoryItems` (1) $\rightarrow$ (Many) `InventoryTransactions`.

#### 29. InventoryStock

- **Purpose**: Real-time stock counts at a specific warehouse, building, or branch.
- **Key Columns**:
  - `location_id` & `location_type`: Polymorphic logs to determine where parts reside (e.g., `location_type = 'STORE'`, `location_id = 1` or `location_type = 'BUILDING'`, `location_id = 18`).
- **Relations**:
  - Connects `item_id` $\rightarrow$ `inventory_items.id`.

#### 30. InventoryTransactions

- **Purpose**: Log of every stock transaction (purchasing `IN`, deploying `OUT`, or transfer between locations).
- **Key Columns**:
  - `from_location_id` & `to_location_id`: Traceability fields.
- **Relations**:
  - Connects `item_id` $\rightarrow$ `inventory_items.id`.

### Domain 9: System Settings, Audits & Communications

Communications and backend operations tracking.

#### 31. BillingSettings

- **Purpose**: Configures building-level invoice rules.
- **Key Columns**:
  - `billing_day`: The target date for monthly generation.
  - `due_days`: Days allowed for online checkout.
- **Relations**:
  - Connects `building_id` $\rightarrow$ `buildings.id`. Only one configuration per building.

#### 32. SMSSettings & 35. EmailSettings

- **Purpose**: Connection parameters for SMTP servers and SMS API gateways (e.g., Twilio API Key, SMTP Host).

#### 33. SMSTemplates

- **Purpose**: Pre-configured messages with placeholders (e.g., `Dear {customer_name}, your bill {bill_number} is ready.`) triggered by system events.

#### 34. NotificationSettings

- **Purpose**: Toggles global alerts (e.g., enable SMS billing notifications, disable email).

#### 36. GeneralSettings

- **Purpose**: key-value logs for UI configuration (e.g. `company_logo_url`).

#### 37. AuditLog

- **Purpose**: Tracks user actions to monitor security.
- **Key Columns**:
  - `action`: Operations category (`CREATE`, `UPDATE`, `DELETE`).
  - `entity_type` & `entity_id`: Record references.
  - `old_values` & `new_values` (`JSONB`): **Highly Important!** Keeps a strict diff of updated fields for audits.
- **Relations**:
  - Connects `user_id` $\rightarrow$ `users.id`.

#### 38. NotificationQueue

- **Purpose**: Stores scheduled outbound transactional SMS/Emails to prevent bottlenecks. Background queue workers process and send messages in batches.
- **Key Columns**:
  - `recipient`: Phone or email.
  - `status`: Dispatch state (`PENDING`, `SENT`, `FAILED`).
  - `retry_count`: Increments automatically on network errors.

### DB Connection Verdict

This relational model is highly robust, safe, and normalized. There are no redundant tables or loose foreign key connections. Enforcing cascading deletes on parents (tenants) while strictly blocking deletes on audit-trail rows (meters, flats, bills, charge_configs) ensures excellent financial database consistency!

# end xxx
