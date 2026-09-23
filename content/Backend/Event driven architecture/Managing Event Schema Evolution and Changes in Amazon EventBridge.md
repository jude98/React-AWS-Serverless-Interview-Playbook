# Managing Event Schema Evolution and Changes in Amazon EventBridge

## Key Concepts

- **Event Bus Decoupling Paradox**: EventBridge decouples producers from consumers at runtime, but tight semantic coupling remains at the **event schema/contract** level.

- **Tolerant Reader Pattern**: Consumers should read only the fields they care about and ignore unrecognized fields, allowing producers to add non-breaking fields freely.

- **Non-Breaking vs. Breaking Changes**:

    - _Additive/Non-Breaking_: Adding optional fields, adding new event types (`detail-type`).

    - _Breaking_: Renaming fields, changing data types, removing fields, altering nullability, changing semantic meaning of an existing field.

- **EventBridge Schema Registry & Discovery**: Automatically detects event structures from the bus, infers OpenAPI 3.0 or JSONSchema specifications, and generates strongly typed client code bindings (TypeScript, Java, Python).

- **Dual-Publishing & Versioned `detail-type`**: Breaking changes should be introduced via explicit semantic versioning in `detail-type` (e.g., `OrderPlaced.v2`) or metadata payloads (`schemaVersion: "2.0"`), running dual publishing until legacy consumers migrate.

- **Contract Testing & CI/CD Guardrails**: Use schema validation in the publishing pipeline (e.g., JSON Schema validation in code/interceptors or CI linting) to prevent breaking mutations from reaching production buses.

## Common Interview Questions

- How do you handle schema evolution across dozens of independent microservices consuming the same EventBridge bus?

- What constitutes a breaking change in an asynchronous event-driven system, and how does EventBridge rule pattern matching behave when schemas change?

- How does the EventBridge Schema Registry work, and what is the role of Schema Discovery in staging vs. production environments?

- What strategies would you use to deprecate an existing event version without breaking legacy downstream subscribers?

- How do EventBridge Input Transformers interact with schema changes, and why can they become an operational liability?

- How do you implement schema governance and prevent unauthorized event schema drift across cross-account buses?

## Strong Answers / Talking Points

### 1. The Core Strategy: Forward & Backward Compatibility

- **Tolerant Reader Strategy**:

    - Consumers must never perform strict schema validation that throws errors on unexpected properties (e.g., in Zod/TypeScript, avoid `.strict()` unless parsing internal models; allow passthrough of unknown attributes).

    - All new fields added to existing events **must be optional**.

- **Rule Pattern Matching Resilience**:

    - EventBridge Rules match patterns strictly against fields in the event envelope and `detail`.

    - If a rule filters on `{ "detail": { "status": ["PAID"] } }`, renaming `status` to `orderStatus` breaks rule evaluation immediately, silently causing the consumer to receive zero events.

### 2. Versioning Strategies for Breaking Changes

- **Pattern A: Versioned Event Types (`detail-type`) — Recommended**:

    - Keep the original event untouched: `detail-type: "OrderPlaced.v1"`.

    - Introduce the breaking schema as `detail-type: "OrderPlaced.v2"`.

    - Producers **dual-publish** both `v1` and `v2` during the transition window.

    - Consumers migrate to `v2` at their own pace. Once metrics show zero invocations or after a sunset deadline, deprecate and remove `v1`.

- **Pattern B: Envelope Metadata Versioning**:

    - Place a `schemaVersion` attribute inside the payload (`"schemaVersion": "2.0.0"`).

    - Consumers branch processing logic internally based on `schemaVersion`, or EventBridge rules filter specifically on the version.

- **Pattern C: Event Translation Layer (Adapter / Step Functions)**:

    - If producers cannot dual-publish, route the new event format through an AWS Lambda or Step Function adapter that translates `v2` into `v1` and publishes both onto the bus.

### 3. EventBridge Schema Registry & Discovery

- **Schema Discovery**: When enabled on an event bus, EventBridge samples ingested events, infers their JSON Schema/OpenAPI definition, and uploads them to the Schema Registry.

    > [!warning] Production Governance
    >
    > Enable Schema Discovery in **Development and Staging environments**, but disable it in high-volume production environments to avoid unnecessary AWS Discovery costs and unintended schema version churn.
    >
    >

- **Code Bindings**: Generate typed SDK packages directly from the registry using the AWS SAM CLI or CloudFormation, providing end-to-end compile-time safety for event producers and consumers.

### 4. Input Transformers: The Silent Failure Risk

- Input Transformers restructure an event's JSON before passing it to targets (e.g., mapping `$.detail.userId` to `<cognitoId>`).

- If a producer renames `userId` to `customerId`, the Input Transformer will output empty strings or fail validation, causing target delivery failures without alerting the producer.

- Prefer delivering the raw event envelope to the consumer when possible, or ensure input transformers are covered by end-to-end integration tests.

### Schema Change Decision Matrix

|**Change Type**|**Example**|**Resolution Strategy**|**Dual-Publish Needed?**|
|---|---|---|---|
|**Additive (Non-Breaking)**|Adding `deliveryNotes`|Keep existing `detail-type`; mark field optional in schema.|No|
|**Subtractive (Breaking)**|Removing `customerPhone`|Mint new version (e.g., `OrderPlaced.v2`); keep `v1` intact.|Yes|
|**Type Mutation (Breaking)**|`orderTotal: 100` $\to$ `orderTotal: { amount: 100, currency: "USD" }`|Mint new version (`OrderPlaced.v2`).|Yes|
|**Semantic Change (Breaking)**|`status: "PENDING"` meaning changes from "unpaid" to "awaiting fulfillment"|Treat as a new event or increment version to prevent silent logic errors.|Yes|

## Code Snippets / Examples

### Producer: Dual-Publishing Strategy during Schema Migration

```typescript
import { EventBridgeClient, PutEventsCommand, PutEventsRequestEntry } from "@aws-sdk/client-eventbridge";

const ebClient = new EventBridgeClient({ region: "us-east-1" });
const EVENT_BUS_NAME = "ecommerce-events";

interface OrderPlacedV1 {
  orderId: string;
  amount: number; // Single currency assumption (legacy)
}

interface OrderPlacedV2 {
  orderId: string;
  money: {
    amount: number;
    currency: string;
  };
  metadata: {
    schemaVersion: "2.0";
  };
}

export async function publishOrderPlaced(order: { id: string; total: number; currency: string }): Promise<void> {
  const v1Payload: OrderPlacedV1 = {
    orderId: order.id,
    amount: order.total,
  };

  const v2Payload: OrderPlacedV2 = {
    orderId: order.id,
    money: {
      amount: order.total,
      currency: order.currency,
    },
    metadata: {
      schemaVersion: "2.0",
    },
  };

  const entries: PutEventsRequestEntry[] = [
    // Legacy v1 event for existing consumers
    {
      EventBusName: EVENT_BUS_NAME,
      Source: "ecommerce.orders",
      DetailType: "OrderPlaced.v1",
      Detail: JSON.stringify(v1Payload),
    },
    // Modern v2 event for upgraded consumers
    {
      EventBusName: EVENT_BUS_NAME,
      Source: "ecommerce.orders",
      DetailType: "OrderPlaced.v2",
      Detail: JSON.stringify(v2Payload),
    },
  ];

  await ebClient.send(new PutEventsCommand({ Entries: entries }));
}
```

### SAM Template: EventBridge Schema Registry & Version-Filtered Rules

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Schema Registry and EventBridge Rule routing by versioned detail-type.

Resources:
  CustomEventBus:
    Type: AWS::Events::EventBus
    Properties:
      Name: ecommerce-events

  # Schema definition for OrderPlaced.v2
  OrderPlacedV2Schema:
    Type: AWS::EventSchemas::Schema
    Properties:
      RegistryName: "aws.events"
      SchemaName: "ecommerce.orders@OrderPlacedV2"
      Type: "OpenApi3"
      Content: |
        openapi: "3.0.0"
        info:
          version: "2.0.0"
          title: "OrderPlaced"
        paths: {}
        components:
          schemas:
            OrderPlaced:
              type: object
              required: [orderId, money]
              properties:
                orderId:
                  type: string
                money:
                  type: object
                  required: [amount, currency]
                  properties:
                    amount:
                      type: number
                    currency:
                      type: string

  # Consumer Rule listening strictly to v2 events
  ModernOrderConsumerRule:
    Type: AWS::Events::Rule
    Properties:
      EventBusName: !Ref CustomEventBus
      EventPattern:
        source:
          - "ecommerce.orders"
        detail-type:
          - "OrderPlaced.v2" # Guarantees consumer never receives incompatible v1 payloads
      Targets:
        - Arn: !GetAtt V2ConsumerQueue.Arn
          Id: "V2QueueTarget"

  V2ConsumerQueue:
    Type: AWS::SQS::Queue
```

### Consumer: Tolerant Reader Implementation (Node.js with Zod)

```typescript
import { z } from "zod";

// Non-strict schema: accepts unknown future fields without error
const OrderPlacedV2Schema = z.object({
  orderId: z.string(),
  money: z.object({
    amount: z.number(),
    currency: z.string().default("USD"),
  }),
  // Optional field with safe fallback
  discountCode: z.string().optional(),
}).passthrough(); // Crucial: passthrough() ignores unknown additive fields

export const handler = async (event: { detail: unknown }) => {
  const result = OrderPlacedV2Schema.safeParse(event.detail);

  if (!result.success) {
    console.error("Critical Schema Mismatch:", result.error);
    // Route to DLQ or alert; this indicates an unhandled breaking change
    throw new Error("Invalid schema received");
  }

  const validOrder = result.data;
  console.log(`Processing Order ${validOrder.orderId}`);
};
```

## Related Topics

- [[Amazon EventBridge - Event Buses, Pipes, Patterns & Schemas]]

- [[Handling Event Clogging and Backpressure in Amazon EventBridge]]

- [[AWS SNS vs. Amazon EventBridge - Architecture, Differences, and Combined Patterns]]

- [[AWS Serverless & Event-Driven Architecture (EDA)]]

- [[Clean Architecture, Directory Structure & DTOs]]

## Tags

#fullstack #interview #aws #eventbridge #serverless #system-design #event-driven-architecture

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
