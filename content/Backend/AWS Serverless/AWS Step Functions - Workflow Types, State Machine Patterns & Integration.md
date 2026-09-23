# AWS Step Functions: Workflow Types, State Machine Patterns & Integration

## Key Concepts

- **Core Purpose:** A serverless visual workflow orchestrator using state machines defined in Amazon States Language (ASL - JSON/YAML) to coordinate microservices, distributed transactions (Saga pattern), and data pipelines.

- **Orchestration vs. Choreography:**

    - **Choreography (EventBridge/SNS/SQS):** Services react independently to events; no central controller; best for loosely coupled domain boundaries.

    - **Orchestration (Step Functions):** A centralized coordinator explicitly controls the flow of execution, state transitions, retries, and compensation logic.

- **Workflow Types:**

    - **Standard Workflows:** Long-running (up to 1 year), durable, auditable, exactly-once execution, billed per state transition.

    - **Express Workflows:** High-volume, short-duration (up to 5 minutes), at-least-once execution, billed by execution count and duration (GB-seconds).

- **Task Tokens (`.waitForTaskToken`):** Pauses execution indefinitely until an external process, human approval, or third-party callback calls `SendTaskSuccess` or `SendTaskFailure`.

- **Payload Limit:** Maximum execution input/output payload size is **256 KB** (larger state requires S3 claim-check pointers).

## Common Interview Questions

- What are the architectural and billing differences between Standard and Express Workflows?

- When should you use Event Orchestration (Step Functions) vs. Event Choreography (SNS/SQS/EventBridge)?

- How do you implement the distributed Saga Pattern (Compensating Transactions) using Step Functions?

- How does the `waitForTaskToken` integration pattern work for human-in-the-loop approvals?

- How do you handle error handling, retries with exponential backoff, and fallback paths natively in ASL without custom code?

- What are Map and Parallel states, and how does Distributed Map enable large-scale serverless batch processing?

## Strong Answers / Talking Points

### 1. Standard Workflows vs. Express Workflows

|**Feature**|**Standard Workflows**|**Express Workflows**|
|---|---|---|
|**Max Duration**|Up to **1 year**.|Up to **5 minutes**.|
|**Execution Semantics**|**Exactly-once** state execution.|**At-least-once** execution (requires idempotency).|
|**Execution Rate**|$\approx 2,000\text{ / second}$.|Over **100,000\text{ / second}$**.|
|**Pricing Model**|**$0.000025 per state transition** ($25 per 1M transitions).|**$1.00 per 1M requests** + duration based on memory ($0.00001667/GB-s).|
|**Execution History**|Full visual, step-by-step audit history in console for 90 days.|Shipped to CloudWatch Logs (adds log ingestion cost).|
|**Integration Support**|Supports `.sync` and `.waitForTaskToken`.|Does not support `.waitForTaskToken` (cannot pause for external callbacks).|
|**Best For**|Financial transactions, order fulfillment, long ETL, human approval.|High-frequency API backends, real-time streaming ingestion, high-rate IoT telemetry.|

### 2. Service Integration Patterns

1. **Request-Response (Default):** Step Functions calls an API/service (Lambda, DynamoDB, ECS) and immediately transitions to the next state once the HTTP call completes.

2. **Run a Job (`.sync`):** Calls a service (e.g., AWS Batch, Glue, ECS Task) and polls until the job finishes before moving to the next state.

3. **Wait for Callback (`.waitForTaskToken`):** Passes an auto-generated token to the task payload (e.g., SQS or an email link). The state machine enters a paused state (costing $0 while waiting in Standard workflows) until the token is returned.

### 3. Distributed Saga Pattern with Step Functions

- **Challenge:** In microservices, distributed ACID transactions across multiple databases are anti-patterns (two-phase commit causes locking and fragility).

- **Step Functions Solution (Orchestrated Saga):**

    - Executes a series of local transactions: `AuthorizePayment` $\rightarrow$ `ReserveInventory` $\rightarrow$ `BookDelivery`.

    - Uses `Catch` blocks in Amazon States Language. If `BookDelivery` fails, the state machine routes to compensating states in reverse order: `CancelInventoryReservation` $\rightarrow$ `RefundPayment`.

    - Eliminates the need for custom retry/coordination spaghetti code in application layers.

### 4. Native Parallelism: `Parallel` vs. `Map` vs. `Distributed Map`

- **Parallel State:** Executes fixed, hardcoded concurrent branches simultaneously (e.g., branch A executes fraud check, branch B executes credit check).

- **Inline Map State:** Iterates over a dynamic input array within a single execution (concurrency up to 40).

- **Distributed Map:** High-scale iteration for big data/ETL. Reads millions of records directly from S3 files (CSV, JSON, Parquet) and launches up to **10,000 parallel child workflow executions**.

## Code Snippets / Examples

### 1. Amazon States Language (ASL): Saga Pattern with Retry & Compensating Catch

```json
{
  "Comment": "Order Processing Saga with Compensation",
  "StartAt": "ReserveCredit",
  "States": {
    "ReserveCredit": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:ReserveCredit",
        "Payload.$": "$"
      },
      "ResultPath": "$.creditResult",
      "Retry": [
        {
          "ErrorEquals": ["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "OrderFailed"
        }
      ],
      "Next": "ReserveInventory"
    },
    "ReserveInventory": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:ReserveInventory",
        "Payload.$": "$"
      },
      "ResultPath": "$.inventoryResult",
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "Next": "CompensateCredit"
        }
      ],
      "Next": "OrderSuccess"
    },
    "CompensateCredit": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "arn:aws:lambda:us-east-1:123456789012:function:RefundCredit",
        "Payload.$": "$"
      },
      "Next": "OrderFailed"
    },
    "OrderSuccess": {
      "Type": "Succeed"
    },
    "OrderFailed": {
      "Type": "Fail",
      "Cause": "Order processing failed during execution"
    }
  }
}
```

### 2. Task Token Pattern: Callback Resumption (TypeScript / Node.js)

```typescript
import { SFNClient, SendTaskSuccessCommand, SendTaskFailureCommand } from "@aws-sdk/client-sfn";

const sfnClient = new SFNClient({});

// Called by external webhook, admin dashboard, or queue consumer when human/async task completes
export async function completeTask(taskToken: string, approved: boolean, approverId: string) {
  if (approved) {
    await sfnClient.send(
      new SendTaskSuccessCommand({
        taskToken: taskToken,
        output: JSON.stringify({ approved: true, approver: approverId, timestamp: Date.now() }),
      })
    );
  } else {
    await sfnClient.send(
      new SendTaskFailureCommand({
        taskToken: taskToken,
        error: "ApprovalRejected",
        cause: `Approval declined by user ${approverId}`,
      })
    );
  }
}
```

## Related Topics

- [[Distributed Transactions & Event-Driven Architecture - Sagas, 2PC, Resilience & Messaging Selection]]

- [[Amazon SQS - Queue Types, Internal Mechanics & Limits]]

- [[Amazon EventBridge - Event Buses, Pipes, Patterns & Schemas]]

- [[Amazon SNS (Simple Notification Service) - Architecture, Fanout & Delivery]]

- [[AWS Lambda Event Invocations - Synchronous, Asynchronous & Event Source Mappings]]

## Tags

#fullstack #interview #aws #step-functions #saga-pattern #orchestration #distributed-systems

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups