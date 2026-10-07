# Shop API: Integration Guide
## Manage a Torob Shop Panel From Code

## 0. Diagram

How a shop generator connects to a shop:

```mermaid
sequenceDiagram
    participant Merchant Browser
    participant Shop Generator
    participant Torob

    Merchant Browser->>Shop Generator: Clicks "Connect to Torob"
    Shop Generator->>Torob: POST /shop-generator/v1/connections/ (shop_domain, return_url)
    Torob-->>Shop Generator: connection_id, approve_url
    Shop Generator->>Shop Generator: Store connection_id with the merchant's session
    Shop Generator-->>Merchant Browser: Redirect to approve_url
    Merchant Browser->>Torob: Log in and approve
    Torob-->>Merchant Browser: Redirect to return_url?connection_id=...&code=...
    Merchant Browser->>Shop Generator: Callback
    Shop Generator->>Shop Generator: Check connection_id belongs to this session
    Shop Generator->>Torob: POST /shop-generator/v1/connections/{id}/confirm/ (code)
    Torob-->>Shop Generator: status: active
    Shop Generator->>Torob: GET /shop-api/v1/products/?domain=... (Bearer token)
```

## 1. Introduction

The Shop API lets shops and shop generators (shop builders) do from code what the Torob shop panel
does: read the shop's products as Torob sees them, and turn products on or off on Torob.

- **Shops** call the API for their own shop with a token they create in their Torob shop panel.
- **Shop generators** call it on behalf of the shops built on their platform. Each shop must
  approve the generator once (section 3).

This API is the reverse direction of the [Product API](product_api_v3.md): there, Torob calls your
server; here, your server calls Torob.

## 2. Authentication

Every request carries a token in the `Authorization` header:

```
Authorization: Bearer <token>
```

Never send the token in the URL.

### 2.1. Shops

