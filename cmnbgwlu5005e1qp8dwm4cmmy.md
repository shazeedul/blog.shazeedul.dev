---
title: "Modular Monolith with Clean Architecture."
seoTitle: "Modular Monolith Clean Architecture in Laravel: A Practical "
seoDescription: "Learn how to build a modular monolith with clean architecture in Laravel. Explore folder structure, module boundaries, shared components, migrations, "
datePublished: 2026-03-29T07:58:46.930Z
cuid: cmnbgwlu5005e1qp8dwm4cmmy
slug: modular-monolith-with-clean-architecture
cover: https://cdn.hashnode.com/uploads/covers/668d00ea32e1643b75a2d676/4ad71f39-ea8b-4740-9195-d72f27e60ce2.png
ogImage: https://cdn.hashnode.com/uploads/og-images/668d00ea32e1643b75a2d676/13ea3220-cea5-473d-a2fd-2647ae0da4fc.png
tags: clean-architecture, architecture-design

---

# Modular Monolith with Clean Architecture in Laravel

A practical implementation-level example for building a **modular monolith** using **clean architecture** in **Laravel/PHP**.

This document is written to be shared with engineers who want a concrete reference for structuring modules, separating business logic from frameworks, and keeping inter-module boundaries clean.

## 1. Goal

This architecture aims to achieve:

- **one deployable application**
- **multiple business modules**
- **clear code ownership per module**
- **framework-independent domain logic**
- **controlled communication between modules**

This is a **modular monolith**, not microservices.

That means:

- one Laravel application
- one deployment unit
- usually one database
- modules separated by code boundaries, not network boundaries

## 2. Core Architecture Rules

### Rule 1: Keep business logic away from Laravel details

Business rules should not depend directly on:

- controllers
- requests
- Eloquent models
- facades
- route files

### Rule 2: Modules talk through contracts

A module should not directly use another module's internal database models or repositories.

Use:
- interfaces
- gateways
- application contracts
- domain events when needed

### Rule 3: Dependency direction goes inward

```text
Presentation -> Application -> Domain
Infrastructure -> Application / Domain
```

The domain should be the most stable and framework-independent part.

## 3. Example Business Flow

We will model this use case:

**Place Order**
Flow:
1. User sends an order request
2. Orders module validates input
3. Orders use case builds the order
4. Orders asks Inventory to reserve stock
5. Inventory checks and decrements stock
6. Orders saves the order
7. API returns the created order

## 4. Suggested Folder Structure

```text
app/
  Modules/
    Orders/
      Domain/
        Entities/
          Order.php
          OrderItem.php
        Repositories/
          OrderRepositoryInterface.php
        Services/
          InventoryGatewayInterface.php
        Exceptions/
          InsufficientStockException.php

      Application/
        DTOs/
          PlaceOrderCommand.php
        UseCases/
          PlaceOrder.php

      Infrastructure/
        Persistence/
          Eloquent/
            Models/
              OrderModel.php
              OrderItemModel.php
            Repositories/
              EloquentOrderRepository.php
        Providers/
          OrdersServiceProvider.php

      Presentation/
        Http/
          Controllers/
            OrderController.php
          Requests/
            PlaceOrderRequest.php
          Resources/
            OrderResource.php
        Routes/
          api.php

      Database/
        Migrations/
          2026_03_29_000001_create_orders_table.php
          2026_03_29_000002_create_order_items_table.php
        Factories/
        Seeders/

    Inventory/
      Domain/
        Repositories/
          ProductStockRepositoryInterface.php

      Application/
        Services/
          ReserveStockService.php

      Infrastructure/
        Persistence/
          Eloquent/
            Models/
              ProductStockModel.php
            Repositories/
              EloquentProductStockRepository.php
        Services/
          StockReservationService.php
        Providers/
          InventoryServiceProvider.php

      Database/
        Migrations/
          2026_03_29_000003_create_product_stocks_table.php

    Shared/
      Domain/
        ValueObjects/
          Money.php
      Application/
        Contracts/
          ClockInterface.php
```

## 5. What Each Layer Does

### Domain

Contains pure business concepts and rules.

Examples:

- entities
- value objects
- repository interfaces
- business exceptions
- gateway contracts

Domain should not know Laravel exists.

### Application

Contains use cases.

Examples:

- `PlaceOrder`
- `CancelOrder`
- `RefundPayment`

Application orchestrates business flow using domain contracts.

### Infrastructure

