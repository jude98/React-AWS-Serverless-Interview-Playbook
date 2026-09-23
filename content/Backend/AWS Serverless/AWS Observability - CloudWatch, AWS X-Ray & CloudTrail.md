# AWS Observability: CloudWatch, AWS X-Ray & CloudTrail

## Key Concepts

- **The Three Pillars of Observability:**

    - **Metrics (What is happening?):** Aggregated numerical measurements over time (CPU, memory, request count, latency).

    - **Logs (Why is it happening?):** Timestamped textual records of discrete application and infrastructure events.

    - **Traces (Where is it happening?):** End-to-end request journeys across distributed network hops and microservices.

- **Amazon CloudWatch:** Telemetry hub covering **Metrics, Logs, and Alarms**; monitors health, performance, and resource utilization of AWS infrastructure and workloads.

- **AWS X-Ray:** Distributed tracing system that collects segment data using an injected `X-Amzn-Trace-Id` header to map service dependencies and pinpoint latency bottlenecks.

- **AWS CloudTrail:** Audit, governance, and compliance tool recording **API calls** made across your AWS account (captures _who_ called _what API_, _when_, from _which IP_).

## Service Comparison: CloudWatch vs. X-Ray vs. CloudTrail

|**Dimension**|**CloudWatch**|**AWS X-Ray**|**AWS CloudTrail**|
|---|---|---|---|
|**Core Question**|_"What is happening to my system?"_|_"Where is the latency or error occurring?"_|_"Who made this API call or state change?"_|
|**Primary Domain**|System performance, log search, threshold alarms.|Distributed request tracking and call waterfall analysis.|Security, compliance, auditing, and change tracking.|
|**Data Produced**|Metrics (RCU, CPU), Log Groups, Alarms.|Traces, Segments, Subsegments, Service Maps.|CloudTrail Event Records (JSON in S3).|
|**Latency/Ingestion**|Real-time to 1-minute aggregation.|Real-time streaming (sampled traces).|Near real-time (delivered to S3 within ~5–15 mins).|
|**Instrumentation**|Built into AWS; custom metrics via SDK/EMF.|Requires X-Ray SDK, AWS Distro for OpenTelemetry (ADOT), or Lambda active tracing.|Enabled by default for management events across all AWS accounts.|

## Common Interview Questions

- How do CloudWatch, AWS X-Ray, and CloudTrail complement each other during an incident investigation?

- What is CloudWatch Embedded Metric Format (EMF), and why is it preferred over `PutMetricData` for high-throughput Lambda?

- How does AWS X-Ray propagate trace context across asynchronous boundaries (e.g., API Gateway $\rightarrow$ SQS $\rightarrow$ Lambda)?

- What is the difference between CloudTrail Management Events and Data Events, and what are the cost implications?

- How do you construct an alert that triggers a self-healing action (e.g., auto-remediation Lambda via CloudWatch Alarms)?

## Strong Answers / Talking Points

### 1. Incident Investigation Workflow (Putting All Three Together)

1. **Detection (CloudWatch Alarms):** A CloudWatch Alarm fires because p99 latency on the checkout API crossed 2,000 ms, or 5xx errors spiked.

2. **Isolation (AWS X-Ray):** You open the X-Ray Service Map; it highlights that the call between the Order Lambda and DynamoDB is red. The trace waterfall reveals that a specific database query is taking 1,800 ms due to an unindexed scan.

3. **Root Cause / Audit (CloudTrail):** You query CloudTrail to see who recently changed the DynamoDB table or IAM configuration. You identify that an engineer deployed a schema change that dropped the Global Secondary Index (GSI) 30 minutes prior.

### 2. High-Throughput Metrics: Embedded Metric Format (EMF) vs. `PutMetricData`

- Calling `cloudwatch.putMetricData()` makes synchronous HTTP calls, adding network latency and billable API calls that throttle under burst traffic.

- **Embedded Metric Format (EMF):** Prints structured JSON directly to standard output (`stdout`). The CloudWatch Logs agent parses the JSON asynchronously in the background and extracts custom metrics automatically with zero latency impact and lower cost.

### 3. AWS X-Ray Context Propagation & Sampling

- **Trace Header:** Upstream services generate and pass `X-Amzn-Trace-Id: Root=1-5759e988-bd862e3...;Parent=53995cbe...;Sampled=1`.

- **Segments vs. Subsegments:**

    - _Segment:_ Represents compute resource boundaries (e.g., the Lambda runtime environment or API Gateway stage).

    - _Subsegment:_ Fine-grained breakdowns within a segment (e.g., downstream HTTP call, DynamoDB read, custom function block).

- **Sampling Rules:** Avoids 100% trace capture costs; default rule traces the first 1 request per second and 5% of additional requests, configurable via X-Ray sampling rules.

### 4. CloudTrail: Management Events vs. Data Events

- **Management Events (Control Plane):** Operations performed on AWS resources themselves (e.g., `CreateBucket`, `AttachRolePolicy`, `TerminateInstances`). First copy is free across all regions.

- **Data Events (Data Plane):** High-volume operations performed _within_ or _on_ a resource (e.g., S3 `GetObject`/`PutObject`, Lambda `Invoke`, DynamoDB `GetItem`). Disabled by default due to high event volumes; charged per 100,000 events.

## Code Snippets / Examples

### 1. Zero-Latency Custom Metrics with CloudWatch EMF (TypeScript)

```typescript
import { metricScope, Unit } from "aws-embedded-metrics";

// Logs structured EMF JSON to stdout; CloudWatch extracts metrics asynchronously
export const handler = metricScope((metrics) => async (event: any) => {
  metrics.setNamespace("ECommerce/Checkout");
  metrics.setDimensions({ Service: "PaymentProcessor", Environment: "Production" });

  const startTime = Date.now();
  try {
    await processPayment(event.body);
    metrics.putMetric("PaymentSuccess", 1, Unit.Count);
  } catch (error) {
    metrics.putMetric("PaymentFailure", 1, Unit.Count);
    throw error;
  } finally {
    metrics.putMetric("ExecutionLatency", Date.now() - startTime, Unit.Milliseconds);
  }
});
```

### 2. AWS SAM Template: Enabling X-Ray Tracing and CloudWatch Alarms

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Observability Stack with X-Ray and Alarms

Resources:
  InstrumentedOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: index.handler
      Runtime: nodejs20.x
      # Enable active X-Ray tracing on Lambda environment
      Tracing: Active
      Policies:
        - AWSXRayDaemonWriteAccess
      Environment:
        Variables:
          AWS_XRAY_CONTEXT_MISSING: LOG_ERROR

  # Metric Alarm for Function Errors
  LambdaErrorAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: "Trigger alarm if 5xx errors exceed 5 in a 1-minute window"
      Namespace: "AWS/Lambda"
      MetricName: "Errors"
      Dimensions:
        - Name: "FunctionName"
          Value: !Ref InstrumentedOrderFunction
      Statistic: "Sum"
      Period: 60
      EvaluationPeriods: 1
      Threshold: 5
      ComparisonOperator: "GreaterThanOrEqualToThreshold"
      AlarmActions:
        - !Ref AlertSNSTopic

  AlertSNSTopic:
    Type: AWS::SNS::Topic
```

## Related Topics

- [[AWS Lambda Core Architecture & Execution Model]]

- [[AWS API Gateway - Architecture, Security & Limitations]]

- [[SQS DLQ Processing - Correlation IDs, Error Context, and Redrive Pipelines]]

- [[AWS Serverless Interview Scenarios - Advanced System Design & Debugging]]

- [[Engineering Execution, Collaboration & Behavioral Scenarios]]

## Tags

#fullstack #interview #aws #observability #cloudwatch #xray #cloudtrail #devops

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
