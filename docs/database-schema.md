# Inventory Management System - Database Schema and Migration Plan

## 1. Purpose

This document defines the database schema and migration plan for a Laravel-based Inventory Management System with Point of Sale (POS) integration. The schema is designed to support configurable inventory workflows, supplier and purchase management, sales order processing, POS transactions, auditability, and role-based access.

The design emphasizes:

- transactional inventory correctness
- auditability of all stock changes
- support for warehouses, locations, and batch tracking
- multi-user role-based operations
- POS transaction integrity
- reporting readiness and future extension

## 2. Design Principles

- One canonical source of truth for inventory quantity
- Inventory mutations must always be recorded in stock_movements
- Every critical change must be tracked in audit_logs
- POS sales and refunds must update inventory and stock movement history
- Use normalized relational tables for inventory logic
- Use JSON where dynamic business configuration is needed
- Support multi-warehouse and multi-location operations
- Keep migratable, versioned schema changes in Laravel migrations

## 3. Database Platform

Recommended database:

- PostgreSQL 14+

Why PostgreSQL:
- strong transactional integrity
- robust relational model for complex stock logic
- good reporting support
- better suited to inventory and audit-heavy systems than MySQL in large-scale operations

## 4. Migration Strategy

Follow Laravel migration sequence:

1. core system tables: users, roles, permissions, settings
2. product catalog tables
3. warehouse and inventory tables
4. supplier and procurement tables
5. sales and customer tables
6. POS tables
7. movement, alert, and audit tables
8. seeders and initial admin data

This order ensures dependencies are created before transaction tables are used.

## 5. Core Tables

### 5.1 Users, Roles, and Permissions

#### users

```sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    email_verified_at TIMESTAMP NULL,
    password VARCHAR(255) NOT NULL,
    remember_token VARCHAR(100) NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    deleted_at TIMESTAMP NULL
);
```

#### roles

