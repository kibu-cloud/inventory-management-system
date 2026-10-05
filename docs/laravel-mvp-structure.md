# Laravel MVP Structure for Inventory Management System

## 1. Overview

This document defines the Laravel module structure and implementation approach for an inventory management system with POS support. The project is organized as a modular monolith to keep the codebase manageable while still supporting a clean separation of responsibilities.

## 2. Recommended Laravel Architecture

Use the following structure:

```text
app/
  Console/
    Kernel.php
  Exceptions/
    Handler.php
  Http/
    Controllers/
      Admin/
      Inventory/
      Purchases/
      Sales/
      Pos/
      Auth/
      Reports/
    Middleware/
    Requests/
      Inventory/
      Purchases/
      Sales/
      Pos/
  Models/
    User.php
    Brand.php
    Category.php
    Product.php
    ProductVariant.php
    Warehouse.php
    Location.php
    InventoryItem.php
    StockMovement.php
    Supplier.php
    PurchaseOrder.php
    PurchaseOrderItem.php
    GoodsReceipt.php
    GoodsReceiptItem.php
    Customer.php
    SalesOrder.php
    SalesOrderItem.php
    PosTerminal.php
    PosTransaction.php
    PosTransactionItem.php
    Payment.php
    ReturnItem.php
    ReturnRecord.php
    Alert.php
    AuditLog.php
    Setting.php
  Services/
    InventoryService.php
    ProcurementService.php
    SalesService.php
    PosService.php
    ReportingService.php
    AlertService.php
    AuditService.php
    ConfigurationService.php
  Policies/
    ProductPolicy.php
    InventoryPolicy.php
    PurchaseOrderPolicy.php
    SalesOrderPolicy.php
    PosTransactionPolicy.php
  Jobs/
    ProcessLowStockAlert.php
    GenerateDailySalesReport.php
    SendInventoryNotification.php
  Notifications/
    LowStockAlertNotification.php
    DailySummaryNotification.php
  Events/
    InventoryUpdated.php
    PosTransactionCompleted.php
  Listeners/
    SyncInventoryAlertListener.php
    LogAuditListener.php
  Providers/
    AppServiceProvider.php
    AuthServiceProvider.php
    EventServiceProvider.php
    RouteServiceProvider.php
  Helpers/
    InventoryHelper.php
    Formatter.php

config/
  app.php
  auth.php
  database.php
  queue.php
  services.php
  settings.php

database/
  factories/
  migrations/
    2014_10_12_000000_create_users_table.php
    2024_01_01_000001_create_roles_table.php
    2024_01_01_000002_create_permissions_table.php
    2024_01_01_000003_create_model_has_roles_table.php
    2024_01_01_000004_create_model_has_permissions_table.php
    2024_01_01_000005_create_settings_table.php
    2024_01_01_000006_create_brands_table.php
    2024_01_01_000007_create_categories_table.php
    2024_01_01_000008_create_products_table.php
    2024_01_01_000009_create_product_variants_table.php
    2024_01_01_000010_create_product_attributes_table.php
    2024_01_01_000011_create_product_attribute_values_table.php
    2024_01_01_000012_create_warehouses_table.php
    2024_01_01_000013_create_locations_table.php
    2024_01_01_000014_create_inventory_items_table.php
    2024_01_01_000015_create_stock_movements_table.php
    2024_01_01_000016_create_stock_adjustments_table.php
    2024_01_01_000017_create_stock_counts_table.php
    2024_01_01_000018_create_stock_count_items_table.php
    2024_01_01_000019_create_suppliers_table.php
    2024_01_01_000020_create_purchase_orders_table.php
    2024_01_01_000021_create_purchase_order_items_table.php
    2024_01_01_000022_create_goods_receipts_table.php
    2024_01_01_000023_create_goods_receipt_items_table.php
    2024_01_01_000024_create_customers_table.php
    2024_01_01_000025_create_sales_orders_table.php
    2024_01_01_000026_create_sales_order_items_table.php
    2024_01_01_000027_create_returns_table.php
    2024_01_01_000028_create_return_items_table.php
    2024_01_01_000029_create_pos_terminals_table.php
    2024_01_01_000030_create_pos_transactions_table.php
    2024_01_01_000031_create_pos_transaction_items_table.php
    2024_01_01_000032_create_payments_table.php
    2024_01_01_000033_create_pos_reconciliation_table.php
    2024_01_01_000034_create_alerts_table.php
    2024_01_01_000035_create_audit_logs_table.php
  seeders/
    RolesAndPermissionsSeeder.php
    UserSeeder.php
    CategorySeeder.php
    WarehouseSeeder.php
    SupplierSeeder.php
    ProductSeeder.php
    CustomerSeeder.php

routes/
  api.php
  web.php

resources/
  views/
    admin/
      dashboard.blade.php
      products/
      inventory/
      purchases/
      sales/
      pos/
      reports/
    auth/
      login.blade.php
      register.blade.php
    layouts/
      app.blade.php
  js/
    app.js
    pos.js
  css/
    app.css

public/
  images/
  receipts/

storage/
  app/
  logs/
  framework/

phpunit.xml
.env.example
composer.json
artisan
```

## 3. Module Responsibilities

### 3.1 Inventory Module

Responsibilities:
- manage product stock by warehouse/location
- create and validate stock movements
- calculate available stock
- manage stock counts
- process adjustments and transfers
- define stock statuses and alerts

### 3.2 Procurement Module

Responsibilities:
- supplier management
- purchase order creation and tracking
- goods receiving
- purchase order status updates
- inventory increases from inbound goods

### 3.3 Sales Module

Responsibilities:
- customer creation and tracking
- sales order creation
- order status updates
- fulfillment and returns
- sales analytics

### 3.4 POS Module

Responsibilities:
- cashier login and terminal management
- cart and item scanning
- pricing and discount handling
- payment processing
- transaction finalization
- refund flow
- daily close and reconciliation

### 3.5 Reporting Module

Responsibilities:
- low-stock dashboards
- daily POS reports
- inventory value and aging
- fast/slow moving products
- supplier analytics

### 3.6 Admin Module

Responsibilities:
- users and roles
- settings and configuration
- warehouse setup
- custom field management
- permission assignment

## 4. Laravel Core Service Pattern

Use a service-based architecture.

Example service pattern:

```php
class InventoryService
{
    public function addStock(Product $product, Warehouse $warehouse, $quantity, $reason, $performedBy, $locationId = null)
    {
        // validate
        // begin transaction
        // update inventory_item
        // create stock_movement
        // create audit log
        // commit
    }

    public function reduceStock(Product $product, Warehouse $warehouse, $quantity, $reason, $performedBy, $locationId = null)
    {
        // validate available quantity
        // begin transaction
        // decrement inventory_item
        // create stock movement
        // create audit log
        // commit
    }
}
```

This design keeps all inventory logic in one place and prevents duplication across modules.

## 5. Inventory Service Rules

The service layer must enforce these rules:

- no stock reduction below zero without config exception
- all inventory changes create stock movement records
- all changes are performed in a database transaction
- all actions are logged in audit_logs
- low-stock checks occur after changes

## 6. POS Logic in Laravel

The POS module should process sales like this:

```php
public function completeSale(array $items, array $payment, User $cashier, PosTerminal $terminal): PosTransaction
{
    return DB::transaction(function () use ($items, $payment, $cashier, $terminal) {
        $transaction = PosTransaction::create([...]);

        foreach ($items as $item) {
            $product = Product::findOrFail($item['product_id']);
            $inventory = InventoryService::reduceStock(
                $product,
                $terminal->warehouse,
                $item['quantity'],
                'pos_sale',
                $cashier->id,
                $terminal->default_location_id ?? null
            );

            PosTransactionItem::create([
                'pos_transaction_id' => $transaction->id,
                'product_id' => $product->id,
                'quantity' => $item['quantity'],
                'unit_price' => $item['unit_price'],
                'discount_amount' => $item['discount_amount'] ?? 0,
                'tax_amount' => $item['tax_amount'] ?? 0,
                'line_total' => $item['line_total'],
            ]);
        }

        Payment::create([...]);
        AuditLog::create([...]);

        return $transaction;
    });
}
```

## 7. Recommended Controller Groups

### Inventory Controllers
- InventoryDashboardController
- InventoryItemController
- StockMovementController
- StockAdjustmentController
- StockCountController

### Procurement Controllers
- SupplierController
- PurchaseOrderController
- GoodsReceiptController

### Sales Controllers
- SalesOrderController
- CustomerController
- ReturnController

### POS Controllers
- PosTerminalController
- PosTransactionController
- PosReconciliationController

### Admin Controllers
- UserController
- RoleController
- SettingController
- WarehouseController

### Reporting Controllers
- DashboardReportController
- InventoryReportController
- PosReportController
- SupplierPerformanceController

## 8. Recommended Routes

### API routes

