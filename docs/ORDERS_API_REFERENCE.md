# Orders API Reference

## Overview

The Orders API provides comprehensive order management capabilities including creation, updating, listing, and deletion of orders with support for order lines, flexible properties, advanced filtering, and intelligent line diffing. This module handles the complete order lifecycle from draft to delivery.

### Base URL
```
/orders
```

### Authentication
All endpoints require:
- Bearer token in `Authorization` header
- Organization context via `x-org-id` header

### Key Features
- **Intelligent Line Diffing**: Match order lines by ID, variant ID, or description with automatic create/update/delete
- **Properties Merge**: Add, update, or delete custom properties on orders and lines
- **Currency Fallback**: Automatic currency resolution with multi-tier fallback
- **Advanced Filtering**: Status, partnership type, text search, and JSONB properties filtering
- **Pagination & Sorting**: Configurable limit/offset pagination with multiple sort options
- **Internal Reference Generation**: Automatic unique order reference generation with organization-specific prefix
- **Delivery Address Population**: Automatic population from linked location or custom address
- **Soft Deletes**: All deletions are soft deletes with `deletedAt` timestamp tracking

---

## Core Endpoints

### GET /orders
List orders with comprehensive filtering, pagination, and sorting.

**Query Parameters:**

| Parameter | Type | Default | Max | Description |
|-----------|------|---------|-----|-------------|
| `limit` | number | 100 | 1000 | Number of orders to return |
| `offset` | number | 0 | - | Number of orders to skip |
| `sort` | string | createdAt | - | Field to sort by (createdAt, updatedAt, status, etc.) |
| `direction` | string | asc | - | Sort direction (asc or desc) |
| `status` | string | - | - | Filter by order status |
| `partnershipType` | string | - | - | Filter by partnership type (customer or supplier) |
| `search` | string | - | - | Text search across internalReference, externalReference, notes |
| `propertiesPath` | string | - | - | JSONB path for property filtering (dot notation support) |
| `propertiesValue` | string | - | - | Exact value match for specified property path |
| `propertiesExists` | boolean | - | - | Check property existence (true/false) |

**Response:**
```json
{
  "count": 5,
  "data": [
    {
      "id": "ord_xxxx",
      "organizationId": "org_xxxx",
      "partnershipId": "pship_xxxx",
      "status": "pending",
      "internalReference": "ORG-001",
      "externalReference": "EXT-REF-001",
      "notes": "Rush order",
      "totalAmount": 1500.00,
      "currency": "USD",
      "locationId": "loc_xxxx",
      "deliveryAddress": {
        "addressLine1": "123 Main St",
        "addressLine2": "Suite 100",
        "city": "New York",
        "state": "NY",
        "postalCode": "10001",
        "country": "USA"
      },
      "properties": {
        "priority": "high",
        "region": "east"
      },
      "lines": [
        {
          "id": "ordline_xxxx",
          "variantPartnershipPriceId": "vpp_xxxx",
          "description": "Premium Widget",
          "quantity": 5,
          "unitPrice": 250.00,
          "lineTotal": 1250.00,
          "currency": "USD",
          "properties": {
            "warehouse": "Zone A"
          },
          "createdAt": "2024-01-15T10:30:00Z",
          "updatedAt": "2024-01-15T10:30:00Z",
          "deletedAt": null
        }
      ],
      "createdAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-15T10:30:00Z",
      "deletedAt": null
    }
  ]
}
```

**Examples:**

```bash
# List all orders
curl -X GET "https://api.example.com/orders" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx"

# List pending orders with pagination
curl -X GET "https://api.example.com/orders?status=pending&limit=20&offset=0" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx"

# List orders from supplier partnerships only
curl -X GET "https://api.example.com/orders?partnershipType=supplier" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx"

# Search orders by text
curl -X GET "https://api.example.com/orders?search=rush%20order" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx"

# Filter by properties (priority=high)
curl -X GET "https://api.example.com/orders?propertiesPath=priority&propertiesValue=high" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx"

# Filter by property existence
curl -X GET "https://api.example.com/orders?propertiesPath=region&propertiesExists=true" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx"
```

---

### GET /orders/:id
Retrieve a single order by ID with all relations (lines, partnership info, etc.).

**Parameters:**
- `id` (path) - Order ID (required)

**Response:** Same as single order object from list response

**Examples:**

```bash
curl -X GET "https://api.example.com/orders/ord_12345" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx"
```

**Errors:**
- `404 Not Found` - Order does not exist or has been deleted

---

### POST /orders
Create a new order with optional order lines.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `partnershipId` | string | Yes | Partnership ID for this order |
| `status` | string | Yes | Initial order status |
| `internalReference` | string | No | Custom internal reference (auto-generated if omitted) |
| `externalReference` | string | No | External reference/PO number |
| `notes` | string | No | Order notes |
| `totalAmount` | string | No | Total order amount (defaults to "0") |
| `currency` | string | No | Currency code (defaults to "USD") |
| `locationId` | string | No | Linked location for delivery address |
| `deliveryAddress` | object | No | Custom delivery address |
| `properties` | object | No | Custom JSONB properties |
| `lines` | array | No | Order lines to create |

**Delivery Address Object:**
```json
{
  "addressLine1": "string (required)",
  "addressLine2": "string (optional)",
  "city": "string (required)",
  "state": "string (optional)",
  "postalCode": "string (optional)",
  "country": "string (required)"
}
```

**Order Line Objects:**
```json
{
  "variantId": "string (optional) - will be resolved to VPP",
  "variantPartnershipPriceId": "string (optional) - cannot provide both variantId and variantPartnershipPriceId",
  "description": "string (optional) - for manual lines",
  "quantity": "number (required)",
  "unitPrice": "number (optional)",
  "lineTotal": "number (optional) - auto-calculated if not provided",
  "currency": "string (optional)",
  "properties": "object (optional)"
}
```

**Response:** Order object (same as GET /orders/:id)

**Examples:**

```bash
# Create minimal order
curl -X POST "https://api.example.com/orders" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "partnershipId": "pship_xxxx",
    "status": "draft"
  }'

# Create order with full fields
curl -X POST "https://api.example.com/orders" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "partnershipId": "pship_xxxx",
    "status": "pending",
    "internalReference": "ORD-2024-001",
    "externalReference": "PO-12345",
    "notes": "Urgent delivery requested",
    "totalAmount": "2500.00",
    "currency": "USD",
    "deliveryAddress": {
      "addressLine1": "456 Oak Ave",
      "city": "Los Angeles",
      "state": "CA",
      "postalCode": "90001",
      "country": "USA"
    }
  }'

# Create order with properties
curl -X POST "https://api.example.com/orders" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "partnershipId": "pship_xxxx",
    "status": "accepted",
    "properties": {
      "priority": "high",
      "region": "west",
      "warehouse": "A1"
    }
  }'

# Create order with lines (using variantId)
curl -X POST "https://api.example.com/orders" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "partnershipId": "pship_xxxx",
    "status": "pending",
    "lines": [
      {
        "variantId": "prodvar_xxxx",
        "quantity": 5
      }
    ]
  }'

# Create order with manual lines
curl -X POST "https://api.example.com/orders" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "partnershipId": "pship_xxxx",
    "status": "draft",
    "lines": [
      {
        "description": "Bulk Pack",
        "quantity": 10,
        "unitPrice": 50.00,
        "currency": "USD",
        "properties": {
          "packSize": 10
        }
      },
      {
        "description": "Retail Single",
        "quantity": 2,
        "unitPrice": 25.00
      }
    ]
  }'
```

**Errors:**
- `400 Bad Request` - Invalid status, properties not an object, custom ID provided, etc.
- `404 Not Found` - Partnership or Variant Partnership Price not found

---

### PATCH /orders/:id
Update an existing order with intelligent line diffing.

**Parameters:**
- `id` (path) - Order ID (required)

**Request Body:** (All fields optional)

Same structure as POST but with intelligent line matching:
- Lines with `id` are matched and updated
- Lines with `variantPartnershipPriceId` are matched by variant
- Lines with `description` are matched case-insensitively
- Unmatched incoming lines are created
- Unmatched existing lines are deleted

**Response:** Updated order object

**Examples:**

