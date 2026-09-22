## Key Concepts

- **Core Purpose:** The central control plane for identity and access management across AWS resources, operating on the principle of **explicit denial** (default deny unless explicitly allowed).
    
      
    
- **Users vs. Groups vs. Roles:**
    
      
    - **IAM User:** A persistent identity (person or application) with permanent long-term credentials (password, access key/secret key pair).
        
          
        
    - **IAM Group:** A collection of IAM users used solely for bulk permission attachment; cannot be identified as a `Principal` in a policy.
        
          
        
    - **IAM Role:** An identity with specific permissions that is **not** tied to a single entity; it is assumed dynamically by authorized principals to acquire **short-lived temporary credentials** via AWS STS (Security Token Service).
        
          
        
- **Policy Evaluation Logic:**
    
      
    
    $$\text{Explicit Deny} > \text{Explicit Allow} > \text{Default Deny}$$
    
    An explicit deny anywhere in the evaluation chain always overrides any allow.
    
      
    
- **Policy Types:**
    
      
    - **Identity-based Policies:** Attached to Users, Groups, or Roles.
        
          
        
    - **Resource-based Policies:** Attached directly to resources (e.g., S3 Bucket Policies, KMS Key Policies, SQS Access Policies).
        
          
        
    - **Permissions Boundaries & SCPs:** Maximum permission guardrails that restrict the effective permissions an identity can hold.
        
          
        

## Common Interview Questions

- What is the difference between an IAM User and an IAM Role, and why are Roles preferred for workloads?
    
      
    
- How does AWS evaluate IAM policies when both Identity-based and Resource-based policies exist?
    
      
    
- What are the required elements of an IAM Policy statement, and how do Conditions work?
    
      
    
- What is the difference between a Trust Policy and a Permissions Policy on an IAM Role?
    
      
    
- How does AWS STS issue temporary credentials, and how do they mitigate the risk of credential leakage?
    
      
    
- What are Service Control Policies (SCPs) and Permission Boundaries, and how do they enforce security guardrails?
    
      
    

## Strong Answers / Talking Points

### 1. The Anatomy of an IAM Policy (PARC Framework)

An IAM JSON policy document consists of statements defining access permissions:

  

- **`Principal`:** The "who" (required in resource-based policies and trust policies; omitted in identity-based policies because the identity is implicit).
    
      
    
- **`Action`:** The specific API operations allowed or denied (e.g., `s3:GetObject`, `dynamodb:PutItem`).
    
      
    
- **`Resource`:** The target ARN (Amazon Resource Name) the actions apply to.
    
      
    
- **`Effect`:** Either `"Allow"` or `"Deny"`.
    
      
    
- **`Condition`:** Contextual criteria required for the policy to take effect (e.g., MFA required, IP CIDR blocks, secure transport, tags).
    
      
    

### 2. IAM Roles: Trust Policy vs. Permissions Policy

> [!NOTE]
> 
> Every IAM Role requires two distinct policies to function:
> 
>   

1. **Trust Policy (AssumeRole Policy):** Defines **who can assume** the role (e.g., the Lambda service `lambda.amazonaws.com`, an EC2 instance, or another AWS Account ID).
    
      
    
2. **Permissions Policy:** Defines **what actions** the role can perform once assumed (e.g., read from DynamoDB, write logs to CloudWatch).
    
      
    

### 3. Cross-Account Access via STS `AssumeRole`

- Workload in Account A needs to access an S3 bucket in Account B.
    
      
    
- Instead of sharing long-lived keys, Account B creates a Role with a Trust Policy listing Account A's ID as an allowed principal.
    
      
    
- The service in Account A calls `sts:AssumeRole` $\rightarrow$ AWS STS returns temporary access credentials (Access Key ID, Secret Access Key, Session Token) valid for 15 minutes to 12 hours $\rightarrow$ Service signs requests to Account B's bucket.
    
      
    

### 4. IAM Best Practices for Production Systems

- **Eliminate Long-Term Credentials:** Never issue permanent IAM user access keys for applications; use IAM Roles for EC2/ECS/Lambda, and IAM Identity Center (formerly AWS SSO) or OIDC federation for human access.
    
      
    
- **Enforce Least Privilege:** Grant only the minimum permissions necessary for specific resources (avoid `Action: "*"` and `Resource: "*"`).
    
      
    
- **Use Permission Boundaries:** Guardrail self-service IAM creation so developer roles cannot escalate their own privileges.
    
      
    
- **Require Multi-Factor Authentication (MFA):** Enforce MFA via condition keys (`aws:MultiFactorAuthPresent`) for administrative actions or console logins.
    
      
    
- **Centralize Access with AWS Organizations & SCPs:** Use Service Control Policies at the organization level to enforce compliance invariant rules across accounts (e.g., disable unapproved regions, block turning off CloudTrail).
    
      
    
- **Regular Audit & Credential Hygiene:** Use IAM Access Analyzer to detect resources shared outside the organization, and remove inactive credentials older than 90 days.
    
      
    

## Code Snippets / Examples

### 1. IAM Role with Trust Policy and Least-Privilege Permissions Policy


```JSON
{
  "TrustPolicy": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "lambda.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  },
  "PermissionsPolicy": {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowDynamoDBWriteSpecificTable",
        "Effect": "Allow",
        "Action": [
          "dynamodb:PutItem",
          "dynamodb:UpdateItem"
        ],
        "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
      },
      {
        "Sid": "AllowCloudWatchLogsCreation",
        "Effect": "Allow",
        "Action": [
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ],
        "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/lambda/order-service:*"
      }
    ]
  }
}
```

### 2. Condition-Based Policy: Enforce MFA and Source IP


```JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceMFAAndCorporateIPForAdmin",
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        },
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

### 3. Programmatic Role Assumption using AWS SDK v3 (TypeScript)

```TypeScript
import { STSClient, AssumeRoleCommand } from "@aws-sdk/client-sts";
import { S3Client, ListObjectsV2Command } from "@aws-sdk/client-s3";

const stsClient = new STSClient({});

export async function fetchCrossAccountBucket() {
  // Assume the role in the target account
  const assumeRoleCmd = new AssumeRoleCommand({
    RoleArn: "arn:aws:iam::987654321098:role/CrossAccountDataReadRole",
    RoleSessionName: "data-sync-session",
    DurationSeconds: 900,
  });

  const { Credentials } = await stsClient.send(assumeRoleCmd);

  // Initialize a new client with the ephemeral STS credentials
  const crossAccountS3 = new S3Client({
    credentials: {
      accessKeyId: Credentials!.AccessKeyId!,
      secretAccessKey: Credentials!.SecretAccessKey!,
      sessionToken: Credentials!.SessionToken!,
    },
  });

  return await crossAccountS3.send(
    new ListObjectsV2Command({ Bucket: "target-account-analytics-bucket" })
  );
}
```

## Related Topics

- [[AWS-Security-and-Compliance]]
    
      
    
- [[Amazon-S3-Architecture-and-Security]]
    
      
    
- [[AWS-API-Gateway-and-Authentication-Strategies]]
    
      
    
- [[AWS-Organizations-and-Multi-Account-Architecture]]
    
      
    
- [[OAuth2-OIDC-and-Federated-Identities]]
    
      
    

## Tags

#fullstack #interview #aws #iam #cloud-security #devops #system-design

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups