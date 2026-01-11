# Products API Reference

## Overview

The Products API provides complete management of products and their variants, with intelligent variant diffing, flexible properties storage, and semantic search capabilities via vector embeddings.

**Base Path:** `/products`

**Authentication:** All requests require:
- `Authorization: Bearer <token>` header
- `x-org-id: <organization-id>` header

**Key Features:**
- Create, read, update, and delete products with nested variants
- Intelligent variant matching by ID or name (case-insensitive)
- Flexible properties merge on partial updates
- Soft deletes (data preserved in database)
- Vector embeddings for semantic search
- Comprehensive caching for performance
- Event publishing for audit trails and integrations

---

## Core Concepts

### Products and Variants Architecture

**Products** are top-level containers that represent your offerings:
- Each product belongs to a single organization
- Products have metadata: name, code, description, status
- Products can have multiple variants

**Variants** are SKU-level entities representing product dimensions:
- Each variant has a unique combination of attributes
- Variants include pricing, dimensions, and weight information
- Variants can have custom properties

**Properties System:**
Both products and variants support flexible JSON properties for storing custom attributes. Properties are merged on PATCH operations, allowing you to add, update, or delete individual keys without affecting others.

### Key Behaviors

#### Variant Diffing

When updating a product with variants, the API intelligently matches incoming variants to existing ones:

1. **By ID (Priority 1):** If an incoming variant includes an `id` field, it will match against an existing variant with that ID
2. **By Name (Priority 2):** If no ID is provided, the API matches by name (case-insensitive)
3. **Creates:** Incoming variants that don't match any existing variant are created
4. **Deletes:** Existing variants not matched by any incoming variant are soft-deleted
5. **Updates:** Matched variants are updated with the new values

This design allows flexible client implementations: some can track variant IDs, others can identify variants by name.

#### Properties Merge

When updating a product or variant with properties:
- **Add:** Send new keys to add them to existing properties
- **Update:** Send existing keys with new values to update them
- **Delete:** Send an existing key with value `null` to remove it
- **Preserve:** Keys not mentioned in the update are unchanged

Example: If a product has `{ color: "blue", size: "medium" }` and you PATCH with `{ properties: { color: "red", material: null } }`, the result is `{ color: "red", size: "medium" }` (material removed).

#### Soft Deletes

DELETE operations don't remove records from the database. Instead, they set `deletedAt` to the current timestamp. Soft-deleted products:
- Cannot be retrieved via GET endpoints
- Are excluded from LIST results
- Preserve data for audit trails and recovery
- Cascade to all variants when a product is deleted

#### Vector Embeddings

Products and variants have auto-generated vector embeddings for semantic search:
- **Products:** Embedded from name + description
- **Variants:** Embedded from name + sku + barcode
- **Generation:** Asynchronous and non-blocking (doesn't delay response)
- **Use Case:** Power semantic search endpoints to find products by natural language queries

---

## Endpoints

### GET /products

List products with optional filtering, pagination, and sorting.

**Request:**
```
GET /products?status=active&limit=50&offset=0&sort=name&direction=asc
```

**Query Parameters:**

| Parameter | Type | Default | Constraints | Description |
|-----------|------|---------|-------------|-------------|
| `status` | string | (all) | "active" \| "inactive" | Filter by product status |
| `search` | string | - | - | Search in name, code, or description |
| `limit` | integer | 100 | 1-1000 | Number of products per page |
| `offset` | integer | 0 | ≥ 0 | Number of products to skip |
| `sort` | string | - | "name" \| "createdAt" \| "updatedAt" | Field to sort by |
| `direction` | string | "asc" | "asc" \| "desc" | Sort direction |

**Response:**
```json
{
  "data": [
    {
      "id": "prod_01G8ZJ5XK9ABCDEFGHIJ",
      "organizationId": "org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9",
      "name": "Cozy Winter Sweater",
      "code": "SWEATER-001",
      "description": "Warm winter sweater made from 100% merino wool",
      "status": "active",
      "properties": { "material": "wool", "color": "navy" },
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null,
      "variants": [
        {
          "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
          "productId": "prod_01G8ZJ5XK9ABCDEFGHIJ",
          "name": "Standard Size",
          "sku": "SWEATER-001-STD",
          "barcode": "1234567890123",
          "uom": "each",
          "defaultPrice": 79.99,
          "quantityInBaseUom": 1,
          "netWeightKg": 0.5,
          "grossWeightKg": 0.6,
          "lengthCm": 60,
          "widthCm": 45,
          "heightCm": 5,
          "properties": { "size": "standard" },
          "createdAt": "2024-10-05T14:23:00.000Z",
          "updatedAt": "2024-10-05T14:23:00.000Z",
          "deletedAt": null
        }
      ]
    }
  ],
  "count": 1
}
```

**Status Codes:**
- `200 OK` - Successfully retrieved products
- `400 Bad Request` - Invalid query parameters
- `401 Unauthorized` - Missing or invalid authentication

---

### GET /products/:id

Retrieve a specific product by ID, including all its variants.

**Request:**
```
GET /products/prod_01G8ZJ5XK9ABCDEFGHIJ
```

**Response:**
```json
{
  "id": "prod_01G8ZJ5XK9ABCDEFGHIJ",
  "organizationId": "org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9",
  "name": "Cozy Winter Sweater",
  "code": "SWEATER-001",
  "description": "Warm winter sweater made from 100% merino wool",
  "status": "active",
  "properties": { "material": "wool" },
  "createdAt": "2024-10-05T14:23:00.000Z",
  "updatedAt": "2024-10-05T14:23:00.000Z",
  "deletedAt": null,
  "variants": [
    {
      "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
      "productId": "prod_01G8ZJ5XK9ABCDEFGHIJ",
      "name": "Standard Size",
      "sku": "SWEATER-001-STD",
      "barcode": null,
      "uom": "each",
      "defaultPrice": 79.99,
      "quantityInBaseUom": null,
      "netWeightKg": null,
      "grossWeightKg": null,
      "lengthCm": null,
      "widthCm": null,
      "heightCm": null,
      "properties": {},
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null
    }
  ]
}
```

**Status Codes:**
- `200 OK` - Product found
- `401 Unauthorized` - Missing or invalid authentication
- `404 Not Found` - Product doesn't exist or is soft-deleted

---

### POST /products

Create a new product with optional variants.

**Request Body:**
```json
{
  "name": "Cozy Winter Sweater",
  "code": "SWEATER-001",
  "description": "Warm winter sweater made from 100% merino wool, perfect for cold weather",
  "status": "active",
  "properties": {
    "material": "wool",
    "color": "navy",
    "size": "one-size"
  },
  "variants": [
    {
      "name": "Standard Size",
      "sku": "SWEATER-001-STD",
      "barcode": "1234567890123",
      "uom": "each",
      "defaultPrice": 79.99,
      "quantityInBaseUom": 1,
      "netWeightKg": 0.5,
      "grossWeightKg": 0.6,
      "lengthCm": 60,
      "widthCm": 45,
      "heightCm": 5,
      "properties": { "size": "standard" }
    }
  ]
}
```

**Request Field Specifications:**

**Product Fields:**
| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `name` | string | Yes | 1-255 chars, no whitespace-only | Product name |
| `code` | string | No | 1-255 chars | Unique product code per organization |
| `description` | string | No | 1-4096 chars | Product description |
| `status` | string | Yes | "active" \| "inactive" | Product status |
| `properties` | object | No | - | Custom JSON properties |
| `variants` | array | No | - | Array of variant objects |

**Variant Fields:**
| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `name` | string | Yes | 1-255 chars, no whitespace-only | Variant name |
| `sku` | string | No | 1-255 chars | Stock keeping unit |
| `barcode` | string | No | 1-255 chars | Barcode identifier |
| `uom` | string | Yes | See UOM enum | Unit of measure |
| `defaultPrice` | number | Yes | ≥ 0 | Price for this variant |
| `quantityInBaseUom` | number | No | ≥ 0 | Quantity in base unit |
| `netWeightKg` | number | No | ≥ 0 | Net weight in kg |
| `grossWeightKg` | number | No | ≥ 0 | Gross weight in kg |
| `lengthCm` | number | No | ≥ 0 | Length in cm |
| `widthCm` | number | No | ≥ 0 | Width in cm |
| `heightCm` | number | No | ≥ 0 | Height in cm |
| `properties` | object | No | - | Custom JSON properties |

**UOM (Unit of Measure) Values:**
- `each` - Individual items
- `kg` - Kilograms
- `g` - Grams
- `litre` - Liters
- `box` - Box quantities
- `carton` - Carton quantities
- `crate` - Crate quantities
- `pallet` - Pallet quantities

**Minimal Request:**
```json
{
  "name": "Simple Mug",
  "status": "active",
  "variants": [
    {
      "name": "Standard Variant",
      "uom": "each",
      "defaultPrice": 12.50
    }
  ]
}
```

**Response:** Same as GET /products/:id

**Status Codes:**
- `200 OK` - Product created successfully
- `400 Bad Request` - Validation error (e.g., custom ID provided, whitespace-only name)
- `401 Unauthorized` - Missing or invalid authentication

---

### PATCH /products/:id

Update an existing product with intelligent variant diffing and properties merge.

**Request:**
```
PATCH /products/prod_01G8ZJ5XK9ABCDEFGHIJ
```

**Request Body:**
```json
{
  "name": "Updated Product Name",
  "status": "inactive",
  "properties": {
    "newField": "newValue",
    "fieldToDelete": null
  },
  "variants": [
    {
      "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
      "name": "Updated Variant Name",
      "defaultPrice": 89.99
    },
    {
      "name": "New Variant",
      "sku": "NEW-SKU-001",
      "uom": "each",
      "defaultPrice": 59.99
    }
  ]
}
```

**Variant Diffing Logic:**

1. **Variants with ID:** Matched to existing variants by ID. Properties are merged.
2. **Variants without ID:** Matched to existing variants by name (case-insensitive). Properties are merged.
3. **New Variants:** Incoming variants without ID and no matching name are created.
4. **Deleted Variants:** Existing variants not matched by any incoming variant are soft-deleted.
5. **Empty Array:** Sending `variants: []` soft-deletes all variants.

**Properties Merge:**
- New keys in the properties object are added
- Existing keys with new values are updated
- Keys with value `null` are deleted
- Keys not mentioned are preserved

**Response:** Same as GET /products/:id

**Status Codes:**
- `200 OK` - Product updated successfully
- `400 Bad Request` - Validation error
- `401 Unauthorized` - Missing or invalid authentication
- `404 Not Found` - Product doesn't exist or is soft-deleted

---

### DELETE /products/:id

Soft delete a product and all its variants.

**Request:**
```
DELETE /products/prod_01G8ZJ5XK9ABCDEFGHIJ
```

**Response:**
```json
{
  "message": "success"
}
```

**Status Codes:**
- `200 OK` - Product deleted successfully
- `401 Unauthorized` - Missing or invalid authentication
- `404 Not Found` - Product doesn't exist

---

## Schema Reference

### Product Object

```typescript
interface Product {
  id: string;                    // System-generated, format: prod_*
  organizationId: string;        // From authentication context
  name: string;                  // 1-255 characters
  code?: string;                 // Optional, unique per org
  description?: string;          // Optional, 1-4096 characters
  status: "active" | "inactive"; // Required
  properties: Record<string, any>; // Flexible JSON
  createdAt: string;            // ISO 8601 timestamp
  updatedAt: string;            // ISO 8601 timestamp
  deletedAt: string | null;     // ISO 8601 timestamp or null
  variants: Variant[];          // Array of variant objects
}
```

### Variant Object

```typescript
interface Variant {
  id: string;                    // System-generated, format: prodvar_*
  productId: string;             // Parent product ID
  name: string;                  // 1-255 characters
  sku?: string;                  // Optional, 1-255 characters
  barcode?: string;              // Optional, 1-255 characters
  uom: string;                   // Unit of measure enum
  defaultPrice: number;          // Price (decimal returned as number)
  quantityInBaseUom?: number;    // Optional, min 0
  netWeightKg?: number;          // Optional, min 0
  grossWeightKg?: number;        // Optional, min 0
  lengthCm?: number;             // Optional, min 0
  widthCm?: number;              // Optional, min 0
  heightCm?: number;             // Optional, min 0
  properties: Record<string, any>; // Flexible JSON
  createdAt: string;             // ISO 8601 timestamp
  updatedAt: string;             // ISO 8601 timestamp
  deletedAt: string | null;      // ISO 8601 timestamp or null
}
```

---

## Test-Driven Examples

### Example 1: Minimal Product Creation

**Scenario:** Create a product with the minimum required fields.

**Request:**
```bash
curl -X POST http://localhost:3001/products \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9" \
  -d '{
    "name": "Test Product A",
    "status": "active"
  }'
```

**Expected Response (200 OK):**
```json
{
  "id": "prod_...",
  "organizationId": "org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9",
  "name": "Test Product A",
  "code": null,
  "description": null,
  "status": "active",
  "properties": {},
  "createdAt": "2024-10-05T14:23:00.000Z",
  "updatedAt": "2024-10-05T14:23:00.000Z",
  "deletedAt": null,
  "variants": []
}
```

**Key Points:**
- Minimal product requires only `name` and `status`
- Optional fields default to null
- Properties defaults to empty object
- Variants array can be omitted or empty
- IDs are auto-generated

---

### Example 2: Product with Multiple Variants

**Scenario:** Create a product with comprehensive metadata and multiple variants with different units of measure.

**Request:**
```bash
curl -X POST http://localhost:3001/products \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9" \
  -d '{
    "name": "Test Product E",
    "code": "PROD-E-001",
    "status": "active",
    "variants": [
      {
        "name": "Bulk Pack",
        "sku": "SKU-E-BULK-001",
        "barcode": "9876543210987",
        "uom": "box",
        "defaultPrice": 199.99,
        "quantityInBaseUom": 10,
        "netWeightKg": 5.5,
        "grossWeightKg": 6.2,
        "lengthCm": 30,
        "widthCm": 20,
        "heightCm": 15,
        "properties": {
          "boxCount": 10,
          "discount": 0.15
        }
      },
      {
        "name": "Pallet Quantity",
        "sku": "SKU-E-PALLET-001",
        "uom": "pallet",
        "defaultPrice": 4999.99,
        "quantityInBaseUom": 100,
        "netWeightKg": 500,
        "grossWeightKg": 550,
        "properties": {
          "boxesPerPallet": 100
        }
      }
    ]
  }'
```

**Expected Response (200 OK):**
```json
{
  "id": "prod_...",
  "organizationId": "org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9",
  "name": "Test Product E",
  "code": "PROD-E-001",
  "description": null,
  "status": "active",
  "properties": {},
  "createdAt": "2024-10-05T14:23:00.000Z",
  "updatedAt": "2024-10-05T14:23:00.000Z",
  "deletedAt": null,
  "variants": [
    {
      "id": "var_...",
      "productId": "prod_...",
      "name": "Bulk Pack",
      "sku": "SKU-E-BULK-001",
      "barcode": "9876543210987",
      "uom": "box",
      "defaultPrice": 199.99,
      "quantityInBaseUom": 10,
      "netWeightKg": 5.5,
      "grossWeightKg": 6.2,
      "lengthCm": 30,
      "widthCm": 20,
      "heightCm": 15,
      "properties": { "boxCount": 10, "discount": 0.15 },
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null
    },
    {
      "id": "var_...",
      "productId": "prod_...",
      "name": "Pallet Quantity",
      "sku": "SKU-E-PALLET-001",
      "barcode": null,
      "uom": "pallet",
      "defaultPrice": 4999.99,
      "quantityInBaseUom": 100,
      "netWeightKg": 500,
      "grossWeightKg": 550,
      "lengthCm": null,
      "widthCm": null,
      "heightCm": null,
      "properties": { "boxesPerPallet": 100 },
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null
    }
  ]
}
```

**Key Points:**
- Multiple variants created in single request
- Each variant can have different UOM
- Physical dimensions stored separately
- Properties are flexible and variant-specific
- Prices stored as decimals, returned as numbers

---

### Example 3: Variant Update by ID

**Scenario:** Update a specific variant using its ID to change pricing and SKU.

**Request:**
```bash
curl -X PATCH http://localhost:3001/products/prod_01G8ZJ5XK9ABCDEFGHIJ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9" \
  -d '{
    "variants": [
      {
        "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
        "name": "Standard Size",
        "sku": "SKU-D-STD-UPDATED",
        "defaultPrice": 59.99
      }
    ]
  }'
```

**Expected Response (200 OK):**
```json
{
  "id": "prod_01G8ZJ5XK9ABCDEFGHIJ",
  "organizationId": "org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9",
  "name": "Test Product D",
  "code": "PROD-D-001",
  "description": null,
  "status": "active",
  "properties": {},
  "createdAt": "2024-10-05T14:23:00.000Z",
  "updatedAt": "2024-10-05T14:23:00.000Z",
  "deletedAt": null,
  "variants": [
    {
      "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
      "productId": "prod_01G8ZJ5XK9ABCDEFGHIJ",
      "name": "Standard Size",
      "sku": "SKU-D-STD-UPDATED",
      "barcode": "1234567890123",
      "uom": "each",
      "defaultPrice": 59.99,
      "quantityInBaseUom": null,
      "netWeightKg": null,
      "grossWeightKg": null,
      "lengthCm": null,
      "widthCm": null,
      "heightCm": null,
      "properties": {},
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null
    }
  ]
}
```

**Key Points:**
- Include `id` field to match by ID (priority 1)
- Only send fields you want to update
- Other fields on the variant remain unchanged
- Partial updates are fully supported
- Only one variant in the array means other variants are unchanged

---

### Example 4: Variant Update by Name (Case-Insensitive)

**Scenario:** Update variants without using IDs, matching by name instead. Demonstrates flexibility for clients that don't track variant IDs.

**Request:**
```bash
curl -X PATCH http://localhost:3001/products/prod_01G8ZJ5XK9ABCDEFGHIJ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9" \
  -d '{
    "variants": [
      {
        "name": "bulk pack",
        "sku": "SKU-E-BULK-UPDATED",
        "uom": "box",
        "defaultPrice": 209.99
      },
      {
        "name": "PALLET QUANTITY",
        "uom": "pallet",
        "defaultPrice": 5099.99
      }
    ]
  }'
```

**Expected Response (200 OK):**
```json
{
  "id": "prod_01G8ZJ5XK9ABCDEFGHIJ",
  "organizationId": "org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9",
  "name": "Test Product E",
  "code": "PROD-E-001",
  "description": null,
  "status": "active",
  "properties": {},
  "createdAt": "2024-10-05T14:23:00.000Z",
  "updatedAt": "2024-10-05T14:23:00.000Z",
  "deletedAt": null,
  "variants": [
    {
      "id": "var_...",
      "productId": "prod_01G8ZJ5XK9ABCDEFGHIJ",
      "name": "Bulk Pack",
      "sku": "SKU-E-BULK-UPDATED",
      "barcode": "9876543210987",
      "uom": "box",
      "defaultPrice": 209.99,
      "quantityInBaseUom": 10,
      "netWeightKg": 5.5,
      "grossWeightKg": 6.2,
      "lengthCm": 30,
      "widthCm": 20,
      "heightCm": 15,
      "properties": { "boxCount": 10, "discount": 0.15 },
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null
    },
    {
      "id": "var_...",
      "productId": "prod_01G8ZJ5XK9ABCDEFGHIJ",
      "name": "Pallet Quantity",
      "sku": "SKU-E-PALLET-001",
      "barcode": null,
      "uom": "pallet",
      "defaultPrice": 5099.99,
      "quantityInBaseUom": 100,
      "netWeightKg": 500,
      "grossWeightKg": 550,
      "lengthCm": null,
      "widthCm": null,
      "heightCm": null,
      "properties": { "boxesPerPallet": 100 },
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null
    }
  ]
}
```

**Key Points:**
- Variant name matching is case-insensitive ("bulk pack" matches "Bulk Pack")
- Don't include `id` to use name-based matching
- Original name is preserved in database
- Other properties on matched variants are updated
- Matching happens first by ID (if provided), then by name

---

### Example 5: Properties Merge

**Scenario:** Add new properties, update existing ones, and delete specific keys without affecting other properties.

**Request:**
```bash
curl -X PATCH http://localhost:3001/products/prod_01G8ZJ5XK9ABCDEFGHIJ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9" \
  -d '{
    "properties": {
      "warehouse": "Building A",
      "priority": "high",
      "supplier": "New Supplier",
      "rating": null
    }
  }'
```

**Initial State:** `{ category: "Electronics", supplier: "Test Supplier", rating: 4.5 }`

**Expected Response (200 OK) - Properties:**
```json
{
  "properties": {
    "category": "Electronics",
    "supplier": "New Supplier",
    "warehouse": "Building A",
    "priority": "high"
  }
}
```

**Key Points:**
- `warehouse` and `priority` are new keys - they're added
- `supplier` is updated with a new value
- `rating` is set to `null` - the key is deleted
- `category` is unchanged - keys not mentioned are preserved
- Properties merge happens on both product and variant level

---

### Example 6: Variant Properties Merge

**Scenario:** Update variant properties while preserving existing keys.

**Request:**
```bash
curl -X PATCH http://localhost:3001/products/prod_01G8ZJ5XK9ABCDEFGHIJ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9" \
  -d '{
    "variants": [
      {
        "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
        "name": "Standard Pack",
        "properties": {
          "warehouse": "Zone A",
          "stockLevel": 500
        }
      }
    ]
  }'
```

**Initial Variant Properties:** `{ packSize: 5, color: "blue" }`

**Expected Response (200 OK) - Variant Properties:**
```json
{
  "properties": {
    "packSize": 5,
    "color": "blue",
    "warehouse": "Zone A",
    "stockLevel": 500
  }
}
```

**Then Update Further:**
```bash
curl -X PATCH http://localhost:3001/products/prod_01G8ZJ5XK9ABCDEFGHIJ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9" \
  -d '{
    "variants": [
      {
        "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
        "name": "Standard Pack",
        "properties": {
          "packSize": 6,
          "color": null
        }
      }
    ]
  }'
```

**Final Variant Properties:**
```json
{
  "properties": {
    "packSize": 6,
    "warehouse": "Zone A",
    "stockLevel": 500
  }
}
```

**Key Points:**
- Variant properties follow the same merge logic as product properties
- Use `null` to delete keys from variant properties
- Each variant maintains its own properties independently

---

### Example 7: Complex Multi-Field Update

**Scenario:** Update product name, status, and properties while updating existing variants and adding new ones in a single request.

**Request:**
```bash
curl -X PATCH http://localhost:3001/products/prod_01G8ZJ5XK9ABCDEFGHIJ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9" \
  -d '{
    "name": "Product E - Final Update",
    "status": "inactive",
    "properties": {
      "newField": "newValue",
      "deleted_category": null
    },
    "variants": [
      {
        "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
        "name": "Standard Pack",
        "defaultPrice": 119.99
      },
      {
        "name": "Final Variant",
        "sku": "SKU-E-FINAL-001",
        "uom": "each",
        "defaultPrice": 19.99
      }
    ]
  }'
```

**Expected Response (200 OK):**
```json
{
  "id": "prod_01G8ZJ5XK9ABCDEFGHIJ",
  "organizationId": "org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9",
  "name": "Product E - Final Update",
  "code": "PROD-E-001",
  "description": null,
  "status": "inactive",
  "properties": {
    "newField": "newValue"
  },
  "createdAt": "2024-10-05T14:23:00.000Z",
  "updatedAt": "2024-10-05T14:23:00.000Z",
  "deletedAt": null,
  "variants": [
    {
      "id": "var_01G8ZJ5XK9ABCDEFGHIJ",
      "productId": "prod_01G8ZJ5XK9ABCDEFGHIJ",
      "name": "Standard Pack",
      "sku": "SKU-E-STD-001",
      "barcode": null,
      "uom": "box",
      "defaultPrice": 119.99,
      "quantityInBaseUom": 10,
      "netWeightKg": 5.5,
      "grossWeightKg": 6.2,
      "lengthCm": 30,
      "widthCm": 20,
      "heightCm": 15,
      "properties": { "packSize": 6 },
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null
    },
    {
      "id": "var_...",
      "productId": "prod_01G8ZJ5XK9ABCDEFGHIJ",
      "name": "Final Variant",
      "sku": "SKU-E-FINAL-001",
      "barcode": null,
      "uom": "each",
      "defaultPrice": 19.99,
      "quantityInBaseUom": null,
      "netWeightKg": null,
      "grossWeightKg": null,
      "lengthCm": null,
      "widthCm": null,
      "heightCm": null,
      "properties": {},
      "createdAt": "2024-10-05T14:23:00.000Z",
      "updatedAt": "2024-10-05T14:23:00.000Z",
      "deletedAt": null
    }
  ]
}
```

**Key Points:**
- Product name, status, and properties all updated in single request
- Existing variant updated by ID
- New variant created (no ID, new name)
- Other variants not mentioned become soft-deleted
- All operations complete atomically

---

### Example 8: Status Filtering

**Scenario:** List only active products and then list only inactive products.

**List Active Products:**
```bash
curl -X GET "http://localhost:3001/products?status=active" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Response (200 OK):**
- Returns only products where `status: "active"`
- Excludes soft-deleted products
- Excludes products with `status: "inactive"`

**List Inactive Products:**
```bash
curl -X GET "http://localhost:3001/products?status=inactive" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Response (200 OK):**
- Returns only products where `status: "inactive"`
- Excludes soft-deleted products
- Excludes products with `status: "active"`

**Key Points:**
- Status filter is optional (omit to get all active products)
- Soft-deleted products never returned regardless of status
- Status values are case-sensitive: must be exactly "active" or "inactive"

---

### Example 9: Pagination

**Scenario:** Paginate through products using limit and offset.

**First Page (10 items):**
```bash
curl -X GET "http://localhost:3001/products?limit=10&offset=0&sort=createdAt&direction=asc" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Second Page (10 items):**
```bash
curl -X GET "http://localhost:3001/products?limit=10&offset=10&sort=createdAt&direction=asc" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Response Structure:**
```json
{
  "data": [ /* 10 product objects */ ],
  "count": 150
}
```

**Offset Beyond End:**
```bash
curl -X GET "http://localhost:3001/products?limit=10&offset=9999&sort=createdAt&direction=asc" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Response (200 OK):**
```json
{
  "data": [],
  "count": 150
}
```

**Key Points:**
- `limit`: 1-1000, default 100 if omitted
- `offset`: 0 or greater, default 0 if omitted
- `count` is consistent across pages
- No overlapping IDs between pages (pagination is deterministic)
- Empty data array when offset exceeds total count

---

### Example 10: Sorting

**Scenario:** Sort products by different fields in ascending and descending order.

**Sort by Name (Ascending):**
```bash
curl -X GET "http://localhost:3001/products?sort=name&direction=asc&limit=20" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Expected Order:** Products sorted alphabetically by name (case-insensitive)

**Sort by Name (Descending):**
```bash
curl -X GET "http://localhost:3001/products?sort=name&direction=desc&limit=20" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Expected Order:** Products sorted reverse alphabetically by name (case-insensitive)

**Sort by CreatedAt (Descending - Most Recent First):**
```bash
curl -X GET "http://localhost:3001/products?sort=createdAt&direction=desc&limit=20" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Expected Order:** Products sorted by creation date, newest first

**Sort by UpdatedAt (Ascending - Oldest First):**
```bash
curl -X GET "http://localhost:3001/products?sort=updatedAt&direction=asc&limit=20" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -H "x-org-id: org_f3c5b78a-e19e-40a4-9fdd-6a898dca1bf9"
```

**Expected Order:** Products sorted by last update, oldest modifications first

**Key Points:**
- `sort` can be: `name`, `createdAt`, or `updatedAt`
- `direction` can be: `asc` (ascending) or `desc` (descending)
- Default sort is by `createdAt` descending (most recent first)
- Sorting is case-insensitive for `name` field
- All sorts are deterministic (consistent ordering)

---

## Advanced Features

### Caching Strategy

The Products API implements a multi-level caching strategy for optimal performance:

**Resource Cache:**
- Key: `product:{organizationId}:{productId}`
- Stores individual product objects by ID
- TTL: 5 minutes (default, configurable)
- Used by GET /products/:id

**List Cache:**
- Key: `products:list:{organizationId}:{query_hash}`
- Stores filtered product lists with their query parameters
- TTL: 5 minutes (default, configurable)
- Used by GET /products (with any filter combination)
- Query parameters hashed to include: status, search, limit, offset, sort, direction

**Code Cache:**
- Key: `product:code:{organizationId}:{code}`
- Stores product lookup by code for quick access
- TTL: 5 minutes (default, configurable)
- Used internally by getByCode operations

**Cache Invalidation:**
When a product is created, updated, or deleted:
1. Resource cache for that product is cleared
2. All list caches for that organization are cleared (all filter combinations)
3. Code cache entries are cleared if code changed or product deleted

This ensures consistency: mutations always reflect immediately, but subsequent reads benefit from caching.

### Vector Embeddings

**Purpose:** Enable semantic search to find products by natural language queries.

**Auto-Generation:**
- Products: Embedding generated from `name + description`
- Variants: Embedding generated from `name + sku + barcode`
- Generation is asynchronous (non-blocking)
- Doesn't delay API response
- Generated on create and whenever name/description (product) or name/sku/barcode (variant) changes

**Embedding Dimension:** 1536 (OpenAI ada-002 model)

**Similarity Search:**
When enabled, you can search products using:
```bash
GET /products/search?q=winter+sweater&limit=10
```

Returns products ranked by semantic similarity to the query.

**Storage:**
- Embeddings stored in PostgreSQL with `pgvector` extension
- Not returned in API responses (embeddings field explicitly excluded)
- Supports vector distance calculations

### Event Publishing

**Purpose:** Notify external systems of product changes for audit trails, integrations, and workflows.

**Events Published:**
- `product.created` - When a product is created
- `product.updated` - When a product is updated
- `product.deleted` - When a product is soft-deleted
- `productVariant.created` - When a variant is created
- `productVariant.updated` - When a variant is updated
- `productVariant.deleted` - When a variant is soft-deleted

**Event Properties:**
```json
{
  "resource": "product" | "productVariant",
  "action": "create" | "update" | "delete",
  "resourceId": "prod_... or var_...",
  "organizationId": "org_...",
  "timestamp": "2024-10-05T14:23:00.000Z"
}
```

**Delivery:**
- Events published asynchronously (non-blocking)
- Decoupled from API response
- Can be consumed by event listeners or webhook systems
- Useful for inventory management, analytics, and notifications

---

## Error Handling

### Error Response Format

All error responses follow this format:

```json
{
  "error": "ErrorType",
  "message": "Human-readable error message",
  "statusCode": 400
}
```

### Common Error Codes

#### 400 Bad Request

**Malformed JSON:**
```
POST /products
Body: { invalid json }

Response (400):
{
  "error": "BadRequest",
  "message": "Invalid JSON",
  "statusCode": 400
}
```

**Whitespace-Only Name:**
```
POST /products
Body: { "name": "   ", "status": "active" }

Response (400):
{
  "error": "BadRequest",
  "message": "Product name cannot be empty or whitespace-only",
  "statusCode": 400
}
```

**Custom ID Provided:**
```
POST /products
Body: { "id": "custom_id", "name": "Product", "status": "active" }

Response (400):
{
  "error": "BadRequest",
  "message": "Custom IDs are not allowed. IDs are automatically generated by the system.",
  "statusCode": 400
}
```

**Invalid Status Value:**
```
GET /products?status=pending

Response (400):
{
  "error": "BadRequest",
  "message": "Invalid status value: \"pending\". Valid values are: active, inactive",
  "statusCode": 400
}
```

**Invalid Properties Type:**
```
PATCH /products/:id
Body: { "properties": ["array", "instead", "of", "object"] }

Response (400):
{
  "error": "BadRequest",
  "message": "Properties must be an object",
  "statusCode": 400
}
```

**Invalid Limit Parameter:**
```
GET /products?limit=5000

Response (400):
{
  "error": "BadRequest",
  "message": "Invalid limit value: \"5000\". Limit must be between 1 and 1000.",
  "statusCode": 400
}
```

#### 401 Unauthorized

**Missing Authorization Header:**
```
GET /products
(no Authorization header)

Response (401):
{
  "error": "Unauthorized",
  "message": "Missing authorization credentials",
  "statusCode": 401
}
```

**Missing Organization Header:**
```
GET /products
Authorization: Bearer YOUR_TOKEN
(no x-org-id header)

Response (401):
{
  "error": "Unauthorized",
  "message": "Organization context required",
  "statusCode": 401
}
```

#### 404 Not Found

**Product Doesn't Exist:**
```
GET /products/prod_nonexistent

Response (404):
{
  "error": "NotFound",
  "message": "Product prod_nonexistent not found",
  "statusCode": 404
}
```

**Product Is Soft-Deleted:**
```
GET /products/prod_deleted_id

Response (404):
{
  "error": "NotFound",
  "message": "Product prod_deleted_id not found",
  "statusCode": 404
}
```

#### 500 Internal Server Error

**Database Connection Error:**
```
Response (500):
{
  "error": "InternalServerError",
  "message": "An unexpected error occurred",
  "statusCode": 500
}
```

### Error Handling Best Practices

1. **Always Check Status Codes:** Don't assume success. Check HTTP status code.
2. **Validate Input:** Pre-validate on client before sending to reduce 400 errors.
3. **Handle Retries:** 5xx errors are usually temporary. Implement exponential backoff.
4. **Log Error Details:** Log the full error object for debugging.
5. **Don't Display Raw Errors:** Show user-friendly messages, log technical details.

---

## Implementation Notes

### For API Consumers

**Authentication:**
- Always include both `Authorization: Bearer <token>` and `x-org-id: <org-id>` headers
- Tokens are organization-specific
- Invalid tokens result in 401 Unauthorized

**ID Management:**
- System generates IDs automatically (format: `prod_*`, `var_*`)
- Never send custom IDs - they will be rejected
- Always use returned IDs for subsequent operations
- IDs are globally unique across organization

**PATCH Idempotency:**
- PATCH operations are safe to retry
- Sending the same update twice produces the same result
- Useful for handling network failures

**Variant Matching:**
- Prefer sending variant IDs if available (more reliable)
- If not tracking IDs, use distinct variant names
- Duplicate names may cause matching issues

**Properties Usage:**
- Use properties for flexible, custom metadata
- Set to `null` to delete keys (not recommended for production critical data)
- Properties don't affect sorting or filtering (use dedicated fields instead)
- Maximum property value size depends on PostgreSQL limits

**Search:**
- Use `search` parameter for full-text search in name/code/description
- Use semantic search endpoint for natural language queries (future feature)
- Case-insensitive matching for names

### Database Layer

**Technology Stack:**
- PostgreSQL for persistence
- Drizzle ORM for type-safe queries
- pgvector extension for embeddings

**Soft Deletes:**
- Data never physically deleted (except admin operations)
- `deletedAt` timestamp marks deletion
- All queries filter out soft-deleted records automatically
- Recovery possible by clearing `deletedAt`

**Type Conversion:**
- Numeric fields (prices, weights, dimensions) stored as PostgreSQL `decimal` type
- Returned to API as JavaScript `number` type
- Preserve precision up to 2 decimal places for prices

**Timestamp Handling:**
- All timestamps stored as UTC in PostgreSQL
- Returned as ISO 8601 strings in API
- `null` value for `deletedAt` indicates active record

**Relationships:**
- `product` table has `organizationId` foreign key
- `productVariant` table has `productId` foreign key
- Referential integrity enforced at database level
- Cascade delete on product delete (hard delete at DB level handles soft-deleted variants)

### Performance Considerations

**Query Optimization:**
- Indexes on `organizationId`, `status`, `createdAt`, `deletedAt`
- Indexes on variant `productId` for fast lookups
- Query API (Drizzle) uses efficient JOINs

**Pagination:**
- Default limit of 100 prevents large result sets
- Max limit of 1000 for advanced scenarios
- Consistent pagination via offset-based approach
- Always include sorting for deterministic results

**Caching:**
- Redis-based caching for reduced database hits
- Automatic cache invalidation on mutations
- Significantly improves list performance

**Concurrency:**
- Parallel variant operations on PATCH (creates, updates, deletes)
- Database transactions ensure consistency
- No race conditions between product and variant operations

**Async Operations:**
- Embedding generation async (doesn't block response)
- Event publishing async (doesn't block response)
- Both operations retried on failure

### Monitoring and Debugging

**Logging:**
- All operations logged with correlation IDs
- Timing information included for performance monitoring
- Debug logs include query details and transformations

**Correlation IDs:**
- Unique ID per request for tracing
- Passed through to DAL layer
- Useful for debugging multi-step operations

**Performance Metrics:**
- Response times logged for all endpoints
- Cache hit/miss rates trackable
- Database query times available in logs

---

## Summary

The Products API provides a robust, scalable foundation for product management. Key differentiators:

1. **Intelligent Variant Matching:** ID-first, then name-based matching for flexible integrations
2. **Properties Merge:** Flexible custom data without schema changes
3. **Comprehensive Caching:** Performance optimized with automatic invalidation
4. **Event-Driven:** Integration-ready with async events
5. **Type Safety:** Fully typed with validation at multiple layers
6. **Soft Deletes:** Data preservation with recovery capability
7. **Vector Embeddings:** Ready for semantic search

For questions or issues, refer to the test suite at `tests/products/products.test.ts` for comprehensive usage examples.
