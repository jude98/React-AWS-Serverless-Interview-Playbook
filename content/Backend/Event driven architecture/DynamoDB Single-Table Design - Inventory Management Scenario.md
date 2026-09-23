# DynamoDB Single-Table Design: Inventory Management Scenario

## Key Concepts

- **Access-Pattern First Design**: In DynamoDB, schema follows access patterns, not entity normalization. All queries, filters, and range parameters must be known before finalizing the keys.

- **Generic Overloaded Keys**: Name keys generically as `PK` and `SK` (and `GSI1PK`, `GSI1SK`) to store heterogeneous entities (Product, Warehouse Inventory, Stock Reservation, Inventory Audit Log) in a single table.

- **Item Collections**: Items sharing the exact same `PK` reside in the same physical partition. Querying by `PK` with `begins_with(SK, ...)` retrieves parent-child relationships (e.g., Product metadata + stock across all warehouses) in a single network round-trip.

- **Atomic Operations for Stock**: Decrements must use `UpdateExpression: "SET stock = stock - :qty"` combined with `ConditionExpression: "stock >= :qty"` to prevent negative inventory without distributed locks.

- **Sparse Global Secondary Indexes (GSIs)**: GSIs only index items containing the index's partition key. This creates efficient out-of-the-box filtered views (e.g., querying low-stock or active reservation holds) without table-wide scans.

## Common Interview Questions

- How do you design a Single-Table DynamoDB schema for multi-warehouse inventory to support both product-centric and warehouse-centric views?

- How do you fulfill 5 distinct APIs using a single table without resorting to table `Scan` operations?

- How does `begins_with` on a composite sort key enable hierarchical range queries (e.g., filtering inventory movements by date range)?

- Why are generic key names (`PK`, `SK`) preferred over semantic attribute names (`productId`, `warehouseId`) in single-table design?

- How do you handle temporary stock reservations (e.g., 10-minute cart hold) using DynamoDB TTL alongside native inventory tracking?

## Strong Answers / Talking Points

### Scenario: The 5 Target Inventory APIs

1. **API 1 — `GET /products/{productId}`**: Fetch product details plus current inventory levels across **all** warehouses in one query.

2. **API 2 — `GET /warehouses/{warehouseId}/inventory`**: Fetch all product inventories stored in a **specific** warehouse.

3. **API 3 — `POST /orders/reserve`**: Atomically decrement stock at a specific warehouse if available; fail if insufficient.

4. **API 4 — `GET /warehouses/{warehouseId}/logs?from={ts}&to={ts}`**: Retrieve stock movement history (IN/OUT/ADJUST) for a warehouse within a timestamp window.

5. **API 5 — `GET /inventory/low-stock`**: Find all items flagged as low stock across the entire system.

### Data Model & Key Architecture

|**Entity**|**PK (Partition Key)**|**SK (Sort Key)**|**GSI1PK**|**GSI1SK**|**Additional Attributes**|
|---|---|---|---|---|---|
|**Product Metadata**|`PROD#<productId>`|`METADATA`|`CATEGORY#<category>`|`PROD#<productId>`|`title`, `sku`, `price`, `reorderThreshold`|
|**Warehouse Stock**|`PROD#<productId>`|`WH#<warehouseId>`|`WH#<warehouseId>`|`STOCK#<stockLevel>`|`availableStock`, `reservedStock`, `location`|
|**Stock Movement Log**|`WH#<warehouseId>`|`LOG#<ISO_Timestamp>#<logId>`|—|—|`productId`, `delta` (`+10`, `-2`), `reason`|
|**Low-Stock Alert** _(Sparse)_|`PROD#<productId>`|`WH#<warehouseId>`|`STATUS#LOW_STOCK`|`PROD#<productId>`|_Only present if stock $\le$ threshold_|

### Mapping the 5 APIs to Key Conditions

#### API 1: Product + All Warehouse Stock

- **Call**: `Query` on Base Table

- **KeyCondition**: `PK = :prodId AND begins_with(SK, :skPrefix)`

- **Values**: `:prodId = "PROD#P100"`, `:skPrefix = ""` (or omit `SK` condition to return `METADATA` + all `WH#` items).

- **Result**: Single read round-trip returns product details and stock across all warehouses.

#### API 2: All Inventory for a Specific Warehouse

- **Call**: `Query` on **GSI-1**

- **KeyCondition**: `GSI1PK = :whId AND begins_with(GSI1SK, "STOCK#")`

- **Values**: `:whId = "WH#W001"`

- **Result**: Direct index query yielding all products and current quantities in that warehouse.

#### API 3: Atomic Stock Reservation

- **Call**: `UpdateItem` on Base Table

- **Key**: `PK = "PROD#P100"`, `SK = "WH#W001"`

- **UpdateExpression**: `SET availableStock = availableStock - :qty, reservedStock = reservedStock + :qty`

- **ConditionExpression**: `availableStock >= :qty`

- **Result**: Atomic, isolated decrement. If concurrency creates a race condition, DynamoDB throws `ConditionalCheckFailedException`.

#### API 4: Warehouse Stock Movement Logs within Time Window

- **Call**: `Query` on Base Table

- **KeyCondition**: `PK = :whId AND SK BETWEEN :start AND :end`

- **Values**: `:whId = "WH#W001"`, `:start = "LOG#2026-09-01T00:00:00Z"`, `:end = "LOG#2026-09-22T23:59:59Z"`

- **Result**: Efficient chronological range scan within the warehouse partition.

#### API 5: Query Low Stock Items Across System

- **Call**: `Query` on **GSI-1** (Sparse Index)

- **KeyCondition**: `GSI1PK = :lowStockStatus`

- **Values**: `:lowStockStatus = "STATUS#LOW_STOCK"`

- **Result**: Highly cost-effective; only records explicitly flagged with `GSI1PK = "STATUS#LOW_STOCK"` appear in the index.

## Code Snippets / Examples

### API 1 Implementation: Fetch Product + Inventories (AWS SDK v3)

```typescript
import { DynamoDBClient, QueryCommand } from "@aws-sdk/client-dynamodb";
import { unmarshall } from "@aws-sdk/util-dynamodb";

const ddb = new DynamoDBClient({ region: "us-east-1" });
const TABLE_NAME = "InventoryTable";

export async function getProductWithAllWarehouseStock(productId: string) {
  const command = new QueryCommand({
    TableName: TABLE_NAME,
    KeyConditionExpression: "PK = :pk",
    ExpressionAttributeValues: {
      ":pk": { S: `PROD#${productId}` },
    },
  });

  const response = await ddb.send(command);
  const items = (response.Items || []).map((item) => unmarshall(item));

  const metadata = items.find((i) => i.SK === "METADATA");
  const warehouseInventory = items.filter((i) => i.SK.startsWith("WH#"));

  return {
    product: metadata,
    warehouses: warehouseInventory,
  };
}
```

### API 3 Implementation: Conditional Atomic Decrement

```typescript
import { DynamoDBClient, UpdateItemCommand } from "@aws-sdk/client-dynamodb";

const ddb = new DynamoDBClient({ region: "us-east-1" });
const TABLE_NAME = "InventoryTable";

export async function reserveWarehouseStock(
  productId: string,
  warehouseId: string,
  quantityToReserve: number
): Promise<boolean> {
  try {
    const command = new UpdateItemCommand({
      TableName: TABLE_NAME,
      Key: {
        PK: { S: `PROD#${productId}` },
        SK: { S: `WH#${warehouseId}` },
      },
      UpdateExpression: `
        SET availableStock = availableStock - :qty,
            reservedStock = reservedStock + :qty,
            updatedAt = :now
      `,
      ConditionExpression: "availableStock >= :qty",
      ExpressionAttributeValues: {
        ":qty": { N: quantityToReserve.toString() },
        ":now": { S: new Date().toISOString() },
      },
    });

    await ddb.send(command);
    return true; // Reservation succeeded
  } catch (err: any) {
    if (err.name === "ConditionalCheckFailedException") {
      // Stock unavailable or insufficient
      return false;
    }
    throw err;
  }
}
```

## Related Topics

- [[Amazon DynamoDB -  Architecture, Data Modeling & Scaling]]

- [[Event-Driven Architecture Scenarios - Flash Sales, High-Scale Ordering & Extreme Inventory Contention]]

- [[Debugging DynamoDB Hot Partitions & Hot Keys]]

- [[Adding a Global Secondary Index (GSI) to a Large DynamoDB Table]]

- [[Scaling DynamoDB Streams - High-Volume Event Processing]]

## Tags

#fullstack #interview #aws #dynamodb #single-table-design #database #nosql #system-design

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
