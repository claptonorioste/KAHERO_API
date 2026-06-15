# Kahero GraphQL API — Integration Guide

This guide covers authentication and the most common queries/mutations for integrations: login, devices, items (CRUD), transactions by date range, and Z/X readings.

---

## Base URL & Headers

| Setting | Value |
|--------|--------|
| **Endpoint** | `POST https://graphql.kahero.co/` |
| **Content-Type** | `application/json` |
| **Auth (protected operations)** | `Authorization: Bearer <token>` |

The token returned from the `login` mutation is a Firebase ID token. Pass it on every authenticated request.

**Example HTTP request body:**

```json
{
  "query": "query { getDevices { id device_name } }",
  "variables": {}
}
```

**Example cURL:**

```bash
curl -X POST "https://<your-graphql-host>/" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN_HERE" \
  -d '{"query":"query { getDevices { id device_name } }"}'
```

---

## 1. Get Token — Login (Email & Password)

Authenticates with Firebase email/password and returns a JWT token plus the user ID.

### Mutation

```graphql
mutation Login($email: String!, $password: String!) {
  login(email: $email, password: $password) {
    token
    userId
  }
}
```

### Variables

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `email` | String | Yes | Account email |
| `password` | String | Yes | Account password |

```json
{
  "email": "merchant@example.com",
  "password": "your-password"
}
```

### Sample Response

```json
{
  "data": {
    "login": {
      "token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
      "userId": "abc123FirebaseUid"
    }
  }
}
```

### Sample Error Response

```json
{
  "errors": [
    {
      "message": "Authentication failed",
      "path": ["login"]
    }
  ],
  "data": null
}
```

> **Note:** Some accounts may require OTP via `loginWithOTP`. For standard email/password integrations, use `login`.

---

## 2. Get Devices

Returns all POS devices registered to the authenticated user.

### Query

```graphql
query GetDevices {
  getDevices {
    id
    device_name
    address
    header
    footer
    startTime
    endTime
    userId
    salesWarehouseId
    onlineOrderWarehouseId
    toPay
    isDisabled
    imageUrl
    urlName
    bir_compatible
    onlineCatalog
    canceledAt
  }
}
```

### Parameters

None. Uses the authenticated user's ID from the Bearer token.

### Sample Response

```json
{
  "data": {
    "getDevices": [
      {
        "id": "665a1b2c3d4e5f6789012345",
        "device_name": "Main Counter",
        "address": "123 Store St, Manila",
        "header": "Welcome to My Store",
        "footer": "Thank you!",
        "startTime": 8,
        "endTime": 22,
        "userId": "abc123FirebaseUid",
        "salesWarehouseId": "warehouse-firebase-id-1",
        "onlineOrderWarehouseId": null,
        "toPay": false,
        "isDisabled": false,
        "imageUrl": "https://storage.example.com/device-logo.png",
        "urlName": "my-store",
        "bir_compatible": {
          "company_info": {
            "vatRegistered": "Yes"
          }
        },
        "onlineCatalog": null,
        "canceledAt": null
      }
    ]
  }
}
```

### Get Single Device by ID

```graphql
query GetDeviceById($id: String!) {
  getDeviceById(id: $id) {
    id
    device_name
    salesWarehouseId
    bir_compatible
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `id` | String | Yes | Device MongoDB `_id` |

---

## 3. Item CRUD

Items are scoped to the authenticated user. The primary identifier for updates/deletes is `firebaseId`.

### 3.1 Read — List All Items

```graphql
query AllItems($uid: String, $isAsc: Boolean) {
  allitems(uid: $uid, isAsc: $isAsc) {
    id
    firebaseId
    name
    price
    cost
    category
    categoryId
    barcode
    sku
    saleItem
    managedItemStock
    isNotAvailable
    taxList {
      name
      rate
      type
      value
    }
    createdAt
    updatedAt
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `uid` | String | No | User ID override (defaults to token user) |
| `isAsc` | Boolean | No | Sort by `createdAt` ascending if `true`, else descending |

### 3.2 Read — Single Item by Firebase ID

```graphql
query GetItem($firebaseId: String!) {
  item(firebaseId: $firebaseId) {
    id
    firebaseId
    name
    price
    cost
    category
    categoryId
    barcode
    modifiers
    taxList {
      name
      rate
      type
      value
    }
    rawMaterialList {
      itemKey
      itemName
      qty
    }
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `firebaseId` | String | Yes | Item's `firebaseId` |

### 3.3 Read — Item by ID (alias)

```graphql
query GetItemById($itemId: String!, $userId: String) {
  getItemById(itemId: $itemId, userId: $userId) {
    id
    firebaseId
    name
    price
    category
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `itemId` | String | Yes | Same as `firebaseId` |
| `userId` | String | No | User ID override |

### 3.4 Create Item

```graphql
mutation CreateItem($input: ItemInput!) {
  createItem(input: $input) {
    id
    firebaseId
    name
    price
    cost
    category
    categoryId
    barcode
    saleItem
    createdAt
  }
}
```

**Variables example:**

```json
{
  "input": {
    "name": "Iced Latte",
    "price": 150,
    "cost": 60,
    "category": "Beverages",
    "categoryId": "category-firebase-id",
    "barcode": "1234567890123",
    "sku": "LATTE-001",
    "saleItem": true,
    "managedItemStock": false,
    "vatableItem": true,
    "nonVatItem": false,
    "taxList": [
      {
        "name": "VAT",
        "rate": 12,
        "type": "percentage",
        "value": true
      }
    ]
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `input` | ItemInput | Yes | Item fields (see ItemInput fields below) |
| `userId` | String | No | User ID override |

**Sample Response:**

```json
{
  "data": {
    "createItem": {
      "id": "665b2c3d4e5f67890123456",
      "firebaseId": "665b2c3d4e5f67890123457",
      "name": "Iced Latte",
      "price": 150,
      "cost": 60,
      "category": "Beverages",
      "categoryId": "category-firebase-id",
      "barcode": "1234567890123",
      "saleItem": true,
      "createdAt": "2026-06-15T08:00:00.000Z"
    }
  }
}
```

> **Validation:** `input.name` is required and cannot be empty.

### 3.5 Update Item

```graphql
mutation UpdateItem($firebaseId: String!, $input: ItemInput!) {
  updateItem(firebaseId: $firebaseId, input: $input) {
    id
    firebaseId
    name
    price
    cost
    updatedAt
  }
}
```

**Variables example:**

```json
{
  "firebaseId": "665b2c3d4e5f67890123457",
  "input": {
    "name": "Iced Latte (Large)",
    "price": 175,
    "cost": 65
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `firebaseId` | String | Yes | Item to update |
| `input` | ItemInput | Yes | Fields to change |
| `userId` | String | No | User ID override |

**Sample Response:**

```json
{
  "data": {
    "updateItem": {
      "id": "665b2c3d4e5f67890123456",
      "firebaseId": "665b2c3d4e5f67890123457",
      "name": "Iced Latte (Large)",
      "price": 175,
      "cost": 65,
      "updatedAt": "2026-06-15T09:30:00.000Z"
    }
  }
}
```

### 3.6 Delete Item(s)

There is no single-item delete mutation. Use `deleteManyItem` with a JSON array of `firebaseId` values.

```graphql
mutation DeleteManyItem($userId: String!, $itemIds: String!) {
  deleteManyItem(userId: $userId, itemIds: $itemIds)
}
```

**Variables example:**

```json
{
  "userId": "abc123FirebaseUid",
  "itemIds": "[\"665b2c3d4e5f67890123457\", \"665b2c3d4e5f67890123458\"]"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | String | Yes | Owner user ID |
| `itemIds` | String | Yes | JSON string array of `firebaseId` values |

**Sample Response:**

```json
{
  "data": {
    "deleteManyItem": []
  }
}
```

> Deletes the items, their child option items (`parentItemId`), and related bundle records.

### ItemInput — Available Fields

| Field | Type | Description |
|-------|------|-------------|
| `name` | String | Item name (**required on create**) |
| `price` | Float | Selling price |
| `cost` | Float | Cost price |
| `barcode` | String | Barcode |
| `sku` | String | SKU |
| `category` | String | Category name |
| `categoryId` | String | Category firebase ID |
| `firebaseId` | String | Custom ID (auto-generated if omitted on create) |
| `saleItem` | Boolean | Available for sale |
| `managedItemStock` | Boolean | Track inventory |
| `vatableItem` | Boolean | VAT applicable |
| `nonVatItem` | Boolean | Non-VAT item |
| `isNotAvailable` | Boolean | Mark unavailable |
| `isBundle` | Boolean | Bundle item |
| `isParentItem` | Boolean | Has variants/options |
| `parentItemId` | String | Parent item firebase ID |
| `modifiers` | [String] | Modifier IDs |
| `taxList` | [ItemTaxListInput] | Tax configuration |
| `rawMaterialList` | [ItemRawMaterialListInput] | Recipe/raw materials |
| `priceList` | String | JSON string for price tiers |
| `tags` | [String] | Tags |
| `imagePath` | String | Image path |
| `unit` | String | Unit of measure |
| `discountValue` | Float | Discount amount |
| `onlineOrderPrice` | Float | Online catalog price |

---

## 4. getTransactionsByDateRange

Returns sales transactions for a date range, optionally filtered by device or warehouse.

### Query

```graphql
query GetTransactionsByDateRange(
  $startDate: String!
  $endDate: String!
  $userId: String
  $deviceId: String
  $warehouseId: String
  $orderBy: String
) {
  getTransactionsByDateRange(
    startDate: $startDate
    endDate: $endDate
    userId: $userId
    deviceId: $deviceId
    warehouseId: $warehouseId
    orderBy: $orderBy
  ) {
    id
    firebaseId
    userId
    deviceId
    warehouseId
    createdAt
    modifiedAt
    grandtotal
    subtotal
    paymentType
    paymentName
    refund
    orNumber
    transactionNumber
    referenceNumber
    diningOption
    cashierId
    customerId
    posName
    cashier
    customer
    discounts
    taxes
    vatDetails {
      vatableTotal
      vatExemptTotal
      totalWithVat
    }
    seniorPwdDiscount {
      totalCustomers
      totalSeniors
      totalPwds
      totalDiscount
    }
    payments {
      name
      amount
      paymentType
      refNo
    }
    transactions
  }
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `startDate` | String | Yes | Start date (ISO or parseable date string, e.g. `"2026-06-01"`) |
| `endDate` | String | Yes | End date. Server sets time to `23:59:59.999` on this date |
| `userId` | String | No | User ID override (defaults to token user) |
| `deviceId` | String | No | Filter by device ID. Ignored if empty, `"undefined"`, or `"All Devices"` |
| `warehouseId` | String | No | Filter by warehouse. Ignored if empty, `"undefined"`, or `"All Warehouses"` |
| `orderBy` | String | No | `"asc"` or `"desc"` by `createdAt` (default: `"desc"`) |

**Variables example:**

```json
{
  "startDate": "2026-06-01",
  "endDate": "2026-06-15",
  "deviceId": "665a1b2c3d4e5f6789012345",
  "orderBy": "desc"
}
```

### Sample Response

```json
{
  "data": {
    "getTransactionsByDateRange": [
      {
        "id": "665c3d4e5f6789012345678",
        "firebaseId": "txn-firebase-id-001",
        "userId": "abc123FirebaseUid",
        "deviceId": "665a1b2c3d4e5f6789012345",
        "warehouseId": "warehouse-firebase-id-1",
        "createdAt": "2026-06-15T10:30:00.000Z",
        "modifiedAt": "2026-06-15T10:30:00.000Z",
        "grandtotal": 336,
        "subtotal": 300,
        "paymentType": "Cash",
        "paymentName": "Cash",
        "refund": false,
        "orNumber": 1001,
        "transactionNumber": 1001,
        "referenceNumber": null,
        "diningOption": "Dine In",
        "cashierId": "cashier-firebase-id",
        "customerId": null,
        "posName": "Main Counter",
        "cashier": "juan.delacruz",
        "customer": null,
        "discounts": null,
        "taxes": 36,
        "vatDetails": {
          "vatableTotal": 300,
          "vatExemptTotal": 0,
          "totalWithVat": 336
        },
        "seniorPwdDiscount": null,
        "payments": [
          {
            "name": "Cash",
            "amount": 336,
            "paymentType": "Cash",
            "refNo": null
          }
        ],
        "transactions": [
          {
            "itemName": "Iced Latte",
            "qty": 2,
            "price": 150,
            "total": 300
          }
        ]
      }
    ]
  }
}
```

---

## 5. getZreadingByDateRange

Returns a **Z Reading** report for a date range as a **JSON string**. Parse the response with `JSON.parse()` on the client.

Aggregates shift details, transactions, VAT, payment types, category sales, and items sold.

### Query

```graphql
query GetZreadingByDateRange(
  $startDate: String!
  $endDate: String!
  $birCompatible: Boolean!
  $deviceId: String
  $uid: String
) {
  getZreadingByDateRange(
    startDate: $startDate
    endDate: $endDate
    birCompatible: $birCompatible
    deviceId: $deviceId
    uid: $uid
  )
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `startDate` | String | Yes | Start of range |
| `endDate` | String | Yes | End of range |
| `birCompatible` | Boolean | Yes | Whether BIR-compatible calculations apply |
| `deviceId` | String | No | Filter by POS device ID |
| `uid` | String | No | User ID override |

**Variables example:**

```json
{
  "startDate": "2026-06-15",
  "endDate": "2026-06-15",
  "birCompatible": true,
  "deviceId": "665a1b2c3d4e5f6789012345"
}
```

### Sample Response

The field value is a stringified JSON object:

```json
{
  "data": {
    "getZreadingByDateRange": "{\"openAmount\":1000,\"actualAmount\":5500,\"expectedAmount\":5400,\"differenceAmount\":100,\"paidInAmount\":0,\"paidOutAmount\":0,\"cashSales\":4500,\"cashRefunds\":0,\"totalTransactions\":25,\"totalTendered\":5500,\"refunds\":0,\"taxes\":360,\"serviceFee\":0,\"discounts\":50,\"mobileSales\":500,\"chequeSales\":0,\"creditSales\":0,\"qrSales\":500,\"grossSales\":5500,\"addedTax\":360,\"netSales\":5090,\"diningOptionFee\":0,\"vatableTotal\":4500,\"vatExemptTotal\":0,\"vatTotal\":360,\"zeroRatedSales\":0,\"splitPayments\":{},\"categorySales\":[{\"category\":\"Beverages\",\"total\":3000}],\"itemsSold\":[{\"itemName\":\"Iced Latte\",\"qty\":20,\"total\":3000}],\"firstTransaction\":1001,\"lastTransaction\":1025,\"seniorPwdDiscount\":{\"totalCustomers\":2,\"totalSeniors\":1,\"totalPwds\":1},\"shiftTransac\":[{\"id\":\"shift-id-1\",\"openAmount\":\"1000\",\"cashPayments\":\"4500\",\"totalTransactions\":\"25\"}]}"
  }
}
```

**Parsed object structure (key fields):**

| Field | Type | Description |
|-------|------|-------------|
| `openAmount` | Number | Opening cash |
| `actualAmount` | Number | Actual cash counted |
| `expectedAmount` | Number | Expected cash |
| `differenceAmount` | Number | Over/short |
| `cashSales` | Number | Cash sales total |
| `cashRefunds` | Number | Cash refunds |
| `totalTransactions` | Number | Transaction count |
| `grossSales` | Number | Gross sales |
| `netSales` | Number | Net sales |
| `taxes` | Number | Total taxes |
| `discounts` | Number | Total discounts |
| `mobileSales` | Number | Mobile wallet sales |
| `chequeSales` | Number | Cheque sales |
| `creditSales` | Number | Credit sales |
| `qrSales` | Number | QR payment sales |
| `vatableTotal` | Number | VATable sales |
| `vatExemptTotal` | Number | VAT-exempt sales |
| `vatTotal` | Number | VAT amount |
| `categorySales` | Array | Sales by category |
| `itemsSold` | Array | Items sold summary |
| `firstTransaction` | Number | First OR/transaction number |
| `lastTransaction` | Number | Last OR/transaction number |
| `seniorPwdDiscount` | Object | Senior/PWD discount totals |
| `shiftTransac` | Array | Per-shift breakdown |

### Related: getZReading (Structured Response)

If you prefer a typed GraphQL response instead of a JSON string, use `getZReading`:

```graphql
query GetZReading(
  $startDate: String!
  $endDate: String!
  $birCompatible: Boolean!
  $deviceId: String
  $uid: String
) {
  getZReading(
    startDate: $startDate
    endDate: $endDate
    birCompatible: $birCompatible
    deviceId: $deviceId
    uid: $uid
  ) {
    openAmount
    actualAmount
    expectedAmount
    differenceAmount
    cashSales
    cashRefunds
    totalTransactions
    grossSales
    netSales
    taxes
    discounts
    mobileSales
    chequeSales
    creditSales
    qrSales
    vatableTotal
    vatExemptTotal
    vatTotal
    firstTransaction
    lastTransaction
    categorySales {
      category
      total
    }
    itemSold {
      itemName
      qty
      total
    }
    shiftTransact {
      openAmount
      actualAmount
      cashPayments
      totalTransactions
      grandTotal
    }
  }
}
```

Same parameters as `getZreadingByDateRange`.

**Sample Response:**

```json
{
  "data": {
    "getZReading": {
      "openAmount": 1000,
      "actualAmount": 5500,
      "expectedAmount": 5400,
      "differenceAmount": 100,
      "cashSales": 4500,
      "cashRefunds": 0,
      "totalTransactions": 25,
      "grossSales": 5500,
      "netSales": 5090,
      "taxes": 360,
      "discounts": 50,
      "mobileSales": 500,
      "chequeSales": 0,
      "creditSales": 0,
      "qrSales": 500,
      "vatableTotal": 4500,
      "vatExemptTotal": 0,
      "vatTotal": 360,
      "firstTransaction": 1001,
      "lastTransaction": 1025,
      "categorySales": [
        { "category": "Beverages", "total": 3000 }
      ],
      "itemSold": [
        { "itemName": "Iced Latte", "qty": 20, "total": 3000 }
      ],
      "shiftTransact": {
        "openAmount": 1000,
        "actualAmount": 5500,
        "cashPayments": 4500,
        "totalTransactions": 25,
        "grandTotal": 5500
      }
    }
  }
}
```

---

## 6. GetXreadingByDateRange

> **Important:** `getXreadingByDateRange` is **not currently exposed** in this GraphQL API. There is no query with that exact name in the schema.

For **interim / per-shift (X Reading) data**, use **`getShiftDetailsByDateRange`**, which returns shift-level sales summaries without closing the business day:

```graphql
query GetShiftDetailsByDateRange(
  $startDate: String!
  $endDate: String!
  $deviceId: String
  $uid: String
  $pageNumber: Int!
  $pageSize: Int!
) {
  getShiftDetailsByDateRange(
    startDate: $startDate
    endDate: $endDate
    deviceId: $deviceId
    uid: $uid
    pageNumber: $pageNumber
    pageSize: $pageSize
  ) {
    totalPages
    pageNumber
    shiftDetails {
      id
      deviceId
      cashier
      startShift
      endShift
      openAmount
      actualAmount
      expectedAmount
      differenceAmount
      totalTransactions
      cashPayments
      cashRefunds
      paidInAmount
      paidOutAmount
      salesSummary
      seniorPwdDiscount {
        totalCustomers
        totalSeniors
        totalPwds
      }
      devices {
        device_name
      }
      cashiers {
        username
      }
    }
  }
}
```

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `startDate` | String | Yes | Start date |
| `endDate` | String | Yes | End date (server adds 1 day for range) |
| `deviceId` | String | No | Filter by device |
| `uid` | String | No | User ID override |
| `pageNumber` | Int | Yes | Page number (1-based) |
| `pageSize` | Int | Yes | Records per page |

**Variables example:**

```json
{
  "startDate": "2026-06-15",
  "endDate": "2026-06-15",
  "deviceId": "665a1b2c3d4e5f6789012345",
  "pageNumber": 1,
  "pageSize": 10
}
```

### Sample Response

```json
{
  "data": {
    "getShiftDetailsByDateRange": {
      "totalPages": 1,
      "pageNumber": 1,
      "shiftDetails": [
        {
          "id": "665d4e5f678901234567890",
          "deviceId": "665a1b2c3d4e5f6789012345",
          "cashier": "cashier-firebase-id",
          "startShift": "2026-06-15T08:00:00.000Z",
          "endShift": "2026-06-15T16:00:00.000Z",
          "openAmount": 1000,
          "actualAmount": 5500,
          "expectedAmount": 5400,
          "differenceAmount": 100,
          "totalTransactions": 25,
          "cashPayments": 4500,
          "cashRefunds": 0,
          "paidInAmount": 0,
          "paidOutAmount": 0,
          "salesSummary": "{\"grossSales\":5500,\"netSales\":5090}",
          "seniorPwdDiscount": {
            "totalCustomers": 2,
            "totalSeniors": 1,
            "totalPwds": 1
          },
          "devices": {
            "device_name": "Main Counter"
          },
          "cashiers": {
            "username": "juan.delacruz"
          }
        }
      ]
    }
  }
}
```

If you need a dedicated `getXreadingByDateRange` endpoint with the same shape as Z Reading, that would need to be added to the API separately.

---

## Quick Reference

| Operation | Type | Auth Required |
|-----------|------|---------------|
| `login` | Mutation | No |
| `getDevices` | Query | Yes |
| `getDeviceById` | Query | Yes |
| `allitems` / `item` / `getItemById` | Query | Yes |
| `createItem` | Mutation | Yes |
| `updateItem` | Mutation | Yes |
| `deleteManyItem` | Mutation | Yes |
| `getTransactionsByDateRange` | Query | Yes |
| `getZreadingByDateRange` | Query | Yes |
| `getZReading` | Query | Yes |
| `getShiftDetailsByDateRange` | Query | Yes (X Reading alternative) |

---

## Error Handling

GraphQL errors appear in the top-level `errors` array:

```json
{
  "errors": [
    {
      "message": "User not authenticated",
      "path": ["getDevices"]
    }
  ],
  "data": {
    "getDevices": null
  }
}
```

Common causes:

- Missing or expired Bearer token
- Invalid date format
- Item not found on update
- Empty item name on create

---

## Date Format Notes

- Use ISO 8601 strings: `"2026-06-15"` or `"2026-06-15T00:00:00.000Z"`
- `getTransactionsByDateRange`: `endDate` is inclusive through end of day (`23:59:59.999`)
- `getShiftDetailsByDateRange`: `endDate` range extends to start of the next calendar day