```php
Route::middleware('auth:sanctum')->group(function () {
    Route::resource('products', ProductController::class);
    Route::resource('warehouses', WarehouseController::class);
    Route::resource('inventory-items', InventoryItemController::class);
    Route::resource('purchase-orders', PurchaseOrderController::class);
    Route::resource('goods-receipts', GoodsReceiptController::class);
    Route::resource('sales-orders', SalesOrderController::class);
    Route::resource('customers', CustomerController::class);
    Route::resource('pos-terminals', PosTerminalController::class);
    Route::resource('pos-transactions', PosTransactionController::class);
    Route::get('reports/inventory', [InventoryReportController::class, 'index']);
    Route::get('reports/pos', [PosReportController::class, 'index']);
});
```

### Web routes

```php
Route::middleware(['auth'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index'])->name('dashboard');
    Route::get('/inventory', [InventoryDashboardController::class, 'index'])->name('inventory.index');
    Route::get('/pos', [PosDashboardController::class, 'index'])->name('pos.index');
    Route::get('/purchases', [PurchaseOrderController::class, 'index'])->name('purchases.index');
    Route::get('/sales', [SalesOrderController::class, 'index'])->name('sales.index');
    Route::get('/reports', [ReportController::class, 'index'])->name('reports.index');
});
```

## 9. Suggested Model Behaviors

### Product
- belongsTo category
- belongsTo brand
- belongsTo supplier
- hasMany variants
- hasMany inventoryItems

### InventoryItem
- belongsTo product
- belongsTo warehouse
- belongsTo location
- hasMany stockMovements

### StockMovement
- belongsTo product
- belongsTo warehouse
- belongsTo user (performed_by)

### PosTransaction
- belongsTo terminal
- belongsTo cashier user
- belongsTo customer
- hasMany items
- hasMany payments

### PurchaseOrder
- belongsTo supplier
- belongsTo warehouse
- belongsTo user
- hasMany items

## 10. Suggested Validation Patterns

Use Form Requests for input validation.

Examples:
- StoreProductRequest
- UpdateProductRequest
- StorePurchaseOrderRequest
- StorePosTransactionRequest
- AdjustInventoryRequest
- RefundRequest

This keeps controller methods simpler and validates business rules consistently.

## 11. Laravel Permissions

Using Spatie Permission, define roles and permissions like:

```php
Role::create(['name' => 'admin']);
Role::create(['name' => 'inventory_manager']);
Role::create(['name' => 'procurement']);
Role::create(['name' => 'warehouse_staff']);
Role::create(['name' => 'sales_manager']);
Role::create(['name' => 'cashier']);
Role::create(['name' => 'auditor']);
```

Example permissions:
- view_inventory
- create_purchase_order
- receive_goods
- save_stock_adjustment
- process_pos_sale
- void_pos_transaction
- view_reports
- manage_users

## 12. Recommended Seeder Flow

Use a structured seeder flow:

1. `RolesAndPermissionsSeeder`
2. `UserSeeder`
3. `WarehouseSeeder`
4. `SupplierSeeder`
5. `CategorySeeder`
6. `ProductSeeder`
7. `CustomerSeeder`

## 13. Priority MVP Features

The initial MVP should include:

- product catalog
- warehouse/location management
- stock movements and inventory summary
- purchase orders and receiving
- sales orders and POS sales
- returns and refunds
- low-stock alerts
- basic reports
- role-based access

## 14. Recommended Development Sequence

### Phase 1: Build core modules
- users and permissions
- products
- warehouses and inventory items
- suppliers and purchase orders
- stock movements

### Phase 2: Add POS
- terminals
- POS transactions
- payments
- ticket/receipt generation
- end-of-day reconciliation

### Phase 3: Add analytics
- dashboards
- low-stock widgets
- stock aging reports
- sales reports

### Phase 4: Improve operations
- imports/exports
- barcode scanning
- advanced discount engine
- notifications and reminders

## 15. Implementation Notes

- Keep business logic out of controllers
- Validate all requests using Form Requests
- Put inventory write logic in service classes only
- Keep audit and alert generation after successful transaction commits
- Use queues for alert dispatch and large reporting jobs
- Use database transactions on all stock-affecting flows

## 16. Summary

This Laravel MVP structure gives you a clean foundation for a configurable inventory and POS system. It balances speed, maintainability, and correctness while keeping the system modular enough for future expansion.

The most important architectural rule is:

Inventory must be managed by one central service layer using a transactional ledger model.

---

Prepared for the Inventory Management System project.