```sql
CREATE TABLE roles (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    guard_name VARCHAR(255) NOT NULL DEFAULT 'web',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

#### permissions

```sql
CREATE TABLE permissions (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE,
    guard_name VARCHAR(255) NOT NULL DEFAULT 'web',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

#### model_has_roles

```sql
CREATE TABLE model_has_roles (
    role_id BIGINT NOT NULL,
    model_type VARCHAR(255) NOT NULL,
    model_id BIGINT NOT NULL,
    PRIMARY KEY (role_id, model_id, model_type)
);
```

#### model_has_permissions

```sql
CREATE TABLE model_has_permissions (
    permission_id BIGINT NOT NULL,
    model_type VARCHAR(255) NOT NULL,
    model_id BIGINT NOT NULL,
    PRIMARY KEY (permission_id, model_id, model_type)
);
```

#### settings

```sql
CREATE TABLE settings (
    id BIGSERIAL PRIMARY KEY,
    key VARCHAR(255) NOT NULL UNIQUE,
    value JSONB NULL,
    category VARCHAR(100) NOT NULL DEFAULT 'general',
    scope VARCHAR(100) NOT NULL DEFAULT 'global',
    scope_id BIGINT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

## 6. Product Catalog Tables

### 6.1 brands

```sql
CREATE TABLE brands (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

### 6.2 categories

```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    parent_id BIGINT NULL,
    slug VARCHAR(255) NOT NULL UNIQUE,
    description TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_categories_parent FOREIGN KEY (parent_id) REFERENCES categories(id)
);
```

### 6.3 products

```sql
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    sku VARCHAR(100) NOT NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    description TEXT NULL,
    category_id BIGINT NULL,
    brand_id BIGINT NULL,
    supplier_id BIGINT NULL,
    barcode VARCHAR(255) NULL UNIQUE,
    unit_of_measure VARCHAR(50) NULL,
    cost_price DECIMAL(15,2) NOT NULL DEFAULT 0,
    selling_price DECIMAL(15,2) NOT NULL DEFAULT 0,
    reorder_level DECIMAL(15,2) NOT NULL DEFAULT 0,
    safety_stock DECIMAL(15,2) NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    has_variants BOOLEAN NOT NULL DEFAULT FALSE,
    taxable BOOLEAN NOT NULL DEFAULT TRUE,
    metadata JSONB NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_products_category FOREIGN KEY (category_id) REFERENCES categories(id),
    CONSTRAINT fk_products_brand FOREIGN KEY (brand_id) REFERENCES brands(id)
);
```

### 6.4 product_variants

```sql
CREATE TABLE product_variants (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    sku VARCHAR(100) NOT NULL UNIQUE,
    barcode VARCHAR(255) NULL UNIQUE,
    name VARCHAR(255) NOT NULL,
    attributes JSONB NULL,
    cost_price DECIMAL(15,2) NOT NULL DEFAULT 0,
    selling_price DECIMAL(15,2) NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_product_variants_product FOREIGN KEY (product_id) REFERENCES products(id)
);
```

### 6.5 product_attributes

```sql
CREATE TABLE product_attributes (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    data_type VARCHAR(50) NOT NULL DEFAULT 'string',
    is_required BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

### 6.6 product_attribute_values

```sql
CREATE TABLE product_attribute_values (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NULL,
    variant_id BIGINT NULL,
    attribute_id BIGINT NOT NULL,
    value TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_pav_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_pav_variant FOREIGN KEY (variant_id) REFERENCES product_variants(id),
    CONSTRAINT fk_pav_attribute FOREIGN KEY (attribute_id) REFERENCES product_attributes(id)
);
```

## 7. Warehouse and Inventory Tables

### 7.1 warehouses

```sql
CREATE TABLE warehouses (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL UNIQUE,
    address TEXT NULL,
    phone VARCHAR(50) NULL,
    email VARCHAR(255) NULL,
    manager_id BIGINT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_warehouses_manager FOREIGN KEY (manager_id) REFERENCES users(id)
);
```

### 7.2 locations

```sql
CREATE TABLE locations (
    id BIGSERIAL PRIMARY KEY,
    warehouse_id BIGINT NOT NULL,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(50) NOT NULL,
    type VARCHAR(100) NOT NULL DEFAULT 'shelf',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_locations_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id)
);
```

### 7.3 inventory_items

This table holds the current quantity summary by product and warehouse/location.

```sql
CREATE TABLE inventory_items (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    variant_id BIGINT NULL,
    warehouse_id BIGINT NOT NULL,
    location_id BIGINT NULL,
    quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    reserved_quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    damaged_quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    quarantined_quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    available_quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    last_counted_at TIMESTAMP NULL,
    last_movement_at TIMESTAMP NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'in_stock',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_inventory_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_inventory_variant FOREIGN KEY (variant_id) REFERENCES product_variants(id),
    CONSTRAINT fk_inventory_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_inventory_location FOREIGN KEY (location_id) REFERENCES locations(id)
);
```

### 7.4 stock_movements

This is the critical inventory ledger table.

```sql
CREATE TABLE stock_movements (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    variant_id BIGINT NULL,
    warehouse_id BIGINT NOT NULL,
    location_id BIGINT NULL,
    movement_type VARCHAR(50) NOT NULL,
    quantity DECIMAL(15,2) NOT NULL,
    source_warehouse_id BIGINT NULL,
    source_location_id BIGINT NULL,
    destination_warehouse_id BIGINT NULL,
    destination_location_id BIGINT NULL,
    reference_type VARCHAR(100) NULL,
    reference_id BIGINT NULL,
    reason VARCHAR(255) NULL,
    notes TEXT NULL,
    performed_by BIGINT NOT NULL,
    batch_number VARCHAR(255) NULL,
    expiry_date DATE NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_stock_movements_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_stock_movements_variant FOREIGN KEY (variant_id) REFERENCES product_variants(id),
    CONSTRAINT fk_stock_movements_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_stock_movements_source_warehouse FOREIGN KEY (source_warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_stock_movements_dest_warehouse FOREIGN KEY (destination_warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_stock_movements_user FOREIGN KEY (performed_by) REFERENCES users(id)
);
```

### 7.5 stock_adjustments

```sql
CREATE TABLE stock_adjustments (
    id BIGSERIAL PRIMARY KEY,
    inventory_item_id BIGINT NOT NULL,
    adjustment_type VARCHAR(50) NOT NULL,
    quantity DECIMAL(15,2) NOT NULL,
    reason VARCHAR(255) NOT NULL,
    notes TEXT NULL,
    approved_by BIGINT NULL,
    created_by BIGINT NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_stock_adjustments_item FOREIGN KEY (inventory_item_id) REFERENCES inventory_items(id),
    CONSTRAINT fk_stock_adjustments_approved FOREIGN KEY (approved_by) REFERENCES users(id),
    CONSTRAINT fk_stock_adjustments_created FOREIGN KEY (created_by) REFERENCES users(id)
);
```

### 7.6 stock_counts

```sql
CREATE TABLE stock_counts (
    id BIGSERIAL PRIMARY KEY,
    warehouse_id BIGINT NOT NULL,
    counted_by BIGINT NOT NULL,
    counted_at TIMESTAMP NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft',
    notes TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_stock_counts_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_stock_counts_user FOREIGN KEY (counted_by) REFERENCES users(id)
);
```

### 7.7 stock_count_items

```sql
CREATE TABLE stock_count_items (
    id BIGSERIAL PRIMARY KEY,
    stock_count_id BIGINT NOT NULL,
    inventory_item_id BIGINT NOT NULL,
    expected_quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    counted_quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    variance DECIMAL(15,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_stock_count_items_count FOREIGN KEY (stock_count_id) REFERENCES stock_counts(id),
    CONSTRAINT fk_stock_count_items_inventory FOREIGN KEY (inventory_item_id) REFERENCES inventory_items(id)
);
```

## 8. Supplier and Procurement Tables

### 8.1 suppliers

```sql
CREATE TABLE suppliers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) NULL UNIQUE,
    contact_person VARCHAR(255) NULL,
    phone VARCHAR(50) NULL,
    email VARCHAR(255) NULL,
    address TEXT NULL,
    payment_terms VARCHAR(255) NULL,
    lead_time_days INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

### 8.2 purchase_orders

```sql
CREATE TABLE purchase_orders (
    id BIGSERIAL PRIMARY KEY,
    po_number VARCHAR(100) NOT NULL UNIQUE,
    supplier_id BIGINT NOT NULL,
    warehouse_id BIGINT NOT NULL,
    order_date TIMESTAMP NOT NULL,
    expected_delivery_date TIMESTAMP NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft',
    notes TEXT NULL,
    created_by BIGINT NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_purchases_supplier FOREIGN KEY (supplier_id) REFERENCES suppliers(id),
    CONSTRAINT fk_purchases_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_purchases_user FOREIGN KEY (created_by) REFERENCES users(id)
);
```

### 8.3 purchase_order_items

```sql
CREATE TABLE purchase_order_items (
    id BIGSERIAL PRIMARY KEY,
    purchase_order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    variant_id BIGINT NULL,
    quantity DECIMAL(15,2) NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL DEFAULT 0,
    received_quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_poi_purchase FOREIGN KEY (purchase_order_id) REFERENCES purchase_orders(id),
    CONSTRAINT fk_poi_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_poi_variant FOREIGN KEY (variant_id) REFERENCES product_variants(id)
);
```

### 8.4 goods_receipts

```sql
CREATE TABLE goods_receipts (
    id BIGSERIAL PRIMARY KEY,
    receipt_number VARCHAR(100) NOT NULL UNIQUE,
    purchase_order_id BIGINT NULL,
    warehouse_id BIGINT NOT NULL,
    received_by BIGINT NOT NULL,
    received_at TIMESTAMP NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'completed',
    notes TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_goods_receipts_po FOREIGN KEY (purchase_order_id) REFERENCES purchase_orders(id),
    CONSTRAINT fk_goods_receipts_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_goods_receipts_user FOREIGN KEY (received_by) REFERENCES users(id)
);
```

### 8.5 goods_receipt_items

```sql
CREATE TABLE goods_receipt_items (
    id BIGSERIAL PRIMARY KEY,
    goods_receipt_id BIGINT NOT NULL,
    purchase_order_item_id BIGINT NULL,
    product_id BIGINT NOT NULL,
    variant_id BIGINT NULL,
    quantity DECIMAL(15,2) NOT NULL,
    received_quantity DECIMAL(15,2) NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_gri_receipt FOREIGN KEY (goods_receipt_id) REFERENCES goods_receipts(id),
    CONSTRAINT fk_gri_poi FOREIGN KEY (purchase_order_item_id) REFERENCES purchase_order_items(id),
    CONSTRAINT fk_gri_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_gri_variant FOREIGN KEY (variant_id) REFERENCES product_variants(id)
);
```

## 9. Customer and Sales Tables

### 9.1 customers

```sql
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NULL,
    phone VARCHAR(50) NULL,
    address TEXT NULL,
    customer_type VARCHAR(50) NOT NULL DEFAULT 'retail',
    loyalty_points INTEGER NOT NULL DEFAULT 0,
    total_spent DECIMAL(15,2) NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);
```

### 9.2 sales_orders

```sql
CREATE TABLE sales_orders (
    id BIGSERIAL PRIMARY KEY,
    order_number VARCHAR(100) NOT NULL UNIQUE,
    customer_id BIGINT NULL,
    warehouse_id BIGINT NOT NULL,
    order_date TIMESTAMP NOT NULL,
    requested_delivery_date TIMESTAMP NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'draft',
    notes TEXT NULL,
    created_by BIGINT NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_sales_customer FOREIGN KEY (customer_id) REFERENCES customers(id),
    CONSTRAINT fk_sales_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_sales_user FOREIGN KEY (created_by) REFERENCES users(id)
);
```

### 9.3 sales_order_items

```sql
CREATE TABLE sales_order_items (
    id BIGSERIAL PRIMARY KEY,
    sales_order_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    variant_id BIGINT NULL,
    quantity DECIMAL(15,2) NOT NULL,
    reserved_quantity DECIMAL(15,2) NOT NULL DEFAULT 0,
    unit_price DECIMAL(15,2) NOT NULL DEFAULT 0,
    discount_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_soi_order FOREIGN KEY (sales_order_id) REFERENCES sales_orders(id),
    CONSTRAINT fk_soi_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_soi_variant FOREIGN KEY (variant_id) REFERENCES product_variants(id)
);
```

### 9.4 returns

```sql
CREATE TABLE returns (
    id BIGSERIAL PRIMARY KEY,
    return_number VARCHAR(100) NOT NULL UNIQUE,
    customer_id BIGINT NULL,
    sales_order_id BIGINT NULL,
    warehouse_id BIGINT NOT NULL,
    reason VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'pending',
    refund_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    created_by BIGINT NOT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_returns_customer FOREIGN KEY (customer_id) REFERENCES customers(id),
    CONSTRAINT fk_returns_order FOREIGN KEY (sales_order_id) REFERENCES sales_orders(id),
    CONSTRAINT fk_returns_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_returns_user FOREIGN KEY (created_by) REFERENCES users(id)
);
```

### 9.5 return_items

```sql
CREATE TABLE return_items (
    id BIGSERIAL PRIMARY KEY,
    return_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    variant_id BIGINT NULL,
    quantity DECIMAL(15,2) NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL DEFAULT 0,
    refund_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_return_items_return FOREIGN KEY (return_id) REFERENCES returns(id),
    CONSTRAINT fk_return_items_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_return_items_variant FOREIGN KEY (variant_id) REFERENCES product_variants(id)
);
```

## 10. POS Tables

### 10.1 pos_terminals

```sql
CREATE TABLE pos_terminals (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    code VARCHAR(100) NOT NULL UNIQUE,
    warehouse_id BIGINT NOT NULL,
    cashier_user_id BIGINT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'active',
    opening_balance DECIMAL(15,2) NOT NULL DEFAULT 0,
    expected_balance DECIMAL(15,2) NOT NULL DEFAULT 0,
    actual_balance DECIMAL(15,2) NOT NULL DEFAULT 0,
    last_reconciliation_at TIMESTAMP NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_pos_terminal_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_pos_terminal_cashier FOREIGN KEY (cashier_user_id) REFERENCES users(id)
);
```

### 10.2 pos_transactions

```sql
CREATE TABLE pos_transactions (
    id BIGSERIAL PRIMARY KEY,
    receipt_number VARCHAR(100) NOT NULL UNIQUE,
    pos_terminal_id BIGINT NOT NULL,
    cashier_id BIGINT NOT NULL,
    customer_id BIGINT NULL,
    transaction_date TIMESTAMP NOT NULL,
    subtotal DECIMAL(15,2) NOT NULL DEFAULT 0,
    discount_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    tax_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    total_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    payment_status VARCHAR(50) NOT NULL DEFAULT 'pending',
    status VARCHAR(50) NOT NULL DEFAULT 'completed',
    notes TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_pos_transaction_terminal FOREIGN KEY (pos_terminal_id) REFERENCES pos_terminals(id),
    CONSTRAINT fk_pos_transaction_cashier FOREIGN KEY (cashier_id) REFERENCES users(id),
    CONSTRAINT fk_pos_transaction_customer FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

### 10.3 pos_transaction_items

```sql
CREATE TABLE pos_transaction_items (
    id BIGSERIAL PRIMARY KEY,
    pos_transaction_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    variant_id BIGINT NULL,
    quantity DECIMAL(15,2) NOT NULL,
    unit_price DECIMAL(15,2) NOT NULL DEFAULT 0,
    discount_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    tax_amount DECIMAL(15,2) NOT NULL DEFAULT 0,
    line_total DECIMAL(15,2) NOT NULL DEFAULT 0,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_pos_item_transaction FOREIGN KEY (pos_transaction_id) REFERENCES pos_transactions(id),
    CONSTRAINT fk_pos_item_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_pos_item_variant FOREIGN KEY (variant_id) REFERENCES product_variants(id)
);
```

### 10.4 payments

```sql
CREATE TABLE payments (
    id BIGSERIAL PRIMARY KEY,
    pos_transaction_id BIGINT NULL,
    sales_order_id BIGINT NULL,
    payment_method VARCHAR(50) NOT NULL,
    payment_reference VARCHAR(255) NULL,
    amount DECIMAL(15,2) NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'completed',
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_payments_pos_transaction FOREIGN KEY (pos_transaction_id) REFERENCES pos_transactions(id),
    CONSTRAINT fk_payments_sales_order FOREIGN KEY (sales_order_id) REFERENCES sales_orders(id)
);
```

### 10.5 pos_reconciliation

```sql
CREATE TABLE pos_reconciliation (
    id BIGSERIAL PRIMARY KEY,
    pos_terminal_id BIGINT NOT NULL,
    reconciled_by BIGINT NOT NULL,
    reconciliation_date TIMESTAMP NOT NULL,
    opening_balance DECIMAL(15,2) NOT NULL DEFAULT 0,
    expected_cash DECIMAL(15,2) NOT NULL DEFAULT 0,
    actual_cash DECIMAL(15,2) NOT NULL DEFAULT 0,
    variance DECIMAL(15,2) NOT NULL DEFAULT 0,
    notes TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_pos_reconciliation_terminal FOREIGN KEY (pos_terminal_id) REFERENCES pos_terminals(id),
    CONSTRAINT fk_pos_reconciliation_user FOREIGN KEY (reconciled_by) REFERENCES users(id)
);
```

## 11. Alert and Audit Tables

### 11.1 alerts

```sql
CREATE TABLE alerts (
    id BIGSERIAL PRIMARY KEY,
    alert_type VARCHAR(100) NOT NULL,
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    product_id BIGINT NULL,
    warehouse_id BIGINT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'open',
    triggered_by BIGINT NULL,
    acknowledged_by BIGINT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_alerts_product FOREIGN KEY (product_id) REFERENCES products(id),
    CONSTRAINT fk_alerts_warehouse FOREIGN KEY (warehouse_id) REFERENCES warehouses(id),
    CONSTRAINT fk_alerts_trigger_user FOREIGN KEY (triggered_by) REFERENCES users(id),
    CONSTRAINT fk_alerts_ack_user FOREIGN KEY (acknowledged_by) REFERENCES users(id)
);
```

### 11.2 audit_logs

```sql
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NULL,
    action VARCHAR(255) NOT NULL,
    model_type VARCHAR(255) NOT NULL,
    model_id BIGINT NOT NULL,
    old_values JSONB NULL,
    new_values JSONB NULL,
    ip_address VARCHAR(45) NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL,
    CONSTRAINT fk_audit_logs_user FOREIGN KEY (user_id) REFERENCES users(id)
);
```

## 12. Recommended Indexes

Critical indexes for performance and lookup speed:

```sql
CREATE INDEX idx_products_sku ON products (sku);
CREATE INDEX idx_products_barcode ON products (barcode);
CREATE INDEX idx_products_category ON products (category_id);
CREATE INDEX idx_products_supplier ON products (supplier_id);

CREATE INDEX idx_inventory_product_warehouse ON inventory_items (product_id, warehouse_id);
CREATE INDEX idx_inventory_warehouse_location ON inventory_items (warehouse_id, location_id);
CREATE INDEX idx_inventory_status ON inventory_items (status);

CREATE INDEX idx_stock_movements_product_date ON stock_movements (product_id, created_at);
CREATE INDEX idx_stock_movements_warehouse_date ON stock_movements (warehouse_id, created_at);
CREATE INDEX idx_stock_movements_type ON stock_movements (movement_type);
CREATE INDEX idx_stock_movements_reference ON stock_movements (reference_type, reference_id);

CREATE INDEX idx_purchase_orders_supplier ON purchase_orders (supplier_id);
CREATE INDEX idx_purchase_orders_status ON purchase_orders (status);

CREATE INDEX idx_sales_orders_customer ON sales_orders (customer_id);
CREATE INDEX idx_sales_orders_status ON sales_orders (status);

CREATE INDEX idx_pos_transactions_terminal_date ON pos_transactions (pos_terminal_id, transaction_date);
CREATE INDEX idx_pos_transactions_cashier ON pos_transactions (cashier_id);
CREATE INDEX idx_pos_transactions_status ON pos_transactions (status);

CREATE INDEX idx_audit_logs_model ON audit_logs (model_type, model_id);
CREATE INDEX idx_alerts_status ON alerts (status, alert_type);
```

## 13. Relationship Overview

### 13.1 Product to Inventory Relationship

- `products` has many `product_variants`
- `products` has many `inventory_items`
- `inventory_items` belong to one `product`
- `inventory_items` belong to one `warehouse`
- `inventory_items` may belong to a `location`

### 13.2 Purchase Flow Relationship

- `suppliers` have many `purchase_orders`
- `purchase_orders` have many `purchase_order_items`
- `purchase_order_items` belong to `products`
- `goods_receipts` receive ordered goods
- `goods_receipt_items` record specific received quantities

### 13.3 Sales and POS Relationship

- `customers` have many `sales_orders`
- `sales_orders` have many `sales_order_items`
- `pos_transactions` may reference a `customer`
- `pos_transactions` have many `pos_transaction_items`
- `payments` belong to a `pos_transaction` or sales order
- `returns` link to original sales order or POS transaction

### 13.4 Stock Ledger Relationship

- Every stock mutation creates a `stock_movements` row
- `inventory_items` are current summary records
- `stock_adjustments` record manual corrections
- `audit_logs` capture who changed what and when

## 14. Inventory Calculation Rules

Current stock totals should be computed from the aggregate of inventory_items and validated against stock movement history.

Suggested fields in `inventory_items`:
- `quantity` = raw on-hand count after movements
- `reserved_quantity` = allocated but not fulfilled quantity
- `available_quantity` = quantity - reserved - damaged - quarantined

Rule:

```text
available_quantity = quantity - reserved_quantity - damaged_quantity - quarantined_quantity
```

This should be recalculated whenever stock changes occur and should also be cross-checked against the movement log in reporting.

## 15. POS Sale Integrity Rules

### Rule 1: Finalized POS transaction reduces stock
When a POS sale is completed successfully:
- create pos_transaction record
- create pos_transaction_items for each product
- create stock movement with movement_type = 'sale'
- reduce inventory_item.quantity for the relevant warehouse
- record payment(s)
- create audit_log entry

### Rule 2: Refund increases stock
When a refund is processed:
- create a refund or return row
- reverse original movement or create stock movement with movement_type = 'return_in'
- increase inventory quantity accordingly
- create audit log

### Rule 3: Void or failed sale should not affect stock
If transaction is voided before finalization:
- do not decrease inventory
- cancel pending stock reservation
- log void event

## 16. Laravel Migration Organization

In a Laravel app, create migrations in order:

1. create_users_table
2. create_roles_table
3. create_permissions_table
4. create_settings_table
5. create_brands_table
6. create_categories_table
7. create_products_table
8. create_product_variants_table
9. create_product_attributes_table
10. create_product_attribute_values_table
11. create_warehouses_table
12. create_locations_table
13. create_inventory_items_table
14. create_stock_movements_table
15. create_stock_adjustments_table
16. create_stock_counts_table
17. create_stock_count_items_table
18. create_suppliers_table
19. create_purchase_orders_table
20. create_purchase_order_items_table
21. create_goods_receipts_table
22. create_goods_receipt_items_table
23. create_customers_table
24. create_sales_orders_table
25. create_sales_order_items_table
26. create_returns_table
27. create_return_items_table
28. create_pos_terminals_table
29. create_pos_transactions_table
30. create_pos_transaction_items_table
31. create_payments_table
32. create_pos_reconciliation_table
33. create_alerts_table
34. create_audit_logs_table

## 17. Example Laravel Migration Pattern

Example migration for inventory_items:

```php
Schema::create('inventory_items', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('product_id');
    $table->unsignedBigInteger('variant_id')->nullable();
    $table->unsignedBigInteger('warehouse_id');
    $table->unsignedBigInteger('location_id')->nullable();
    $table->decimal('quantity', 15, 2)->default(0);
    $table->decimal('reserved_quantity', 15, 2)->default(0);
    $table->decimal('damaged_quantity', 15, 2)->default(0);
    $table->decimal('quarantined_quantity', 15, 2)->default(0);
    $table->decimal('available_quantity', 15, 2)->default(0);
    $table->timestamp('last_counted_at')->nullable();
    $table->timestamp('last_movement_at')->nullable();
    $table->string('status')->default('in_stock');
    $table->timestamps();

    $table->foreign('product_id')->references('id')->on('products');
    $table->foreign('variant_id')->references('id')->on('product_variants');
    $table->foreign('warehouse_id')->references('id')->on('warehouses');
    $table->foreign('location_id')->references('id')->on('locations');
});
```

Example migration for stock_movements:

```php
Schema::create('stock_movements', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('product_id');
    $table->unsignedBigInteger('variant_id')->nullable();
    $table->unsignedBigInteger('warehouse_id');
    $table->unsignedBigInteger('location_id')->nullable();
    $table->string('movement_type');
    $table->decimal('quantity', 15, 2);
    $table->unsignedBigInteger('source_warehouse_id')->nullable();
    $table->unsignedBigInteger('source_location_id')->nullable();
    $table->unsignedBigInteger('destination_warehouse_id')->nullable();
    $table->unsignedBigInteger('destination_location_id')->nullable();
    $table->string('reference_type')->nullable();
    $table->unsignedBigInteger('reference_id')->nullable();
    $table->string('reason')->nullable();
    $table->text('notes')->nullable();
    $table->unsignedBigInteger('performed_by');
    $table->string('batch_number')->nullable();
    $table->date('expiry_date')->nullable();
    $table->timestamps();

    $table->foreign('product_id')->references('id')->on('products');
    $table->foreign('warehouse_id')->references('id')->on('warehouses');
    $table->foreign('source_warehouse_id')->references('id')->on('warehouses');
    $table->foreign('destination_warehouse_id')->references('id')->on('warehouses');
    $table->foreign('performed_by')->references('id')->on('users');
});
```

## 18. Laravel Models to Create

Recommended models:

- User
- Role
- Permission
- Brand
- Category
- Product
- ProductVariant
- ProductAttribute
- ProductAttributeValue
- Warehouse
- Location
- InventoryItem
- StockMovement
- StockAdjustment
- StockCount
- StockCountItem
- Supplier
- PurchaseOrder
- PurchaseOrderItem
- GoodsReceipt
- GoodsReceiptItem
- Customer
- SalesOrder
- SalesOrderItem
- ReturnModel
- ReturnItem
- PosTerminal
- PosTransaction
- PosTransactionItem
- Payment
- PosReconciliation
- Alert
- AuditLog
- Setting

## 19. Recommended Relationships in Laravel Models

Examples:

```php
class Product extends Model
{
    public function inventoryItems()
    {
        return $this->hasMany(InventoryItem::class);
    }

    public function variants()
    {
        return $this->hasMany(ProductVariant::class);
    }
}
```

```php
class InventoryItem extends Model
{
    public function product()
    {
        return $this->belongsTo(Product::class);
    }

    public function warehouse()
    {
        return $this->belongsTo(Warehouse::class);
    }

    public function location()
    {
        return $this->belongsTo(Location::class);
    }
}
```

```php
class PosTransaction extends Model
{
    public function items()
    {
        return $this->hasMany(PosTransactionItem::class);
    }

    public function payments()
    {
        return $this->hasMany(Payment::class);
    }
}
```

## 20. Seeders and Initial Data

Recommended seeders:

- AdminUserSeeder
- RolePermissionSeeder
- DefaultWarehouseSeeder
- DefaultCategorySeeder
- TaxAndSettingsSeeder
- SampleSupplierSeeder
- SampleProductSeeder
- SampleCustomerSeeder

## 21. Alternative Schema Notes

If the project later grows into a full ERP environment, this schema can be extended with:

- unit conversion tables
- serial numbers
- shipping integration tables
- batch traceability tables
- supplier invoices
- accounting journal entries
- employee payroll modules

## 22. Migration Execution Notes

Use:

```bash
php artisan make:migration create_products_table
php artisan migrate
php artisan db:seed
```

Production guidelines:
- always use migrations for schema changes
- avoid editing existing tables manually in production
- back up data before altering modified tables
- test migrations in staging before production deployment

## 23. Summary

This schema gives the project a solid transactional foundation for:

- product management
- configurable inventory
- warehouse operations
- supplier procurement
- POS retail and sales operations
- stock movement tracking
- auditability and reporting

It is intentionally designed to support a Laravel modular monolith while remaining extensible for future business growth.

---

Prepared for the Inventory Management System project.