```bash
# Update simple fields
curl -X PATCH "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "accepted",
    "externalReference": "EXT-UPDATED",
    "notes": "Updated notes"
  }'

# Merge properties (add new ones)
curl -X PATCH "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "shipmentMethod": "express",
      "carrier": "FedEx"
    }
  }'

# Update property value
curl -X PATCH "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "priority": "urgent"
    }
  }'

# Delete property by setting to null
curl -X PATCH "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "properties": {
      "warehouse": null
    }
  }'

# Update lines by ID
curl -X PATCH "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "lines": [
      {
        "id": "ordline_xxxx",
        "quantity": 10,
        "unitPrice": 150.00
      }
    ]
  }'

# Match lines by description (case-insensitive)
curl -X PATCH "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "lines": [
      {
        "description": "BULK PACK",
        "quantity": 20,
        "unitPrice": 55.00
      }
    ]
  }'

# Delete all lines (empty array)
curl -X PATCH "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "lines": []
  }'

# Complex update (multiple operations)
curl -X PATCH "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "shipped",
    "totalAmount": "5000.00",
    "properties": {
      "newProperty": "added",
      "oldProperty": null
    },
    "lines": [
      {
        "id": "ordline_xxxx",
        "quantity": 8,
        "properties": {
          "updated": true
        }
      },
      {
        "description": "New Emergency Item",
        "quantity": 1,
        "unitPrice": 999.99
      }
    ]
  }'
```

**Errors:**
- `400 Bad Request` - Invalid status, invalid fields, both variantId and variantPartnershipPriceId provided
- `404 Not Found` - Order, Partnership, or Variant Partnership Price not found

---

### DELETE /orders/:id
Soft delete an order (marks as deleted without removing from database).

**Parameters:**
- `id` (path) - Order ID (required)

**Response:**
```json
{
  "message": "success"
}
```

**Examples:**

```bash
curl -X DELETE "https://api.example.com/orders/ord_xxxx" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_xxxx"
```

**Errors:**
- `404 Not Found` - Order does not exist or already deleted

---

## Business Logic Deep Dives

### Line Diffing Algorithm

The line diffing algorithm enables intelligent matching of order lines during updates. This three-tier priority system determines how incoming lines are matched to existing lines:

#### Priority 1: Match by ID
If the incoming line includes an `id` field, it is matched directly to an existing line by ID. This is the most explicit way to reference a line.

#### Priority 2: Match by Variant Partnership Price ID
If the incoming line includes `variantPartnershipPriceId` and wasn't matched by ID, it searches for an existing line with the same `variantPartnershipPriceId`. This allows updating lines based on what product is being ordered.

#### Priority 3: Match by Description (Case-Insensitive)
If the incoming line has a `description` and wasn't matched by ID or variant, it searches for an existing line with a matching description. The comparison is case-insensitive, so "BULK PACK", "Bulk Pack", and "bulk pack" all match.

#### Unmatched Lines
- **Incoming lines without a match**: Created as new order lines
- **Existing lines without a match**: Soft deleted

#### Examples

```
Existing lines:
- ID: line1, Description: "Standard Widget", VPP: vpp_123
- ID: line2, Description: "Premium Widget", VPP: vpp_456

Incoming lines:
- ID: line1, Quantity: 10          → Matched by ID, UPDATED
- Description: "PREMIUM WIDGET"    → Matched by description, UPDATED
- Description: "New Item"          → No match, CREATED
- ID: line3 (doesn't exist)        → No match, CREATED

Result: line1 updated, line2 updated, new line created, line3 created as new
```

### Properties Merge Behavior

Properties are custom JSONB objects that can store flexible key-value data. When updating, properties are merged intelligently:

#### Add New Keys
Provide new property keys that don't exist in the current object. They are added to the existing properties.

#### Update Existing Keys
Provide a key that already exists with a new value. The value is updated while preserving other properties.

#### Delete Keys
Set a property key to `null` to delete it from the object.

#### Example Flow

```javascript
// Initial properties
{
  "priority": "high",
  "region": "west",
  "warehouse": "A1"
}

// Merge request
{
  "properties": {
    "shipmentMethod": "express",  // Add new key
    "carrier": "FedEx"             // Add new key
  }
}

// Result
{
  "priority": "high",              // Preserved
  "region": "west",                // Preserved
  "warehouse": "A1",               // Preserved
  "shipmentMethod": "express",     // Added
  "carrier": "FedEx"               // Added
}

// Next merge
{
  "properties": {
    "priority": "urgent",          // Update value
    "warehouse": null              // Delete key
  }
}

// Final result
{
  "priority": "urgent",            // Updated
  "region": "west",                // Preserved
  "shipmentMethod": "express",     // Preserved
  "carrier": "FedEx"               // Preserved
  // warehouse deleted
}
```

This merge behavior applies to both order-level and line-level properties.

### Currency Fallback Logic

When creating or updating order lines, the system needs to determine the appropriate currency. It uses a priority-based fallback hierarchy:

```
1. Line-level currency (if provided)
2. Order-level currency (if provided)
3. Partnership default currency (if available)
4. USD (final fallback)
```

**Example Scenarios:**

```javascript
// Scenario 1: Line has currency
Line currency: "EUR"
Order currency: "USD"
Partnership defaultCurrency: "GBP"
Result: "EUR" ✓

// Scenario 2: Only order has currency
Line currency: undefined
Order currency: "CAD"
Partnership defaultCurrency: "GBP"
Result: "CAD" ✓

// Scenario 3: Only partnership has currency
Line currency: undefined
Order currency: undefined
Partnership defaultCurrency: "JPY"
Result: "JPY" ✓

// Scenario 4: No currency specified anywhere
Line currency: undefined
Order currency: undefined
Partnership defaultCurrency: undefined
Result: "USD" ✓
```

### Line Total Calculation

Line totals can be calculated automatically or provided explicitly:

```javascript
// If lineTotal is provided: use it directly
Input: lineTotal = 500.00
Result: 500.00 ✓

// If lineTotal not provided but quantity and unitPrice are available
Input: quantity = 10, unitPrice = 50.00
Calculation: 10 * 50.00 = 500.00
Result: 500.00 ✓

// If neither lineTotal nor enough info to calculate
Input: quantity = 5, unitPrice = undefined
Result: 0 (DAL default) 
```

### Variant ID Resolution

When you provide a `variantId` instead of `variantPartnershipPriceId`, the system automatically resolves it:

1. Query the Variant Partnership Price (VPP) DAL for the given variant and partnership
2. Validate that exactly one VPP exists (warn if multiple found)
3. Extract the VPP ID and use it for the order line
4. Validate the VPP belongs to the order's partnership

**Example:**
```javascript
Input:
{
  "partnershipId": "pship_123",
  "lines": [
    {
      "variantId": "prodvar_456",
      "quantity": 5
    }
  ]
}

Resolution Process:
1. Query VPP for variant=prodvar_456, partnership=pship_123
2. Found: vpp_789 (with price $50)
3. Verify vpp_789 belongs to pship_123
4. Create line with variantPartnershipPriceId=vpp_789

Output:
{
  "lines": [
    {
      "variantPartnershipPriceId": "vpp_789",
      "quantity": 5,
      "unitPrice": 50.00,
      "lineTotal": 250.00
    }
  ]
}
```

### Delivery Address Population

When an order has a `locationId`, the delivery address can be automatically populated from that location:

```
If locationId is set AND deliveryAddress is blank:
  → Use location's address (addressLine1, city, country, etc.)

Otherwise:
  → Use the custom deliveryAddress provided
  → Or use empty defaults
```

This allows organizations to quickly create orders for known delivery locations without re-entering address details.

### Internal Reference Generation

The system automatically generates unique order references with this format:

```
{ORGANIZATION_SLUG}-{SEQUENTIAL_NUMBER}

Examples:
"ANC-PAC-1"
"ANC-PAC-2"
"ACME-CORP-1"
```

**Generation Logic:**
1. Fetch organization slug and capitalize it (e.g., "anc-pac" → "ANC-PAC")
2. Sanitize by replacing non-alphanumeric characters with hyphens
3. Query recent orders for this organization with matching prefix
4. Find the maximum sequential number
5. Increment and create new reference

You can provide a custom `internalReference` at creation time to override auto-generation.

---

## Request/Response Schemas

### Order Object

```json
{
  "id": "ord_xxxxxxxxxx",
  "organizationId": "org_xxxxxxxxxx",
  "partnershipId": "pship_xxxxxxxxxx",
  "locationId": "loc_xxxxxxxxxx | null",
  "status": "draft | pending | rejected | accepted | packed | shipped | delivered | cancelled | returned",
  "internalReference": "ORG-001",
  "externalReference": "PO-12345 | null",
  "notes": "Rush order | null",
  "totalAmount": 1500.00,
  "currency": "USD",
  "deliveryAddress": {
    "addressLine1": "123 Main St",
    "addressLine2": "Suite 100 | null",
    "city": "New York",
    "state": "NY | null",
    "postalCode": "10001 | null",
    "country": "USA"
  },
  "packedBy": "2024-01-20T14:30:00Z | null",
  "packedAt": "2024-01-20T14:30:00Z | null",
  "shippedBy": "2024-01-21T08:00:00Z | null",
  "shippedAt": "2024-01-21T08:00:00Z | null",
  "deliveredBy": "2024-01-25T16:45:00Z | null",
  "deliveredAt": "2024-01-25T16:45:00Z | null",
  "returnedBy": "2024-02-01T10:00:00Z | null",
  "returnedAt": "2024-02-01T10:00:00Z | null",
  "properties": {
    "priority": "high",
    "region": "east",
    "warehouse": "Zone A"
  },
  "lines": [
    {
      "id": "ordline_xxxxxxxxxx",
      "variantPartnershipPriceId": "vpp_xxxxxxxxxx | null",
      "description": "Premium Widget | null",
      "quantity": 5,
      "unitPrice": 250.00 | null,
      "lineTotal": 1250.00,
      "currency": "USD",
      "properties": {
        "warehouse": "Zone A",
        "expedited": true
      },
      "createdAt": "2024-01-15T10:30:00Z",
      "updatedAt": "2024-01-15T10:30:00Z",
      "deletedAt": null
    }
  ],
  "createdAt": "2024-01-15T10:30:00Z",
  "updatedAt": "2024-01-15T10:30:00Z",
  "deletedAt": null
}
```

