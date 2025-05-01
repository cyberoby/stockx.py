# StockX Python SDK

An async Python SDK for interacting with the StockX API, providing both low-level endpoint mappings and high-level inventory management abstractions.

> ⚠️ This SDK is under active development.

> ⚠️ `stockx.py` is in no way endorsed by or affiliated with StockX in any way. Make sure to read and understand the
> [StockX API Terms of Service](https://developer.stockx.com/portal/license-agreement/) before using this package. 

> ⚠️ This library does not circumvent API limits, expose private or proprietary data, or violate security measures. All functionality is based solely on publicly available documentation and data.

The SDK has two layers: 
- `stockx`: low-level API client for easy access to StockX API endpoints
- `stockx.ext`: high-level abstractions for advanced business logic

## stockx
Direct mappings to StockX API endpoints through specialized interfaces:

- `Catalog` - Product catalog operations
- `Listings` - Listing management 
- `Orders` - Sales order operations
- `Batch` - Batch listing operations

#### Main features
- 🔄 Automatic token refresh and session management
- 🚦 Request throttling to prevent rate limits
- 🔁 Automatic retries with exponential backoff for failed requests
- ⚡ Response caching for invariant data (e.g., product details)

### Configuration

#### Client Parameters
- `hostname` - StockX API hostname
- `version` - API version to use
- `client_id` - OAuth client ID
- `client_secret` - OAuth client secret
- `x_api_key` - API key for authentication
- `refresh_token` - OAuth refresh token

### Response models
The SDK converts all JSON responses to typed Python frozen dataclasses, allowing for:
- Type checking
- Easy access to data using dot notation (e.g `listing.id`)
- Automatic JSON serialization and deserialization
- Pretty printing

#### Responses as dataclasses with pretty print functionality

```python
>>> product_id = '16999d83-1c59-4913-ae9f-9a2b2ee9f1b3'
>>> product = await stockx.catalog.get_product(product_id)
>>> print(product)  # Pretty print
Product:
  product_id: 16999d83-1c59-4913-ae9f-9a2b2ee9f1b3
  url_key: gucci-gg-supreme-monogram-apple-card-case-wallet-brown
  style_id:
  product_type: handbags
  title: Gucci GG Supreme Monogram Apple Card Case Wallet Brown
  brand: Gucci
  product_attributes:
    ProductAttributes:
      gender: women
      season: None
      release_date: None
      retail_price: None
      colorway: None
      color: Brown
```

#### Automatic type conversion

```python
>>> from datetime import datetime
>>> from stockx import OrderStatusClosed
>>> client = StockXAPIClient(...)
>>> async with StockX(client) as stockx:
...     # Get an order with status DIDNOTSHIP
...     async for order in stockx.orders.get_orders_history(
...         from_date=datetime(2024, 1, 1),
...         to_date=datetime(2024, 12, 1),
...         order_status=OrderStatusClosed.DIDNOTSHIP,
...         limit=1,
...         page_size=50
...     ):
...         print(f'Order Number: {order.number} (Type: {type(order.number).__name__})')
...         print(f'Status: {order.status} (Type: {type(order.status).__name__})')
...         print(f'Created At: {order.created_at} (Type: {type(order.created_at).__name__})')
...
Order Number: 68322683-68222442 (Type: str)
Status: OrderStatusClosed.DIDNOTSHIP (Type: OrderStatusClosed)
Created At: 2024-10-03 16:12:40+00:00 (Type: datetime) 
```

### Helper Methods

The SDK provides utility methods to simplify common operations.

#### Operation Status Polling
The `operation_succeeded()` method polls the status of asynchronous listing operations:

```python
>>> # Create a new listing
>>> operation = await stockx.listings.create_listing(
...     amount=100.0,
...     variant_id="123",
...     currency=Currency.EUR
... )

>>> # Wait for the operation to complete and check if it succeeded
>>> if await stockx.listings.operation_succeeded(operation):
...     print("Listing created successfully!")
... else:
...     print(f"Listing creation failed: {operation.error}")
```

## stockx.ext

High-level abstractions for advanced business logic and inventory management:

- `Inventory` - Optimized high-level interface for managing listings on StockX
- `Item` / `ListedItem` - Abstraction aggregating multiple equal listings into a single inventory entry
- `mock_listing` - Create temporary listings for testing
- `search` - Product search utilities

### stockx.ext.inventory.Inventory

The `Inventory` class provides a high-level interface for managing listings on StockX.
It optimizes performance by batching multiple updates together into single API calls. 

Features:
- Sell or de-list items in bulk
- Set prices based on market data and custom conditions
- Update item quantities and prices on context exit

### Examples

#### Update item prices and quantities easily

```python
>>> client = StockXAPIClient(...)
>>> async with StockX(client) as stockx:
...     async with Inventory(stockx) as inventory:
...         # Get items listed for over 200 payout
...         items = await (
...             inventory.items()
...             .filter(lambda item: item.payout() > 200)
...             .all()
...         )
...         for item in items:
...             item.price -= 20    # Reduce price by 20
...             item.quantity += 1  # Increase quantity by 1
...         # Changes are automatically applied when exiting context
```

#### Retrieve listed items by custom criteria

```python
>>> async with Inventory(stockx) as inventory:
...     # Get all items with style ID 'L47450600'
...     salomons_gtx = await inventory.items().filter_by(style_ids=['L47450600']).all()
...
...     print(salomons_gtx[0].style_id)
...     print(salomons_gtx[0].name)
...     print(salomons_gtx[0].size)
...     print(f'${salomons_gtx[0].payout():.2f}')
...     print(salomons_gtx[0].quantity)
...
L47450600
Salomon XT-6 Gore-Tex Black Silver
11.5
$167.40
3
```

#### Sell items

```python
>>> client = StockXAPIClient(...)
>>> async with StockX(client) as stockx:
...     async with Inventory(stockx) as inventory:
...         # Create items from SKU and size (Air Force 1 '07 Triple White)
...         af1_size_9 = await Item.from_sku_size(stockx, 'CW2288-111', 'US 9', 110.00)  
...         af1_size_85 = await Item.from_sku_size(stockx, 'CW2288-111', 'US 8.5', 110.00)
...         
...         # Create listings
...         listed_items = await inventory.sell([af1_size_9, af1_size_85])
...         
...         # Print results
...         for item in listed_items:
...             print(f'Expected payout: ${item.payout()}')
...
Expected payout: $89.80
Expected payout: $89.80
```
### Advanced Usage Examples

#### Custom filtering
```python
>>> # Get items with specific criteria
>>> items = await (
...     inventory.items()
...     .filter_by(style_ids=['CW2288-111', 'L47450600'])
...     .filter(lambda item: item.payout() > 150)
...     .filter(lambda item: item.quantity > 5)
...     .all()
... )
```
#### Change prices dynamically by injecting user-defined functions
Using functions:
```python
>>> # Apply 10% discount to items with payout over 200
>>> await inventory.change_price(
...     items=items,
...     new_price=lambda item: item.price * 0.9,
...     condition=lambda item: item.payout() > 200
... )
```
Using async functions:
```python
>>> async def dynamic_beat_by(item: ListedItem) -> float:
...     market_data = await inventory.get_item_market_data(item)
...     lowest_ask = market_data.lowest_ask.amount
...     highest_bid = market_data.highest_bid.amount
...     bid_ask_spread = lowest_ask - highest_bid
...     return bid_ask_spread * 0.01    # Beat by 1.0% of bid-ask spread
...
>>> async def dynamic_condition(item: ListedItem) -> bool:
...     market_data = await inventory.get_item_market_data(item)
...     return item.price > market_data.lowest_ask.amount   # Only if price > lowest ask
...
>>> await inventory.beat_lowest_ask(
...     items=items,
...     beat_by=dynamic_beat_by,      # Dynamic amount based on market
...     condition=dynamic_condition   # Dynamic condition based on market
... )
```

### stockx.ext.mock_listing

The `mock_listing` context manager can be used to create temporary listings for various purposes.
The `Inventory` class uses it to retrieve current selling fees in order to calculate payouts accurately.

### Examples

#### Check if there's a discount on selling fees

```python
>>> async with mock_listing(stockx) as listing:
...     # Check if there's a discount on selling fees 
...     if listing.payout.transaction_fee == 0:
...         print(f'100% discount on selling fees!')
```

### stockx.ext.search

The `search` module provides utilities for searching the StockX catalog.
Results are cached indefinitely to reduce the number of requests to the API.

### Examples

#### Search for a product by SKU

```python
>>> from stockx.ext import search
...
>>> product = await search.product_by_sku(stockx, 'CW2288-111')
>>> print(product.name)
Nike Air Force 1 '07 Triple White

>>> product = await search.product_by_sku(stockx, 'JABBERWOCKY')
>>> print(product)
None
```

## API Endpoints Map

| Method | Endpoint | SDK Function |
|--------|----------|--------------|
| GET | `/catalog/products/{id}` | `Catalog.get_product()` |
| GET | `/catalog/products/{id}/variants` | `Catalog.get_all_product_variants()` |
| GET | `/catalog/products/{id}/variants/{id}` | `Catalog.get_product_variant()` |
| GET | `/catalog/products/{id}/variants/{id}/market-data` | `Catalog.get_variant_market_data()` |
| GET | `/catalog/products/{id}/market-data` | `Catalog.get_product_market_data()` |
| GET | `/catalog/search` | `Catalog.search_catalog()` |
| POST| `/catalog/ingestion` | Coming soon... |
| GET | `/catalog/ingestion/{id}` | Coming soon... |
| GET | `/selling/listings/{id}` | `Listings.get_listing()` |
| GET | `/selling/listings` | `Listings.get_all_listings()` |
| POST | `/selling/listings` | `Listings.create_listing()` |
| DELETE | `/selling/listings/{id}` | `Listings.delete_listing()` |
| PUT | `/selling/listings/{id}/activate` | `Listings.activate_listing()` |
| PUT | `/selling/listings/{id}/deactivate` | `Listings.deactivate_listing()` |
| PATCH | `/selling/listings/{id}` | `Listings.update_listing()` |
| GET | `/selling/listings/{id}/operations/{id}` | `Listings.get_listing_operation()` |
| GET | `/selling/listings/{id}/operations` | `Listings.get_all_listing_operations()` |
| POST | `/selling/batch/create-listing` | `Batch.create_listings()` |
| GET | `/selling/batch/create-listing/{id}` | `Batch.create_listings_status()` |
| GET | `/selling/batch/create-listing/{id}/items` | `Batch.create_listings_items()` |
| POST | `/selling/batch/update-listing` | `Batch.update_listings()` |
| GET | `/selling/batch/update-listing/{id}` | `Batch.update_listings_status()` |
| GET | `/selling/batch/update-listing/{id}/items` | `Batch.update_listings_items()` |
| POST | `/selling/batch/delete-listing` | `Batch.delete_listings()` |
| GET | `/selling/batch/delete-listing/{id}` | `Batch.delete_listings_status()` |
| GET | `/selling/batch/delete-listing/{id}/items` | `Batch.delete_listings_items()` |
| GET | `/selling/orders/{id}` | `Orders.get_order()` |
| GET | `/selling/orders/history` | `Orders.get_orders_history()` |
| GET | `/selling/orders/active` | `Orders.get_active_orders()` |
| GET | `/selling/orders/{orderNumber}/shipping-document/{shippingId}` | Coming soon... |