Create a token on the **API access** page of your Torob shop panel
(<https://panel.torob.com/s/apiAccess>). The token is shown only once; store it safely. Creating a
new token revokes the previous one. The same page lists the shop generators connected to your shop
and lets you disconnect them. Only users of the shop can create or revoke its token, and a
token stops working if the user who created it is removed from the shop.

A shop token works only for its own shop. The `domain` parameter (section 4) is optional; if you
send it, it must be your shop's domain.

The token Torob installs in the WooCommerce plugin is separate and keeps working; it cannot call
this API.

### 2.2. Shop Generators

Use the partner token Torob gave your platform. It only works from your registered IP addresses.
Before calling the Shop API for a shop, connect to that shop (section 3), then send its domain in
the `domain` parameter of every request.

## 3. Connecting a Shop (Shop Generators)

### 3.1. Steps

1. **Request a connection** from your server with `POST /shop-generator/v1/connections/`. Store the
   returned `connection_id` together with the merchant's session in your system.
2. **Redirect the merchant's browser** to the returned `approve_url`.
3. **The merchant approves on Torob.** They log in with their Torob account, and Torob checks that
   they are a user of the shop. The browser then returns to your `return_url` with
   `connection_id` and `code`, or with `error=denied`.
4. **Check the callback.** The `connection_id` in the callback must be the one you stored for
   **this** browser session. If it is not, stop: the approval belongs to someone else.
5. **Confirm** with `POST /shop-generator/v1/connections/{connection_id}/confirm/` and the `code`.
   Only now is the connection active.

The whole flow, from step 1 to step 5, must finish within 15 minutes.

Step 4 is what stops a merchant on your platform from claiming another shop's domain: the approval
redirect goes to the real owner's browser, not theirs. Do not skip it.

### 3.2. Connection Object

```json
{
  "connection_id": "75a60226-4997-4ae3-bd10-68f7014e0779",
  "shop_domain": "example.ir",
  "status": "pending",
  "approve_url": "https://api.torob.com/shop-panel/shop-connections/75a60226-4997-4ae3-bd10-68f7014e0779/approve/"
}
```

| Field | Type | Description |
| ----- | ---- | ----------- |
| `connection_id` | string | Connection identifier |
| `shop_domain` | string | The shop's domain |
| `status` | string | `pending`, `approved`, `active`, `revoked`, or `expired` |
| `approve_url` | string or null | Where to send the merchant; set only while `pending` |

### 3.3. Endpoints

| Method | Path | Description |
| ------ | ---- | ----------- |
| `POST` | `/shop-generator/v1/connections/` | Request a connection. Body: `{"shop_domain": "...", "return_url": "https://..."}`. Returns an active connection as is; anything else starts a new request. |
| `GET` | `/shop-generator/v1/connections/{connection_id}/` | Read a connection's status. |
| `POST` | `/shop-generator/v1/connections/{connection_id}/confirm/` | Activate an approved connection. Body: `{"code": "..."}`. |
| `DELETE` | `/shop-generator/v1/connections/{connection_id}/` | Disconnect, for example when a merchant leaves your platform. Returns 204. |

`return_url` must use `https` (plain `http` is accepted only for `localhost` and `127.0.0.1`, for
local testing).

A shop can also revoke your access at any time from its Torob shop panel. After that, Shop API
calls for that shop return 403 until the shop approves a new connection.

## 4. Shop API

- **Base URL**: `https://api.torob.com/shop-api/v1/`
- **Content-Type**: `application/json`
- **`domain` query parameter**: the shop the request acts on. Required for shop generators,
  optional for shops.

### 4.1. Product Object

| Field | Type | Description |
| ----- | ---- | ----------- |
| `product_id` | string | Your product ID (the shop's own identifier) |
| `page_url` | string | Product page on the shop's site |
| `statuses` | list[string] | Torob statuses (section 4.2); empty when the product has no problem |
| `price` | integer | Current price on Torob, in tomans |
| `availability` | boolean | In stock |
| `error_title` | string | The last crawl error, empty when none |
| `torob_url` | string or null | The product's page on Torob; null until Torob has matched it |
| `updated_at` | datetime | When Torob last changed this product |

Products removed permanently for technical reasons are not returned.

### 4.2. Product Statuses

| Status | Meaning |
| ------ | ------- |
| `page_deleted_permanent_content` | Permanently removed for its content |
| `page_deleted_temporary_content` | Temporarily removed for its content |
| `page_deleted_temporary_technical` | Temporarily removed for a technical reason |
| `page_not_active` | Deactivated |
| `page_not_accessible` | The product page could not be reached |
| `in_progress_add` | Being added to Torob |
| `page_not_available` | Out of stock |
| `base_not_confirmed` | Category waiting for confirmation |
| `price_not_valid` | Price marked invalid |
| `base_not_merged` | Not merged (in categories where merging is required) |
| `base_image_rejected` | One or more images are not shown on Torob |

### 4.3. `GET /shop/`

Returns the shop the request acts on. Use it to check your token or connection.

```json
{"domain": "example.ir", "name": "فروشگاه نمونه"}
```

### 4.4. `GET /products/`

All of the shop's products, newest first, with cursor pagination.

| Parameter | Type | Required | Description |
| --------- | ---- | -------- | ----------- |
| `domain` | string | Generators | The shop |
| `page_size` | integer | Optional | Items per page, default 100, at most 500 |
| `cursor` | string | Optional | Taken from a previous `next` or `previous` link (an invalid cursor gets 400) |
| `status` | string | Optional | Only products with this status (section 4.2), or `no_problem` |
| `updated_since` | datetime | Optional | Only products changed at or after this time (ISO 8601; without an offset it is UTC) |

```json
{
  "previous": null,
  "next": "https://api.torob.com/shop-api/v1/products/?cursor=cD0xMjM0NQ%3D%3D&domain=example.ir&page_size=100",
  "results": [
    {
      "product_id": "12412_1",
      "page_url": "https://example.ir/product/34/",
      "statuses": [],
      "price": 1250000,
      "availability": true,
      "error_title": "",
      "torob_url": "https://torob.com/p/617ff459-4c95-4db1-a83b-de241313dcc2/",
      "updated_at": "2026-10-07T08:15:30.120000+00:00"
    }
  ]
}
```

Follow `next` until it is `null`.

To sync only changes, pass the largest `updated_at` you received, minus a few minutes, as the next
`updated_since`. A change can become visible a little after it happened, so the overlap makes sure
nothing is missed; some products may come back twice. URL-encode the value (an unencoded `+` in
`+03:30` turns into a space), or send it in UTC with `Z`, for example `2026-10-07T08:00:00Z`.

### 4.5. `POST /products/status/`

The same product object for specific products.

Body: `{"product_ids": ["12412_1", "missing"]}` — 1 to 5,000 of your product IDs, as strings.

```json
{
  "results": [
    {"product_id": "12412_1", "exists": true, "product": {"product_id": "12412_1", "statuses": [], "...": "..."}},
    {"product_id": "missing", "exists": false, "product": null}
  ]
}
```

Results follow the order of `product_ids`.

### 4.6. `POST /products/activate/` and `POST /products/deactivate/`

Turn products on or off on Torob. A deactivated product stops showing on Torob until it is
activated again.

Body: `{"product_ids": ["12412_1", "missing"]}` — 1 to 5,000 product IDs.

```json
{"results": [{"product_id": "12412_1", "found": true}, {"product_id": "missing", "found": false}]}
```

## 5. Errors and Limits

| Status | Body | Meaning |
| ------ | ---- | ------- |
| 400 | `{"error": "..."}` | The body is malformed or breaks a limit (`"invalid request body"`), or the connection `code` is wrong, used, or expired |
| 400 | `{"detail": "..."}` | A wrong `domain`, a missing `domain` for a generator, an unknown `status`, or an invalid `cursor` |
| 401 | `{"detail": "Unauthorized"}` | Missing or invalid token, or a request from an IP the token does not allow |
| 403 | `{"detail": "..."}` | The token may not do this: for example, no active connection to the shop, or the user who created a shop token was removed from the shop |
| 404 | `{"error": "..."}` | The connection or shop was not found |
| 429 | `{}` | Too many requests |

Each token may send up to 5 requests per second **for each shop**, across all Shop API endpoints. A
shop generator therefore gets 5 per second for every connected shop. Batch product IDs (up to 5,000
per request) instead of sending one request per product.

## 6. Example Requests

### List Products (Shop Generator)

```bash
curl --header "Authorization: Bearer <token>" \
     "https://api.torob.com/shop-api/v1/products/?domain=example.ir&page_size=100"
```

### Deactivate Products (Shop)

```bash
curl --header "Content-Type: application/json" \
     --header "Authorization: Bearer <token>" \
     --request POST \
     --data '{"product_ids": ["12412_1", "12412_2"]}' \
     "https://api.torob.com/shop-api/v1/products/deactivate/"
```

### Request a Connection (Shop Generator)

```bash
curl --header "Content-Type: application/json" \
     --header "Authorization: Bearer <token>" \
     --request POST \
     --data '{"shop_domain": "example.ir", "return_url": "https://generator.example/torob/callback"}' \
     "https://api.torob.com/shop-generator/v1/connections/"
```