### Order Status Enum
- `draft` - Order being prepared
- `pending` - Awaiting approval/processing
- `rejected` - Order rejected
- `accepted` - Order accepted/confirmed
- `packed` - Items packed and ready to ship
- `shipped` - Order shipped
- `delivered` - Delivered to recipient
- `cancelled` - Order cancelled
- `returned` - Items returned

---

## Filtering Capabilities

### Status Filtering

Filter orders by their current status using the `status` query parameter.

```bash
# Get only pending orders
curl -X GET "https://api.example.com/orders?status=pending"

# Get shipped orders
curl -X GET "https://api.example.com/orders?status=shipped"
```

Valid statuses: `draft`, `pending`, `rejected`, `accepted`, `packed`, `shipped`, `delivered`, `cancelled`, `returned`

### Partnership Type Filtering

Filter orders based on whether they're customer or supplier orders using `partnershipType`.

```bash
# Get customer orders only
curl -X GET "https://api.example.com/orders?partnershipType=customer"

# Get supplier orders only
curl -X GET "https://api.example.com/orders?partnershipType=supplier"
```

Valid partnership types: `customer`, `supplier`

### Text Search

Search across order text fields using the `search` parameter. Searches: `internalReference`, `externalReference`, `notes`

```bash
# Search for "rush"
curl -X GET "https://api.example.com/orders?search=rush"

# Search for PO number
curl -X GET "https://api.example.com/orders?search=PO-12345"
```

### JSONB Properties Filtering

Advanced filtering on custom properties using dot notation for nested access:

**Parameters:**
- `propertiesPath` - Path to property using dot notation (e.g., `priority`, `metadata.warehouse`)
- `propertiesValue` - Exact value to match
- `propertiesExists` - Boolean: true to check existence, false to check non-existence

**Examples:**

```bash
# Filter by exact property value
curl -X GET "https://api.example.com/orders?propertiesPath=priority&propertiesValue=high"

# Filter by nested property
curl -X GET "https://api.example.com/orders?propertiesPath=metadata.warehouse&propertiesValue=Zone%20A"

# Filter by property existence
curl -X GET "https://api.example.com/orders?propertiesPath=region&propertiesExists=true"

# Filter by property non-existence
curl -X GET "https://api.example.com/orders?propertiesPath=nonexistent&propertiesExists=false"
```

### Pagination & Sorting

**Pagination:**
```bash
# Get first 20 orders
curl -X GET "https://api.example.com/orders?limit=20&offset=0"

# Get next 20 orders
curl -X GET "https://api.example.com/orders?limit=20&offset=20"
```

**Sorting:**
```bash
# Sort by creation date (descending - newest first)
curl -X GET "https://api.example.com/orders?sort=createdAt&direction=desc"

# Sort by creation date (ascending - oldest first)
curl -X GET "https://api.example.com/orders?sort=createdAt&direction=asc"

# Sort by updated date
curl -X GET "https://api.example.com/orders?sort=updatedAt&direction=desc"
```

---

## Validation Rules

The Orders API enforces strict validation to ensure data integrity:

| Rule | Condition | Error |
|------|-----------|-------|
| No custom IDs | POST request with `id` field | 400 Bad Request |
| Properties must be object | `properties` is array or null | 400 Bad Request |
| Whitespace-only references | `internalReference` only contains whitespace | 400 Bad Request |
| Address validation | Address fields contain only whitespace | 400 Bad Request |
| Description validation | Line `description` is whitespace-only | 400 Bad Request |
| Variant exclusivity | Both `variantId` and `variantPartnershipPriceId` provided | 400 Bad Request |
| Valid status | Status not in enum | 400 Bad Request |
| Partnership exists | `partnershipId` not found | 404 Not Found |
| VPP validation | `variantPartnershipPriceId` doesn't belong to partnership | 400 Bad Request |