Contains Laravel and persistence details.

Examples:

- Eloquent models
- repository implementations
- service provider bindings
- adapter implementations

### Presentation

Contains delivery-layer code.

Examples:

- controllers
- form requests
- API resources
- routes

## 6. Orders Module Implementation

### 6.1 Domain Entity: `Order`

```php
<?php

namespace App\Modules\Orders\Domain\Entities;

class Order
{
    private ?int $id;
    private int $userId;
    private string $status;

    /** @var OrderItem[] */
    private array $items = [];

    public function __construct(?int $id, int $userId, array $items, string $status = 'pending')
    {
        $this->id = $id;
        $this->userId = $userId;
        $this->items = $items;
        $this->status = $status;
    }

    public static function create(int $userId, array $items): self
    {
        if (empty($items)) {
            throw new \InvalidArgumentException('Order must have at least one item.');
        }

        return new self(
            id: null,
            userId: $userId,
            items: $items,
            status: 'pending',
        );
    }

    public function confirm(): void
    {
        $this->status = 'confirmed';
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setId(int $id): void
    {
        $this->id = $id;
    }

    public function getUserId(): int
    {
        return $this->userId;
    }

    public function getStatus(): string
    {
        return $this->status;
    }

    /** @return OrderItem[] */
    public function getItems(): array
    {
        return $this->items;
    }
}
```

### 6.2 Domain Entity: `OrderItem`

```php
<?php

namespace App\Modules\Orders\Domain\Entities;

class OrderItem
{
    public function __construct(
        private int $productId,
        private int $quantity,
        private int $unitPrice,
    ) {
        if ($quantity < 1) {
            throw new \InvalidArgumentException('Quantity must be at least 1.');
        }
    }

    public function getProductId(): int
    {
        return $this->productId;
    }

    public function getQuantity(): int
    {
        return $this->quantity;
    }

    public function getUnitPrice(): int
    {
        return $this->unitPrice;
    }

    public function getSubtotal(): int
    {
        return $this->quantity * $this->unitPrice;
    }
}
```

### 6.3 Repository Contract

```php
<?php

namespace App\Modules\Orders\Domain\Repositories;

use App\Modules\Orders\Domain\Entities\Order;

interface OrderRepositoryInterface
{
    public function save(Order $order): Order;

    public function findById(int $id): ?Order;
}
```

### 6.4 Cross-Module Contract: Inventory Gateway

Orders should not directly use Inventory Eloquent models.

Instead, it depends on an abstraction.

```php
<?php

namespace App\Modules\Orders\Domain\Services;

interface InventoryGatewayInterface
{
    /**
     * @param array<int, array{product_id:int, quantity:int}> $items
     */
    public function reserve(array $items): void;
}
```

## 7. Orders Application Layer

### 7.1 Command DTO

```php
<?php

namespace App\Modules\Orders\Application\DTOs;

class PlaceOrderCommand
{
    /**
     * @param array<int, array{product_id:int, quantity:int, unit_price:int}> $items
     */
    public function __construct(
        public readonly int $userId,
        public readonly array $items,
    ) {}
}
```

### 7.2 Use Case: `PlaceOrder`

```php
<?php

namespace App\Modules\Orders\Application\UseCases;

use App\Modules\Orders\Application\DTOs\PlaceOrderCommand;
use App\Modules\Orders\Domain\Entities\Order;
use App\Modules\Orders\Domain\Entities\OrderItem;
use App\Modules\Orders\Domain\Repositories\OrderRepositoryInterface;
use App\Modules\Orders\Domain\Services\InventoryGatewayInterface;

class PlaceOrder
{
    public function __construct(
        private OrderRepositoryInterface $orders,
        private InventoryGatewayInterface $inventoryGateway,
    ) {}

    public function execute(PlaceOrderCommand $command): Order
    {
        $items = array_map(
            fn (array $item) => new OrderItem(
                productId: $item['product_id'],
                quantity: $item['quantity'],
                unitPrice: $item['unit_price'],
            ),
            $command->items
        );

        $order = Order::create(
            userId: $command->userId,
            items: $items,
        );

        $reservationItems = array_map(
            fn (OrderItem $item) => [
                'product_id' => $item->getProductId(),
                'quantity' => $item->getQuantity(),
            ],
            $items
        );

        $this->inventoryGateway->reserve($reservationItems);

        $order->confirm();

        return $this->orders->save($order);
    }
}
```

### Why this is clean

This use case knows nothing about:

- HTTP request objects
- controllers
- Eloquent
- facades
- route definitions

It only works with domain entities and contracts.

## 8. Inventory Module Implementation

### 8.1 Inventory Repository Contract

```php
<?php

namespace App\Modules\Inventory\Domain\Repositories;

interface ProductStockRepositoryInterface
{
    public function getAvailableQuantity(int $productId): int;

    public function decrement(int $productId, int $quantity): void;
}
```

### 8.2 Application Service: `ReserveStockService`

```php
<?php

namespace App\Modules\Inventory\Application\Services;

use App\Modules\Inventory\Domain\Repositories\ProductStockRepositoryInterface;

class ReserveStockService
{
    public function __construct(
        private ProductStockRepositoryInterface $stocks,
    ) {}

    /**
     * @param array<int, array{product_id:int, quantity:int}> $items
     */
    public function execute(array $items): void
    {
        foreach ($items as $item) {
            $available = $this->stocks->getAvailableQuantity($item['product_id']);

            if ($available < $item['quantity']) {
                throw new \RuntimeException(
                    "Insufficient stock for product {$item['product_id']}"
                );
            }
        }

        foreach ($items as $item) {
            $this->stocks->decrement($item['product_id'], $item['quantity']);
        }
    }
}
```

### 8.3 Adapter Exposed to Orders

Inventory implements the contract that Orders depends on.

```php
<?php

namespace App\Modules\Inventory\Infrastructure\Services;

use App\Modules\Inventory\Application\Services\ReserveStockService;
use App\Modules\Orders\Domain\Services\InventoryGatewayInterface;

class StockReservationService implements InventoryGatewayInterface
{
    public function __construct(
        private ReserveStockService $reserveStockService,
    ) {}

    public function reserve(array $items): void
    {
        $this->reserveStockService->execute($items);
    }
}
```

This is the clean boundary between modules.

---

## 9. Infrastructure Layer

### 9.1 Eloquent Models

```php
<?php

namespace App\Modules\Orders\Infrastructure\Persistence\Eloquent\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\HasMany;

class OrderModel extends Model
{
    protected $table = 'orders';

    protected $fillable = [
        'user_id',
        'status',
    ];

    public function items(): HasMany
    {
        return $this->hasMany(OrderItemModel::class, 'order_id');
    }
}
```

```php
<?php

namespace App\Modules\Orders\Infrastructure\Persistence\Eloquent\Models;

use Illuminate\Database\Eloquent\Model;

class OrderItemModel extends Model
{
    protected $table = 'order_items';

    protected $fillable = [
        'order_id',
        'product_id',
        'quantity',
        'unit_price',
    ];
}
```

### 9.2 Repository Implementation

```php
<?php

namespace App\Modules\Orders\Infrastructure\Persistence\Eloquent\Repositories;

use App\Modules\Orders\Domain\Entities\Order;
use App\Modules\Orders\Domain\Entities\OrderItem;
use App\Modules\Orders\Domain\Repositories\OrderRepositoryInterface;
use App\Modules\Orders\Infrastructure\Persistence\Eloquent\Models\OrderItemModel;
use App\Modules\Orders\Infrastructure\Persistence\Eloquent\Models\OrderModel;
use Illuminate\Support\Facades\DB;

class EloquentOrderRepository implements OrderRepositoryInterface
{
    public function save(Order $order): Order
    {
        return DB::transaction(function () use ($order) {
            $orderModel = OrderModel::create([
                'user_id' => $order->getUserId(),
                'status' => $order->getStatus(),
            ]);

            foreach ($order->getItems() as $item) {
                OrderItemModel::create([
                    'order_id' => $orderModel->id,
                    'product_id' => $item->getProductId(),
                    'quantity' => $item->getQuantity(),
                    'unit_price' => $item->getUnitPrice(),
                ]);
            }

            return new Order(
                id: $orderModel->id,
                userId: $orderModel->user_id,
                items: array_map(
                    fn ($item) => new OrderItem(
                        productId: $item->product_id,
                        quantity: $item->quantity,
                        unitPrice: $item->unit_price,
                    ),
                    $orderModel->items()->get()->all()
                ),
                status: $orderModel->status,
            );
        });
    }

    public function findById(int $id): ?Order
    {
        $model = OrderModel::with('items')->find($id);

        if (! $model) {
            return null;
        }

        return new Order(
            id: $model->id,
            userId: $model->user_id,
            items: array_map(
                fn ($item) => new OrderItem(
                    productId: $item->product_id,
                    quantity: $item->quantity,
                    unitPrice: $item->unit_price,
                ),
                $model->items->all()
            ),
            status: $model->status,
        );
    }
}
```

### 9.3 Inventory Stock Repository

```php
<?php

namespace App\Modules\Inventory\Infrastructure\Persistence\Eloquent\Repositories;

use App\Modules\Inventory\Domain\Repositories\ProductStockRepositoryInterface;
use App\Modules\Inventory\Infrastructure\Persistence\Eloquent\Models\ProductStockModel;

class EloquentProductStockRepository implements ProductStockRepositoryInterface
{
    public function getAvailableQuantity(int $productId): int
    {
        $stock = ProductStockModel::where('product_id', $productId)->first();

        return $stock?->available_quantity ?? 0;
    }

    public function decrement(int $productId, int $quantity): void
    {
        $stock = ProductStockModel::where('product_id', $productId)->firstOrFail();

        $stock->decrement('available_quantity', $quantity);
    }
}
```

## 10. Presentation Layer

### 10.1 Request Validation

```php
<?php

namespace App\Modules\Orders\Presentation\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;

class PlaceOrderRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'user_id' => ['required', 'integer'],
            'items' => ['required', 'array', 'min:1'],
            'items.*.product_id' => ['required', 'integer'],
            'items.*.quantity' => ['required', 'integer', 'min:1'],
            'items.*.unit_price' => ['required', 'integer', 'min:1'],
        ];
    }
}
```

### 10.2 Controller

```php
<?php

namespace App\Modules\Orders\Presentation\Http\Controllers;

use App\Http\Controllers\Controller;
use App\Modules\Orders\Application\DTOs\PlaceOrderCommand;
use App\Modules\Orders\Application\UseCases\PlaceOrder;
use App\Modules\Orders\Presentation\Http\Requests\PlaceOrderRequest;
use App\Modules\Orders\Presentation\Http\Resources\OrderResource;
use Illuminate\Http\JsonResponse;

class OrderController extends Controller
{
    public function store(
        PlaceOrderRequest $request,
        PlaceOrder $placeOrder,
    ): JsonResponse {
        $command = new PlaceOrderCommand(
            userId: $request->integer('user_id'),
            items: $request->input('items'),
        );

        $order = $placeOrder->execute($command);

        return response()->json([
            'data' => (new OrderResource($order))->toArray($request),
        ], 201);
    }
}
```

### 10.3 Resource

```php
<?php

namespace App\Modules\Orders\Presentation\Http\Resources;

use App\Modules\Orders\Domain\Entities\Order;
use App\Modules\Orders\Domain\Entities\OrderItem;

class OrderResource
{
    public function __construct(
        private Order $order
    ) {}

    public function toArray($request): array
    {
        return [
            'id' => $this->order->getId(),
            'user_id' => $this->order->getUserId(),
            'status' => $this->order->getStatus(),
            'items' => array_map(
                fn (OrderItem $item) => [
                    'product_id' => $item->getProductId(),
                    'quantity' => $item->getQuantity(),
                    'unit_price' => $item->getUnitPrice(),
                    'subtotal' => $item->getSubtotal(),
                ],
                $this->order->getItems()
            ),
        ];
    }
}
```

### 10.4 Module Route File

```php
<?php

use Illuminate\Support\Facades\Route;
use App\Modules\Orders\Presentation\Http\Controllers\OrderController;

Route::prefix('orders')->group(function () {
    Route::post('/', [OrderController::class, 'store']);
});
```

## 11. Service Providers and Dependency Injection

### 11.1 Orders Service Provider

```php
<?php

namespace App\Modules\Orders\Infrastructure\Providers;

use Illuminate\Support\ServiceProvider;
use App\Modules\Orders\Domain\Repositories\OrderRepositoryInterface;
use App\Modules\Orders\Domain\Services\InventoryGatewayInterface;
use App\Modules\Orders\Infrastructure\Persistence\Eloquent\Repositories\EloquentOrderRepository;
use App\Modules\Inventory\Infrastructure\Services\StockReservationService;

class OrdersServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(OrderRepositoryInterface::class, EloquentOrderRepository::class);
        $this->app->bind(InventoryGatewayInterface::class, StockReservationService::class);
    }

    public function boot(): void
    {
        $this->loadRoutesFrom(app_path('Modules/Orders/Presentation/Routes/api.php'));
        $this->loadMigrationsFrom(app_path('Modules/Orders/Database/Migrations'));
    }
}
```

### 11.2 Inventory Service Provider

```php
<?php

namespace App\Modules\Inventory\Infrastructure\Providers;

use Illuminate\Support\ServiceProvider;
use App\Modules\Inventory\Domain\Repositories\ProductStockRepositoryInterface;
use App\Modules\Inventory\Infrastructure\Persistence\Eloquent\Repositories\EloquentProductStockRepository;

class InventoryServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        $this->app->bind(
            ProductStockRepositoryInterface::class,
            EloquentProductStockRepository::class
        );
    }

    public function boot(): void
    {
        $this->loadMigrationsFrom(app_path('Modules/Inventory/Database/Migrations'));
    }
}
```

### Why this matters

This is what makes the architecture real:

- the use case depends on interfaces
- the container provides implementations
- modules remain replaceable behind contracts

## 12. Database Migrations

### 12.1 Orders Table

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('orders', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('user_id');
            $table->string('status');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('orders');
    }
};
```

### 12.2 Order Items Table

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('order_items', function (Blueprint $table) {
            $table->id();
            $table->foreignId('order_id')->constrained('orders')->cascadeOnDelete();
            $table->unsignedBigInteger('product_id');
            $table->integer('quantity');
            $table->integer('unit_price');
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('order_items');
    }
};
```

### 12.3 Product Stocks Table

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration {
    public function up(): void
    {
        Schema::create('product_stocks', function (Blueprint $table) {
            $table->id();
            $table->unsignedBigInteger('product_id')->unique();
            $table->integer('available_quantity')->default(0);
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('product_stocks');
    }
};
```

## 13. Shared Folder: What It Is For

The `Shared` folder is for things that are:

- generic
- reusable across modules
- not owned by one business module

Good examples:

- `Money`
- `Email`
- `ClockInterface`
- `IdGeneratorInterface`

Bad examples:

- `OrderHelper`
- `BillingCommon`
- `InventoryUtils`
- feature-specific services moved to shared just because two modules use them

### Example Shared Value Object

```php
<?php

namespace App\Modules\Shared\Domain\ValueObjects;

final class Money
{
    public function __construct(
        private int $amount,
        private string $currency = 'USD'
    ) {}

    public function add(Money $other): Money
    {
        if ($this->currency !== $other->currency) {
            throw new \InvalidArgumentException('Currency mismatch');
        }

        return new Money($this->amount + $other->amount, $this->currency);
    }

    public function amount(): int
    {
        return $this->amount;
    }

    public function currency(): string
    {
        return $this->currency;
    }
}
```

## 14. Request Flow Summary

For `POST /orders`:

1. `OrderController` receives the request
2. `PlaceOrderRequest` validates input
3. Controller creates `PlaceOrderCommand`
4. `PlaceOrder` use case executes
5. Use case builds domain `Order` and `OrderItem`
6. Use case calls `InventoryGatewayInterface`
7. Laravel resolves it to `StockReservationService`
8. Inventory reserves stock
9. Orders repository persists the order
10. Resource returns JSON response

This is a modular monolith because the whole flow runs inside one Laravel app.

This is clean architecture because the business flow does not depend on framework details.

## 15. What Not to Do

### Bad: Use Eloquent directly in the use case

```php
$order = OrderModel::create([...]);
```

This makes the application layer depend on Laravel ORM.

### Bad: Access another module's internal models

```php
ProductStockModel::where('product_id', $id)->decrement(...);
```

This breaks module boundaries because Orders is now touching Inventory internals.

### Bad: Put business logic in controllers

```php
foreach ($request->items as $item) {
    // reserve stock
    // calculate total
    // create order
}
```

Controllers should adapt HTTP, not implement business rules.

## 16. Recommended Practical Rules

For a real Laravel modular monolith:

- keep **Domain** pure PHP
- keep **Application** focused on use cases
- keep **Infrastructure** responsible for Eloquent and framework integration
- keep **Presentation** thin
- let modules talk through interfaces
- keep `Shared` small and disciplined
- place migrations inside modules and load them through service providers

## 17. Final Takeaway

A clean modular monolith should let you say:

- **Orders can be understood mostly by reading the Orders module**
- **Inventory can change its persistence internals without breaking Orders**
- **Business rules do not depend on Laravel controllers or Eloquent**

If those are true, the architecture is in a good place.
