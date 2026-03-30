# Implementation Migration: Announcement Banner

## Overview
Added a store-wide announcement banner field to the `MerchantStore` entity, exposed via existing store API endpoints.

## Database Migration

A new column is added automatically by Hibernate on startup (DDL auto = update):

```sql
ALTER TABLE MERCHANT_STORE ADD COLUMN ANNOUNCEMENT VARCHAR(255) NULL;
```

## Files Changed

### `sm-core-model`
- `com.salesmanager.core.model.merchant.MerchantStore`
  - Added `@Column(name = "ANNOUNCEMENT") String announcement` field with getter/setter

### `sm-shop-model`
- `com.salesmanager.shop.model.store.MerchantStoreEntity`
  - Added `String announcement` field with getter/setter (inherited by `ReadableMerchantStore` and `PersistableMerchantStore`)

### `sm-shop`
- `com.salesmanager.shop.populator.store.ReadableMerchantStorePopulator`
  - Maps `source.getAnnouncement()` → `target.setAnnouncement()`
- `com.salesmanager.shop.populator.store.PersistableMerchantStorePopulator`
  - Maps `source.getAnnouncement()` → `target.setAnnouncement()`
- `com.salesmanager.shop.application.config.DocumentationConfiguration`
  - Fixed Swagger host from `localhost:8080` → `localhost:30080`

## API

No new endpoints. The `announcement` field is included in existing endpoints:

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/v1/store/{code}` | Returns `announcement` in response |
| `GET` | `/api/v1/private/store/{code}` | Returns `announcement` in response |
| `PUT` | `/api/v1/private/store/{code}` | Accepts `announcement` in request body |

## Example

**Request:**
```json
PUT /api/v1/private/store/DEFAULT
{
  "announcement": "Free shipping on orders over $50!"
}
```

**Response:**
```json
{
  "code": "DEFAULT",
  "announcement": "Free shipping on orders over $50!"
}
```