---

## Error Responses

All errors follow a consistent format:

```json
{
  "error": "ErrorType",
  "message": "Detailed error message",
  "statusCode": 400
}
```

### Common Error Scenarios

**400 Bad Request - Invalid Status**
```json
{
  "error": "BadRequest",
  "message": "Invalid status value: \"invalid\". Valid values are: draft, pending, rejected, accepted, packed, shipped, delivered, cancelled, returned",
  "statusCode": 400
}
```

**400 Bad Request - Properties Not Object**
```json
{
  "error": "BadRequest",
  "message": "Properties must be an object",
  "statusCode": 400
}
```

**400 Bad Request - Custom ID Provided**
```json
{
  "error": "BadRequest",
  "message": "Custom IDs are not allowed. IDs are automatically generated by the system.",
  "statusCode": 400
}
```

**400 Bad Request - Variant Exclusivity**
```json
{
  "error": "BadRequest",
  "message": "Cannot provide both variantId and variantPartnershipPriceId. Provide only one.",
  "statusCode": 400
}
```

**401 Unauthorized**
```json
{
  "error": "Unauthorized",
  "message": "Organization context required",
  "statusCode": 401
}
```

**404 Not Found - Order**
```json
{
  "error": "NotFound",
  "message": "Order with id ord_xxxxx not found",
  "statusCode": 404
}
```

**404 Not Found - Partnership**
```json
{
  "error": "NotFound",
  "message": "Partnership with id pship_xxxxx not found",
  "statusCode": 404
}
```

**404 Not Found - Variant Partnership Price**
```json
{
  "error": "NotFound",
  "message": "Variant Partnership Price with id vpp_xxxxx not found",
  "statusCode": 404
}
```

---

## Advanced Features

### Vector Embeddings & Semantic Search

While not exposed directly in the HTTP API, the Orders DAL layer supports semantic search via vector embeddings. This capability can be integrated into the API in the future for natural language order searches. See [`packages/dal/src/orders.ts`](../../../packages/dal/src/orders.ts) for implementation details.

### Caching Strategy

The Orders module implements intelligent caching to optimize performance:

- **Resource-level caching** - Individual orders are cached after retrieval
- **List query caching** - Order lists with specific filters are cached
- **Cache busting** - Caches are automatically invalidated on create, update, or delete operations
- **Filter-aware hashing** - Different filter combinations get separate cache entries

This reduces database load for frequently accessed orders while ensuring data consistency on mutations.

---

## Test Coverage

The Orders API includes 28 comprehensive integration tests covering all features:

### Phase 1: Create Foundation (Tests 1-5)
- Create minimal order
- Create order with full fields
- Create order with properties
- Create order with single line (variantId resolution)
- Create order with multiple lines

### Phase 2: Basic Updates (Tests 6-9)
- Update simple fields (status, references, notes)
- Update financial fields (totalAmount, currency)
- Update delivery address
- Update status changes

### Phase 3: Properties Merge (Tests 10-12)
- Add new property keys
- Update existing property values
- Delete properties by setting to null

### Phase 4: Line Diffing by ID (Tests 13-15)
- Match line by ID and update
- Update multiple lines by ID
- Create new line alongside existing

### Phase 5: Line Diffing by Description (Tests 16-18)
- Match lines by description (case-insensitive)
- Delete all lines (empty array)
- Re-add lines for subsequent tests

### Phase 6: Line Properties Merge (Tests 19-21)
- Add new line properties
- Update existing line properties
- Delete line properties by setting to null

### Phase 7: Filtering & Pagination (Tests 22-25)
- Status filtering (accepted vs cancelled)
- Pagination (limit, offset, multiple pages)
- Sorting (createdAt asc/desc)
- Properties filtering (value match, existence checks)

### Phase 8: Edge Cases (Tests 26-28)
- Complex update (all operations at once)
- Case-insensitive description matching edge cases
- Soft delete verification

Run all tests with:
```bash
bun run tests/orders/orders.test.ts
```

---

## Related Documentation

- [Database Schema](../../../packages/database/src/schema/orders.ts)
- [Data Access Layer](../../../packages/dal/src/orders.ts)
- [Test Suite](../../tests/orders/orders.test.ts)
