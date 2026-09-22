
## Key Concepts

- **`$LATEST` Antipattern**: By default, Lambda updates point directly to `$LATEST`. Invoking `$LATEST` in production causes immediate, all-at-once code switches with zero rollback buffer or warmup phase.
    
      
    
- **Immutable Versions**: Publishing a version takes an immutable snapshot of code, runtime, and configuration. Versions cannot be modified once minted.
    
      
    
- **Lambda Aliases & Traffic Shifting**: An alias is a mutable pointer to one or two immutable versions. Weighted aliases split traffic across two versions (e.g., $90\%$ on v1, $10\%$ on v2), forming the mechanical backbone for canary and linear rollouts.
    
      
    
- **SAM `AutoPublishAlias`**: Automatically creates and increments an immutable version whenever code (`CodeUri`) or configuration changes, and repoints the specified alias (`live`) to that version.
    
      
    
- **AWS CodeDeploy Integration**: Coordinates progressive traffic shifting via deployment preferences (`Canary10Percent5Minutes`, `Linear10PercentEvery1Minute`).
    
      
    
- **Automated Rollbacks via CloudWatch Alarms**: If alarms breach (e.g., $5\text{xx}$ errors, elevated latency, custom synthetic test failures) during the traffic shift, CodeDeploy halts progression and instantly points $100\%$ of traffic back to the prior stable version.
    
      
    
- **Hooks (Pre/Post-Traffic Tests)**: CodeDeploy invokes a validation Lambda before routing traffic (`PreTrafficHook`) and after routing finishes (`PostTrafficHook`) to execute integration smoke tests before finalizing the deploy.
    
      
    

## Common Interview Questions

- What is the difference between a Lambda Version and a Lambda Alias, and why should production callers never invoke `$LATEST`?
    
      
    
- How does AWS SAM's `AutoPublishAlias` property work under the hood with CloudFormation?
    
      
    
- How does AWS CodeDeploy execute a Canary or Linear deployment for Lambda without changing the invoking endpoint URL?
    
      
    
- What are `PreTraffic` and `PostTraffic` hooks, and how do they prevent a broken deployment from reaching users?
    
      
    
- How do CloudFormation rollbacks differ from CodeDeploy rollbacks during a failed canary release?
    
      
    
- How do Event Source Mappings (SQS/DynamoDB) interact with Lambda Aliases during blue/green or canary rollouts?
    
      
    

## Strong Answers / Talking Points

### 1. The Deployment Foundation: Versions, Aliases, and `AutoPublishAlias`

- **Why Versions Matter**: If you deploy code directly to `$LATEST` while a high-throughput system is running, in-flight requests may execute against mismatched code versions mid-request, causing transient failures.
    
      
    
- **Role of Aliases**: External callers (API Gateway, EventBridge, SQS ESM) should always target an **Alias ARN** (e.g., `:live`), never a bare Function ARN. This decouples the client configuration from underlying version deployments.
    
      
    
- **What SAM Does Behind the Scenes**:
    
      
    - Detects code or configuration drift.
        
          
        
    - Generates a new `AWS::Lambda::Version` resource with a deterministic SHA-256 hash.
        
          
        
    - Updates the `AWS::Lambda::Alias` resource to point to the newly published version.
        
          
        

### 2. Progressive Traffic Shifting (CodeDeploy Strategies)

Instead of switching $100\%$ of traffic instantaneously, CodeDeploy supports three primary modes:

  

|**Deployment Type**|**Pattern Name**|**Progression Mechanic**|**Best Use Case**|
|---|---|---|---|
|**Canary**|`Canary10Percent10Minutes`|Routes $10\%$ immediately; holds for 10 minutes; shifts remaining $90\%$ to new version|Standard production releases with moderate traffic|
|**Linear**|`Linear10PercentEvery1Minute`|Shifts $10\%$ every minute ($10\% \to 20\% \to \dots \to 100\%$) over 10 minutes|High-throughput APIs where gradual load testing is required|
|**All-At-Once**|`AllAtOnce`|Immediate $100\%$ cutover to the new version|Development, staging, or emergency hotfixes|

### 3. Safety Guardrails: Hooks & Rollback Triggers

- **Pre-Traffic Hook**:
    
      
    - Before any public traffic hits the new version ($0\%$ shifted), CodeDeploy invokes a test Lambda.
        
          
        
    - The test function invokes the new version directly using its **Version ARN** or a test-specific alias.
        
          
        
    - If integration tests fail, the hook calls `codedeploy.putLifecycleEventHookExecutionStatus({ status: 'Failed' })`, aborting the deployment before a single customer request hits the new code.
        
          
        
- **Continuous Metric Monitoring (Alarms)**:
    
      
    - During the canary window, CodeDeploy continuously queries specified CloudWatch Alarms (e.g., Lambda Errors, API Gateway 5xx, p99 Latency).
        
          
        
    - If an alarm enters the `ALARM` state at minute 4 of a 10-minute canary, CodeDeploy triggers an **instant automated rollback** ($100\%$ routed back to the old version).
        
          
        
- **Post-Traffic Hook**:
    
      
    - Runs once $100\%$ of traffic is shifted to the new version to perform final health audits before marking the CloudFormation stack update as complete.
        
          
        

### 4. CloudFormation vs. CodeDeploy Rollback Mechanics

- If a deployment fails during the canary phase, **CodeDeploy rolls back the alias immediately** (traffic shifts back in seconds).
    
      
    
- CodeDeploy then reports a failure to CloudFormation.
    
      
    
- CloudFormation marks the stack update as `UPDATE_ROLLBACK_IN_PROGRESS` and cleanly reverts template resources to their prior state, leaving zero orphan infrastructure.
    
      
    

## Code Snippets / Examples

### AWS SAM Template: Safe Canary Deployment with Alarms & Hooks

YAML

```
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Production Lambda canary deployment with CloudWatch alarm rollbacks and validation hooks.

Resources:
  # 1. Main Production Function
  OrderServiceFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/orders/
      Handler: index.handler
      Runtime: nodejs20.x
      Timeout: 15
      MemorySize: 512
      AutoPublishAlias: live # SAM automatically creates and increments versions
      DeploymentPreference:
        Type: Canary10Percent5Minutes # 10% for 5 mins, then 100%
        Alarms:
          - !Ref DeploymentErrorAlarm # Triggers instant rollback if breached
        Hooks:
          PreTraffic: !Ref PreTrafficHookFunction # Runs smoke tests before shifting traffic

  # 2. CloudWatch Metric Alarm for Rollback Trigger
  DeploymentErrorAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: "Rollback canary if 5xx errors or Lambda failures spike"
      Namespace: AWS/Lambda
      MetricName: Errors
      Dimensions:
        - Name: FunctionName
          Value: !Ref OrderServiceFunction
        - Name: ExecutedVersion
          Value: !GetAtt OrderServiceFunction.Version.Version
      Statistic: Sum
      Period: 60
      EvaluationPeriods: 1
      Threshold: 2
      ComparisonOperator: GreaterThanOrEqualToThreshold

  # 3. Pre-Traffic Validation Hook Function
  PreTrafficHookFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/hooks/
      Handler: preTrafficHook.handler
      Runtime: nodejs20.x
      Timeout: 30
      Policies:
        - Version: '2012-10-17'
          Statement:
            - Effect: Allow
              Action:
                - codedeploy:PutLifecycleEventHookExecutionStatus
                - lambda:InvokeFunction
              Resource: '*'
      Environment:
        Variables:
          CURRENT_VERSION_ARN: !Ref OrderServiceFunction.Version
```

### Pre-Traffic Hook Implementation (Node.js SDK v3)

TypeScript

```
import { CodeDeployClient, PutLifecycleEventHookExecutionStatusCommand } from "@aws-sdk/client-codedeploy";
import { LambdaClient, InvokeCommand } from "@aws-sdk/client-lambda";

const codedeploy = new CodeDeployClient({ region: "us-east-1" });
const lambda = new LambdaClient({ region: "us-east-1" });

interface CodeDeployHookEvent {
  DeploymentId: string;
  LifecycleEventHookExecutionId: string;
}

export const handler = async (event: CodeDeployHookEvent): Promise<void> => {
  const deploymentId = event.DeploymentId;
  const lifecycleEventHookExecutionId = event.LifecycleEventHookExecutionId;
  const targetVersionArn = process.env.CURRENT_VERSION_ARN!;

  let validationPassed = false;

  try {
    console.log(`Running PreTraffic validation against: ${targetVersionArn}`);

    // 1. Invoke new version directly with a diagnostic synthetic payload
    const testPayload = JSON.stringify({ isHealthCheck: true });
    const response = await lambda.send(new InvokeCommand({
      FunctionName: targetVersionArn,
      Payload: Buffer.from(testPayload),
    }));

    const result = JSON.parse(Buffer.from(response.Payload!).toString());

    // 2. Validate expected smoke test assertions
    if (result.statusCode === 200 && result.databaseConnected === true) {
      validationPassed = true;
    } else {
      console.error("Synthetic health check returned non-200 or unhealthy state:", result);
    }

  } catch (err) {
    console.error("PreTraffic test execution threw an unhandled error:", err);
  }

  // 3. Notify CodeDeploy whether to proceed with canary shifting or halt
  await codedeploy.send(new PutLifecycleEventHookExecutionStatusCommand({
    deploymentId,
    lifecycleEventHookExecutionId,
    status: validationPassed ? "Succeeded" : "Failed",
  }));
};
```

## Related Topics

- [[AWS CloudFormation: Custom Resources and Drift Detection]]
    
      
    
- [[CI/CD Pipeline Design with AWS CodePipeline and CodeDeploy]]
    
      
    
- [[Synthetic Monitoring and Canary Testing in Distributed Systems]]
    
      
    
- [[Blue-Green vs Rolling vs Canary Deployment Strategies]]
    
      
    
- [[AWS Lambda Cold Starts: Provisioned Concurrency vs SnapStart]]
    
      
    

## Tags

#fullstack #interview #aws #lambda #cloudformation #codedeploy #devops #serverless

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups