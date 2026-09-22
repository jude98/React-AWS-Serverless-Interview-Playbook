
## Key Concepts

- **Provisioned Capacity**: You explicitly set fixed Read Capacity Units (RCUs) and Write Capacity Units (WCUs). You are billed per provisioned hour, regardless of whether you consume that throughput.
    
      
    
- **Application Auto Scaling (Target Tracking)**: Dynamically adjusts provisioned RCUs/WCUs within defined minimum and maximum bounds based on target utilization (typically $70\%$).
    
      
    
- **On-Demand Capacity (`PAY_PER_REQUEST`)**: Serverless, pay-per-request pricing with zero capacity planning. Accommodates instantaneous traffic spikes up to $2\times$ the previous peak throughput without manual pre-warming.
    
      
    
- **Auto Scaling Latency Lag**: DynamoDB Auto Scaling is **reactive, not instantaneous**. It relies on CloudWatch metric alarms (evaluation windows take $2\text{–}15\text{ minutes}$), meaning sudden sharp spikes will experience throttling before scaling takes effect.
    
      
    
- **Cost Differential**: Unit-for-unit, On-Demand requests cost approximately **$5\times\text{ to }7\times$ more** than a fully utilized Provisioned unit.
    
      
    
- **Reserved Capacity Arbitrage**: For predictable steady-state workloads, purchasing DynamoDB Reserved Capacity on Provisioned mode yields up to a $50\%\text{–}75\%$ cost reduction over On-Demand rates.
    
      
    

## Common Interview Questions

- How does DynamoDB Application Auto Scaling decide when to scale up vs. scale down, and why does it struggle with sudden traffic spikes?
    
      
    
- What are the cost and architectural trade-offs between On-Demand mode and Provisioned Mode with Auto Scaling?
    
      
    
- What is the $2\times$ previous peak rule in DynamoDB On-Demand, and how does partition pre-warming work?
    
      
    
- Can you switch between Provisioned and On-Demand capacity modes on a live production table without downtime?
    
      
    
- How does DynamoDB Burst Capacity work, and why shouldn't you rely on it as an architectural crutch for under-provisioning?
    
      
    
- When would you combine Provisioned capacity with AWS Lambda Provisioned Concurrency?
    
      
    

## Strong Answers / Talking Points

### 1. Mechanics of Provisioned Mode with Auto Scaling

- **How It Works**:
    
      
    1. You define `MinCapacity`, `MaxCapacity`, and a `TargetUtilization` percentage (e.g., $70\%$).
        
          
        
    2. CloudWatch publishes `ConsumedReadCapacityUnits` / `ConsumedWriteCapacityUnits` metrics every minute.
        
          
        
    3. AWS Application Auto Scaling creates CloudWatch alarms:
        
          
        - **Scale-Out Alarm**: Triggers when consumed capacity breaches the target for two consecutive 1-minute periods.
            
              
            
        - **Scale-In Alarm**: Triggers when utilization stays below the target for 15 consecutive minutes (conservative cooldown to prevent flapping).
            
              
            
- **The Core Flaw (Spike Vulnerability)**:
    
      
    - Because CloudWatch alarms require at least 2–3 minutes to detect a breach and another 1–2 minutes for the Auto Scaling API to increase table limits, a traffic spike that ramps in seconds will exhaust available burst capacity and **throttle requests** before Auto Scaling reacts.
        
          
        

### 2. Mechanics of On-Demand Mode (`PAY_PER_REQUEST`)

- **How It Works**: DynamoDB manages partition splitting and throughput allocation transparently. There are no RCUs/WCUs to configure.
    
      
    
- **The $2\times$ Peak Ceiling**:
    
      
    - An On-Demand table instantly scales up to **twice its previous highest peak throughput**.
        
          
        
    - _Example_: If your table previously peaked at $10,000\text{ WCU}$, it can instantly absorb a spike up to $20,000\text{ WCU}$ without throttling. If traffic spikes to $30,000\text{ WCU}$ within seconds, you will experience throttling while DynamoDB splits and provisions physical partitions.
        
          
        
- **Pre-Warming On-Demand**: If you anticipate an unprecedented spike (e.g., Super Bowl campaign), you can temporarily switch the table to Provisioned mode with the expected high capacity to force partition allocation, and immediately switch back to On-Demand.
    
      
    

### 3. When to Use Which Mode

```
                          Traffic Pattern Analysis
                                     │
         ┌───────────────────────────┴───────────────────────────┐
         ▼                                                       ▼
Unpredictable / Spiky / Low Utilization            Predictable / Steady / High Volume
   • New application launch                           • Established SaaS core platform
   • Webhook ingestion bursts                         • High baseline 24/7 read/write traffic
   • Infrequently used internal tools                 • Scheduled batch ingestion jobs
         │                                                       │
         ▼                                                       ▼
   USE ON-DEMAND                                          USE PROVISIONED
 (`PAY_PER_REQUEST`)                                  (+ Auto Scaling & Reserved Capacity)
```

- **Choose On-Demand When**:
    
      
    - Traffic is spiky, erratic, or unknown (e.g., viral consumer apps, webhooks).
        
          
        
    - The table experiences long periods of near-zero idle traffic (saves money because idle hours cost $0).
        
          
        
    - Operational simplicity is prioritized over raw compute cost optimization.
        
          
        
- **Choose Provisioned with Auto Scaling When**:
    
      
    - Traffic is predictable, gradual, or has a stable baseline (e.g., daily business cycles).
        
          
        
    - Total volume is consistently high: running steady $10,000+\text{ WCU/RCU}$ on On-Demand is cost-prohibitive.
        
          
        
    - You can leverage **Reserved Capacity** (1-year or 3-year commitments) to slash infrastructure costs by up to $70\%$.
        
          
        

### Direct Comparison Matrix

|**Dimension**|**On-Demand Mode**|**Provisioned + Auto Scaling**|
|---|---|---|
|**Pricing Model**|Pure pay-per-request (per million reads/writes)|Billed per provisioned RCU/WCU per hour|
|**Response to Instant Spikes**|Instant scale up to $2\times$ historical peak|Delayed by $2\text{–}15\text{ minutes}$ (causes throttling)|
|**Cost at High, Steady Traffic**|Very expensive ($5\text{–}7\times$ premium)|Highly cost-effective (especially with Reserved Capacity)|
|**Cost at Zero Traffic**|**$0.00**|Billed for minimum baseline provisioned capacity|
|**Capacity Management**|Zero configuration|Set min/max bounds, target utilization, cooldown periods|
|**Switching Limit**|Can switch table mode up to **4 times per day**|Can switch table mode up to **4 times per day**|

## Code Snippets / Examples

### AWS SAM: Provisioned Table with Application Auto Scaling Policies

YAML

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: DynamoDB table configured with Provisioned mode and Application Auto Scaling.

Resources:
  OrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: HighScaleOrders
      BillingMode: PROVISIONED
      AttributeDefinitions:
        - AttributeName: orderId
          AttributeType: S
      KeySchema:
        - AttributeName: orderId
          KeyType: HASH
      ProvisionedThroughput:
        ReadCapacityUnits: 100
        WriteCapacityUnits: 100

  # 1. Scalable Target for Write Capacity
  WriteCapacityScalableTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      MaxCapacity: 5000 # Scaling ceiling
      MinCapacity: 100  # Baseline floor
      ResourceId: !Sub "table/${OrdersTable}"
      RoleARN: !Sub "arn:aws:iam::${AWS::AccountId}:role/aws-service-role/application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_DynamoDBTable"
      ScalableDimension: dynamodb:table:WriteCapacityUnits
      ServiceNamespace: dynamodb

  # 2. Target Tracking Scaling Policy (Maintains 70% utilization)
  WriteScalingPolicy:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: DynamicWriteScaling
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref WriteCapacityScalableTarget
      TargetTrackingScalingPolicyConfiguration:
        TargetValue: 70.0
        ScaleInCooldown: 900  # 15 mins to prevent aggressive down-scaling
        ScaleOutCooldown: 60  # 1 min to react quickly to spikes
        PredefinedMetricSpecification:
          PredefinedMetricType: DynamoDBWriteCapacityUtilization
```

### AWS SAM: On-Demand Table (Zero Configuration)

YAML

```
Resources:
  ServerlessOrdersTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: ServerlessOrders
      BillingMode: PAY_PER_REQUEST # On-Demand handles scaling automatically
      AttributeDefinitions:
        - AttributeName: orderId
          AttributeType: S
      KeySchema:
        - AttributeName: orderId
          KeyType: HASH
```

## Related Topics

- [[Debugging DynamoDB Hot Partitions & Hot Keys]]
    
      
    
- [[DynamoDB Parallel Scan: Segments, Throughput, and Distributed Processing]]
    
      
    
- [[Bulk Updating 1 Million Rows in DynamoDB: Architectural Approaches and Trade-offs]]
    
      
    
- [[Adding a Global Secondary Index (GSI) to a Large DynamoDB Table]]
    
      
    
- [[AWS Lambda Cold Starts: Provisioned Concurrency vs SnapStart]]
    
      
    

## Tags

#fullstack #interview #aws #dynamodb #system-design #cost-optimization #database

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups