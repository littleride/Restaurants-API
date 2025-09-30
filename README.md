# Little Africa Deliveries API - Integration Guide

**Version:** `2.0` 

**support**: `support.integrations@little.africa`

**Last Updated:** `January 2025 `

**Base URL:** `https://pay.little.africa/deliveries`

---

## Table of Contents

1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [Authentication](#authentication)
4. [Menu Management Flow](#menu-management-flow)
5. [Order Processing Flow](#order-processing-flow)
6. [Webhook Callbacks](#webhook-callbacks)
7. [API Reference](#api-reference)
8. [Integration Steps](#integration-steps)
9. [Error Handling](#error-handling)
10. [Testing](#testing)

---

## Overview

The Little Africa Deliveries API enables restaurant partners to:
- Upload and manage menus
- Receive orders via webhooks
- Update order status
- Track order callbacks

### Key Features
- **Asynchronous Menu Processing**: Menus are queued and processed automatically via Cron jobs
- **Webhook Notifications**: Real-time order notifications and menu processing status updates
- **Retry Logic**: Automatic retries for failed callbacks

---



###  Processing Flow Diagram

```
┌──────────────┐
│   Merchant   │
│  Uploads Menu│
└──────┬───────┘
       │ POST /menu/store/:storeID
       ▼
┌──────────────────────┐
│   API Server         │
│  - Validates Request │
│  - Stores in MongoDB │
│  - Returns uploadId  │
└──────┬───────────────┘
       │ Status: QUEUED
       ▼
┌──────────────────────────┐
│   MongoDB                │
│  SchedulesMenuUploads    │
│  status: PENDING         │
└──────┬───────────────────┘
       │
       │ ┌─────────────────────────┐
       │ │  Cron (Every 5 min) │
       │ └─────────────────────────┘
       ▼
┌──────────────────────────┐
│  Menu Processor          │
│  - Picks PENDING menu    │
│  - Sets status: PROCESSING│
│  - Processes menu data   │
└──────┬───────────────────┘
       │
       ├──► Success
       │    └──► status: PROCESSED
       │         └──► Webhook: menuProcessingCompleted
       │
       └──► Failure
            └──► status: FAILED
                 └──► Webhook: menuProcessingFailed
```

---

## Authentication

### Get Access Token

**Endpoint:** `POST /auth/token`

**Authentication Method:** HTTP Basic Auth

**Headers:**
```
Authorization: Basic base64(email:password)
```

**Example Request:**
```bash
curl -X POST "https://pay.little.africa/deliveries/auth/token" \
  -u "merchant@example.com:your_password"
```

**Success Response:**
```json
{
  "success": true,
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJlbWFpbCI6Im1lcmNoYW50QGV4YW1wbGUuY29tIiwiaWF0IjoxNzEwMzMzMjEzLCJleHAiOjE3MTAzNDQwMTN9.rn4lJakqDbvAr1WQnqO-MERUwvq9SF3uKwhBT45B1lY",
  "exp": "3h"
}
```

**Token Usage:**

All authenticated API calls must include the token in the `Authorization` header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Token Expiry:** 3 hours

---

## Menu Management Flow

### Step 1: Configure Callback URL

Before uploading menus, configure your webhook URL to receive processing notifications.

**Endpoint:** `POST /store/setCallbackUrl`

**Request:**
```bash
curl -X POST "https://pay.little.africa/deliveries/store/setCallbackUrl" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "callbackUrl": "https://your-domain.com/api/webhooks/deliveries"
  }'
```

**Response:**
```json
{
  "success": true,
  "message": "Callback URL set successfully",
  "callbackUrl": "https://your-domain.com/api/webhooks/deliveries",
  "email": "merchant@example.com"
}
```

### Step 2: Upload Menu

**Endpoint:** `POST /menu/store/:storeID`

**URL Parameters:**
- `storeID`: Your masked store identifier

**Request Headers:**
```
Authorization: Bearer YOUR_JWT_TOKEN
Content-Type: application/json
```

**Request Body Structure:**
```json
{
  "categories": [
    {
      "id": "category_001",
      "details": {
        "title": "Main Menu"
      },
      "item_ids": [
        "item_001",
        "item_002"
      ]
    }
  ],
  "items": [
    {
      "id": "item_001",
      "details": {
        "title": "Chicken Burger Combo",
        "description": "A crispy and juicy chicken burger with fresh lettuce, tomatoes, and special sauce. Comes with fries and a drink.",
        "image_url": "https://restaurants.reserveport.com/v1/images/d42e77804c47e4a1860a0b13.png"
      },
      "modifier_group_ids": [
        "modifier_group_001",
        "modifier_group_002"
      ],
      "quantity_info": {
        "max_items_permitted_per_order": 10
      },
      "price_info": {
        "price": 850,
        "special_price": 850
      }
    },
    {
      "id": "item_002",
      "details": {
        "title": "Bottled Water",
        "description": "500ml bottled mineral water",
        "image_url": "https://restaurants.reserveport.com/v1/images/water-bottle.png"
      },
      "modifier_group_ids": [],
      "quantity_info": {
        "max_items_permitted_per_order": 20
      },
      "price_info": {
        "price": 80,
        "special_price": 80
      }
    }
  ],
  "modifier_groups": [
    {
      "id": "modifier_group_001",
      "details": {
        "title": "Choose Your Drink"
      },
      "modifier_options": [
        "modifier_option_001",
        "modifier_option_002",
        "modifier_option_003"
      ],
      "quantity_info": {
        "type_of_selection": "ONE",
        "max_items_permitted_per_order": 1,
        "required": true
      }
    },
    {
      "id": "modifier_group_002",
      "details": {
        "title": "Fries Size"
      },
      "modifier_options": [
        "modifier_option_004",
        "modifier_option_005"
      ],
      "quantity_info": {
        "type_of_selection": "ONE",
        "max_items_permitted_per_order": 1,
        "required": true
      }
    }
  ],
  "modifier_items": [
    {
      "id": "modifier_option_001",
      "modifier_group_id": "modifier_group_001",
      "details": {
        "modifier_name": "Coca-Cola",
        "modifier_description": "Classic Coca-Cola"
      },
      "price_info": {
        "modifier_price": 0,
        "modifier_special_price": 0
      }
    },
    {
      "id": "modifier_option_002",
      "modifier_group_id": "modifier_group_001",
      "details": {
        "modifier_name": "Fanta Orange",
        "modifier_description": "Orange flavored soda"
      },
      "price_info": {
        "modifier_price": 0,
        "modifier_special_price": 0
      }
    },
    {
      "id": "modifier_option_003",
      "modifier_group_id": "modifier_group_001",
      "details": {
        "modifier_name": "Sprite",
        "modifier_description": "Lemon-lime soda"
      },
      "price_info": {
        "modifier_price": 0,
        "modifier_special_price": 0
      }
    },
    {
      "id": "modifier_option_004",
      "modifier_group_id": "modifier_group_002",
      "details": {
        "modifier_name": "Regular Fries",
        "modifier_description": "Standard portion"
      },
      "price_info": {
        "modifier_price": 0,
        "modifier_special_price": 0
      }
    },
    {
      "id": "modifier_option_005",
      "modifier_group_id": "modifier_group_002",
      "details": {
        "modifier_name": "Large Fries",
        "modifier_description": "Extra large portion"
      },
      "price_info": {
        "modifier_price": 100,
        "modifier_special_price": 100
      }
    }
  ]
}
```

**Field Descriptions:**

#### Categories
- **`id`**: Unique identifier for the category (e.g., "category_001")
- **`details`**: Category metadata object
  - **`title`**: Category name displayed to customers (e.g., "Main Menu")
- **`item_ids`**: Array of item IDs that belong to this category

#### Items
- **`id`**: Unique identifier for the menu item (e.g., "item_001")
- **`details`**: Item metadata object
  - **`title`**: Item name displayed to customers
  - **`description`**: Detailed description of the item
  - **`image_url`**: URL to the item's image
- **`modifier_group_ids`**: Array of modifier group IDs applicable to this item
- **`quantity_info`**: Stock and ordering limits
  - **`max_items_permitted_per_order`**: Maximum quantity per order
- **`price_info`**: Pricing information
  - **`price`**: Regular price in cents (e.g., 850 )
  - **`special_price`**: Promotional price in cents (use same as price if no promotion)

#### Modifier Groups
- **`id`**: Unique identifier for the modifier group (e.g., "modifier_group_001")
- **`details`**: Group metadata object
  - **`title`**: Group title (e.g., "Choose Your Drink", "Fries Size")
- **`modifier_options`**: Array of modifier option IDs in this group
- **`quantity_info`**: Selection rules
  - **`type_of_selection`**: "ONE" (single choice) or "MULTIPLE" (multi-select)
  - **`max_items_permitted_per_order`**: Maximum number of options customer can select
  - **`required`**: Boolean - whether customer must make a selection

#### Modifier Items
- **`id`**: Unique identifier for the modifier option (e.g., "modifier_option_001")
- **`modifier_group_id`**: References the parent modifier group ID
- **`details`**: Modifier metadata object
  - **`modifier_name`**: Option name (e.g., "Coca-Cola", "Large Fries")
  - **`modifier_description`**: Description of the option
- **`price_info`**: Additional pricing
  - **`modifier_price`**: Extra cost in cents (0 if included in base price)
  - **`modifier_special_price`**: Promotional extra cost in cents

**Success Response:**
```json
{
  "success": true,
  "message": "Menu queued for processing",
  "uploadId": "65a5f123456789abcdef1234",
  "status": "QUEUED"
}
```

**Important Notes:**
- Menu is **NOT** processed immediately
- Menu is queued with status `PENDING`
- cron processor runs every **5 minutes**
- You'll receive webhook notifications at each processing stage

### Step 3: Track Menu Processing Status

**Endpoint:** `GET /menu/upload/:uploadId/status`

**Request:**
```bash
curl -X GET "https://pay.little.africa/deliveries/menu/upload/65a5f123456789abcdef1234/status" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json"
```

**Response:**
```json
{
  "success": true,
  "upload": {
    "_id": "65a5f123456789abcdef1234",
    "storeID": "store_id",
    "status": "PROCESSED",
    "processingStartedAt": "2025-01-15T14:25:00.000Z",
    "processingCompletedAt": "2025-01-15T14:25:45.000Z",
    "timestamp": "2025-01-15T14:20:00.000Z",
    "errorMessage": null
  }
}
```

**Status Values:**
- `PENDING`: Waiting in queue for processing
- `PROCESSING`: Currently being processed
- `PROCESSED`: Successfully completed
- `FAILED`: Processing failed (check `errorMessage`)
- `VOID`: Manually cancelled

---

## Order Processing Flow

### Order Created Event (You Receive This)

When a customer places an order, the system sends a **POST** request to your configured callback URL.

**Webhook URL:** Your configured endpoint (e.g., `https://your-domain.com/api/webhooks/deliveries`)

**Method:** POST

**Headers:**
```
Content-Type: application/json
```

**Payload Example:**
```json
{
  "eventType": "orderCreated",
  "customer": {
    "mobile": "25471*******2",
    "name": "Mrgan Bill"
  },
  "storeInfo": {
    "store_email": "merchant@example.com",
    "store_id": "store_id"
  },
  "orderDetails": {
    "total_order_amount": 3410,
    "customer_comments": "Extra napkins please",
    "dateTime": "2025-01-15T14:36:42.143Z",
    "orderID": "2FABFB39-03FF-429C-BBF5-3FFFBDFD19F7-2025-01-15"
  },
  "products": [
    {
      "item_id": "unique_item_id",
      "item_title": "12 Pieces Chicken",
      "item_category": "CHICKEN",
      "food_categoryID": "unique_category_id",
      "food_category_title": "CHICKEN",
      "cost_of_item": 2450,
      "quantity": 1,
      "attributes": [
        {
          "modifier_item_id": "unique_modifier_item_id_2",
          "modifier_item_name": "2L Soda",
          "modifier_item_original_price": 370,
          "modifier_item_special_price": 370
        }
      ]
    }
  ],
  "deliveryDetails": {
    "address": "Kabarsiran Avenue, Nairobi",
    "dropoffLL": "-1.2672051,36.7664152",
    "delivery_mode": "Delivery",
    "delivery_cost": 200
  }
}
```

**Your Expected Response:**

You must respond with **HTTP 200** to acknowledge receipt:

```json
{
  "message": "Order received successfully",
  "status": 200,
  "orderID": "2FABFB39-03FF-429C-BBF5-3FFFBDFD19F7-2025-01-15"
}
```

**Important:**
- Response must have **HTTP status 200**
- Non-200 responses trigger automatic retries (max 3 attempts)
- Response time should be < 5 seconds

### Update Order Status

After receiving an order, update its status as it progresses through your system.

**Endpoint:** `PATCH /orders/process`

**Request:**
```bash
curl -X PATCH "https://pay.little.africa/deliveries/orders/process" \
  -H "Authorization: Bearer YOUR_JWT_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "orderID": "2FABFB39-03FF-429C-BBF5-3FFFBDFD19F7-2025-01-15",
    "status": "ORDER_ACCEPTED",
    "merchantEmail": "merchant@example.com"
  }'
```

**Available Status Values:**
- `ORDER_ACCEPTED`: Order confirmed by restaurant
- `ORDER_ALMOST`: Order is almost ready
- `ORDER_READY`: Order ready for pickup/delivery
- `ORDER_DELIVERED`: Order delivered to customer

**Response:**
```json
{
  "success": true,
  "message": "Order status updated successfully"
}
```

---

## Webhook Callbacks

Your system will receive POST requests at your configured callback URL for various events.

### 1. Menu Upload Queued

**Trigger:** Immediately after successful menu upload

**Event Type:** `menuUploadQueued`

**Payload:**
```json
{
  "eventType": "menuUploadQueued",
  "uploadId": "65a5f123456789abcdef1234",
  "storeID": "masked_store_id",
  "status": "QUEUED",
  "uploadedAt": "2025-01-15T14:20:00.000Z"
}
```

### 2. Menu Processing Started

**Trigger:** When PM2 cron processor picks up your menu

**Event Type:** `menuProcessingStarted`

**Payload:**
```json
{
  "eventType": "menuProcessingStarted",
  "uploadId": "65a5f123456789abcdef1234",
  "storeID": "masked_store_id",
  "status": "PROCESSING",
  "processingStartedAt": "2025-01-15T14:25:00.000Z"
}
```

### 3. Menu Processing Completed

**Trigger:** When menu processing succeeds

**Event Type:** `menuProcessingCompleted`

**Payload:**
```json
{
  "eventType": "menuProcessingCompleted",
  "uploadId": "65a5f123456789abcdef1234",
  "storeID": "masked_store_id",
  "status": "PROCESSED",
  "processingCompletedAt": "2025-01-15T14:25:45.000Z"
}
```

### 4. Menu Processing Failed

**Trigger:** When menu processing fails

**Event Type:** `menuProcessingFailed`

**Payload:**
```json
{
  "eventType": "menuProcessingFailed",
  "uploadId": "65a5f123456789abcdef1234",
  "storeID": "masked_store_id",
  "status": "FAILED",
  "processingFailedAt": "2025-01-15T14:25:30.000Z",
  "errorMessage": "Database connection timeout"
}
```

### 5. Order Created

**Trigger:** When a customer places an order

**Event Type:** `orderCreated`

**Payload:** (See Order Processing Flow section above)

---

## API Reference

### Authentication

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/auth/token` | POST | Get JWT access token |

### Menu Management

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/store/setCallbackUrl` | POST | Configure webhook URL |
| `/menu/store/:storeID` | POST | Upload menu (queued for processing) |
| `/menu/upload/:uploadId/status` | GET | Check menu processing status |

### Order Management

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/orders/process` | PATCH | Update order status |

---

## Integration Steps

### Phase 1: Setup & Authentication

1. **Get Credentials**
   - Obtain your merchant email and password
   - Get your  store ID

2. **Test Authentication**
   ```bash
   curl -X POST "https://pay.little.africa/deliveries/auth/token" \
     -u "your-email@example.com:your-password"
   ```

3. **Save Token**
   - Store the JWT token securely
   - Implement token refresh logic (tokens expire in 3 hours)

### Phase 2: Configure Webhooks

1. **Set Up Webhook Endpoint**
   - Create an HTTPS endpoint on your server
   - Endpoint should accept POST requests
   - Must respond within 5 seconds
   - Must return HTTP 200 for successful receipt

2. **Register Callback URL**
   ```bash
   curl -X POST "https://pay.little.africa/deliveries/store/setCallbackUrl" \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"callbackUrl": "https://your-domain.com/api/webhooks/deliveries"}'
   ```

### Phase 3: Upload Menu

1. **Prepare Menu Data**
   - Structure your menu according to the API schema
   - Include all categories, items, and modifiers
   - Set `DropMenuFirst: true` for full menu replacement

2. **Upload Menu**
   ```bash
   curl -X POST "https://pay.little.africa/deliveries/menu/store/YOUR_STORE_ID" \
     -H "Authorization: Bearer YOUR_TOKEN" \
     -H "Content-Type: application/json" \
     -d @menu.json
   ```

3. **Monitor Processing**
   - Save the `uploadId` from the response
   - Poll the status endpoint or wait for webhook notifications
   - Check for `PROCESSED` or `FAILED` status

### Phase 4: Handle Orders

1. **Implement Webhook Handler**
   - Parse incoming `orderCreated` events
   - Validate the payload
   - Store order in your system
   - Return HTTP 200 response immediately

2. **Update Order Status**
   - As order progresses through your workflow
   - Call `/orders/updateStatus` endpoint
   - Update with appropriate status

### Phase 5: Monitor & Debug

1. **Check Logs**
   - All callbacks are logged in `callback_logs` collection
   - Monitor for failed callbacks
   - Check response times

2. **Handle Failures**
   - Implement retry logic for failed menu uploads
   - Monitor webhook delivery success rates
   - Set up alerts for critical failures

---

## Error Handling

### HTTP Status Codes

| Code | Meaning | Action |
|------|---------|--------|
| 200 | Success | Request completed successfully |
| 400 | Bad Request | Check request payload and parameters |
| 401 | Unauthorized | Token missing or invalid - re-authenticate |
| 403 | Forbidden | Token expired or insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 500 | Server Error | Retry request or contact support |

### Common Errors

**1. Invalid Token**
```json
{
  "success": false,
  "message": "Unauthorized - Invalid or expired token"
}
```
**Solution:** Re-authenticate to get a new token

**2. Menu Upload Failed**
```json
{
  "success": false,
  "httpStatusCode": 500,
  "response": "Error processing menu upload"
}
```
**Solution:** Check menu data structure, retry upload

**3. Callback Delivery Failed**
- **Cause:** Your webhook endpoint returned non-200 status
- **Solution:** Fix webhook endpoint, ensure it returns 200
- **Note:** System will retry up to 3 times

---

## Testing

### Test Authentication
```bash
curl -X POST "https://pay.little.africa/deliveries/auth/token" \
  -u "test@example.com:password" \
  -v
```

### Test Callback URL Configuration
```bash
TOKEN="your-jwt-token"

curl -X POST "https://pay.little.africa/deliveries/store/setCallbackUrl" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"callbackUrl": "https://webhook.site/unique-id"}'
```

**Tip:** Use https://webhook.site to test webhook delivery

### Test Menu Upload
```bash
TOKEN="your-jwt-token"
STORE_ID="your-masked-store-id"

curl -X POST "https://pay.little.africa/deliveries/menu/store/$STORE_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "DropMenuFirst": true,
    "categories": [...],
    "items": [...]
  }'
```

### Monitor Menu Processing
```bash
UPLOAD_ID="your-upload-id"

# Check status every 30 seconds
watch -n 30 curl -X GET \
  "https://pay.little.africa/deliveries/menu/upload/$UPLOAD_ID/status" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json"
```

---

## Support

For technical support and questions:

- **Email:** support.integrations@little.africa
- **Maintainer:** Morgan Gicheha
- **Status Page:** https://pay.little.africa/deliveries/health

---

## Appendix


### Menu Processing Timeline

| Time | Event |
|------|-------|
| T+0s | Menu uploaded, status: QUEUED |
| T+0s to T+5m | Waiting for next  cron run |
| T+5m |  cron picks up menu, status: PROCESSING |
| T+5m to T+10m | Menu being processed (actual time varies) |
| T+10m | Processing complete, status: PROCESSED or FAILED |
| T+10m | Webhook notification sent |

**Average Processing Time:** 30-45 seconds per menu  
**Maximum Wait Time:** 5 minutes (until next cron run)  
**Total Time (worst case):** ~6 minutes from upload to completion

---

**End of Integration Guide** 
