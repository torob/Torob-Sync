# Torob Sync

API documentation for integrating with Torob's product and order tracking systems.

> **Note**: These APIs are intended for large shops and shop generators (shop builders) that need direct integration with Torob. Small shops are automatically added to Torob after registration and do not need to implement these APIs.

## Product Sync APIs

APIs for syncing product information with Torob:

| API | Description | Status |
| --- | ----------- | ------ |
| [Product API v3](product_api_v3.md) | Add and update product information | **Current** |
| [Product API v2](product_api_v2.md) | Add and update product information | Deprecated |
| [Product API v1](product_api_v1.md) | Update price and availability only | Deprecated |

### Supporting Documentation

- [Token Guide](torob_api_token_guide.md) - JWT authentication for Product API v3
- [Product Webhook](product_webhook.md) - Notify Torob of product updates in real-time

## Shop API

API for shops and shop generators to manage a Torob shop panel from code:

- [Shop API](shop_api.md) - Read products and their Torob statuses, turn products on or off, and read the shop's product stats. Shop generators connect to each shop with the shop's approval.

## Order Tracking API

API for tracking orders originating from Torob:

- [Order Tracking API](order_tracking_api.md) - Track orders from users referred by Torob

## Getting Started

1. **For new integrations**: Use [Product API v3](product_api_v3.md) with [JWT authentication](torob_api_token_guide.md)
2. **For real-time updates**: Implement the [Product Webhook](product_webhook.md)
3. **For order tracking**: Implement the [Order Tracking API](order_tracking_api.md)
4. **To manage a shop panel from code**: Use the [Shop API](shop_api.md)

## Support

Contact Torob support for:
- API credentials and tokens
- Integration assistance
- Technical questions
