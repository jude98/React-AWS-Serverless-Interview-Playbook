# Amazon S3: Architecture, Storage Classes, Security & Large Uploads

## Key Concepts

- **Amazon S3 Definition:** Highly scalable, distributed object storage service offering $99.999999999\%$ (11 9s) of data durability by redundantly replicating data across multiple Availability Zones.

- **Buckets vs. Objects:**

    - **Bucket:** Globally unique top-level container deployed in a specific AWS Region; controls regional boundaries, billing, access logs, and lifecycle rules.

    - **Object:** The fundamental entity stored consisting of data (the payload), a unique key (the full name/path), metadata (name-value pairs), version ID, and an access control policy.

- **Consistency Model:** Strong read-after-write consistency for `PUT` and `DELETE` requests of objects across all AWS regions without delay.

- **Upload Size Limits:**

    - Single `PUT` operation maximum: **5 GB**.

    - Single object maximum size: **5 TB**.

    - Recommended threshold for **Multipart Upload**: Any file exceeding **100 MB** (mandatory for objects $> 5\text{ GB}$).

## Common Interview Questions

- How does S3 achieve 11 9s of durability compared to availability?

- Walk me through S3 storage classes and how you optimize lifecycle transition policies for cost.

- How do you implement secure client-side uploads without exposing AWS IAM credentials?

- What security controls protect S3 buckets from public data leaks (Block Public Access, Bucket Policies, KMS)?

- How does the Multipart Upload API work, and how do you prevent orphaned, billed part fragments?

- What is S3 Transfer Acceleration and when does it improve upload performance?

## Strong Answers / Talking Points

### 1. Storage Classes & Lifecycle Transitions

|**Storage Class**|**Designed For**|**Retrieval Latency**|**Min Duration**|**Cost Profile**|
|---|---|---|---|---|
|**S3 Standard**|Active, frequently accessed data|Milliseconds|None|High storage, free retrievals|
|**S3 Intelligent-Tiering**|Unknown or unpredictable patterns|Milliseconds|None|Small monitoring fee per object; auto-shifts tiers|
|**S3 Standard-IA**|Long-lived, infrequently accessed|Milliseconds|30 days|Lower storage cost, retrieval fee per GB|
|**S3 One Zone-IA**|Recreatable, non-critical data|Milliseconds|30 days|Single AZ (loses multi-AZ resilience); 20% cheaper than Standard-IA|
|**S3 Glacier Flexible**|Archive, backup data|Minutes to hours|90 days|Very low storage, retrieval fee|
|**S3 Glacier Deep Archive**|Long-term digital preservation|12 to 48 hours|180 days|Lowest storage cost in AWS|

### 2. S3 Security Controls & Defense-in-Depth

> [!NOTE]
>
> S3 security requires layers of controls across identity, transit, at-rest storage, and networking:


- **Block Public Access (BPA):** Account-level and bucket-level master switch that overrides all bucket policies and ACLs, preventing accidental public leaks.

- **Bucket Policies vs. IAM Policies:** IAM policies define _who_ (identities) can do what; Bucket policies define _who can access this bucket_ (resource-based), allowing cross-account delegation and enforcing HTTPS.

- **Enforcing Encryption in Transit:** Use bucket policy conditions denying `aws:SecureTransport: "false"` to block non-HTTPS traffic.

- **Encryption at Rest:**

    - **SSE-S3 (Default):** Keys handled and rotated by Amazon S3 (AES-256).

    - **SSE-KMS:** Keys managed via AWS KMS; provides audit trails in CloudTrail and key rotation, but subject to KMS request rate limits/costs.

    - **SSE-C:** Customer provides and manages keys; AWS does not store encryption keys.

- **Origin Access Control (OAC):** Secures S3 buckets behind CloudFront distributions so objects cannot be accessed directly via S3 URLs.

### 3. Large File Uploads: Multipart Upload & Pre-signed URLs

When uploading files larger than 100 MB, single HTTP `PUT` requests become prone to failure. Network drops require retransmitting the entire file.

1. **Initiate:** Client calls backend API $\rightarrow$ Lambda calls `CreateMultipartUpload` on S3 $\rightarrow$ S3 returns an `UploadId`.

2. **Generate Pre-signed URLs:** Backend generates a pre-signed URL for each separate chunk (part size: 5 MB to 5 GB, up to 10,000 parts).

3. **Parallel Upload:** Client streams individual parts directly to S3 concurrently over HTTPS (optionally via **S3 Transfer Acceleration** utilizing AWS edge locations).

4. **Complete:** Client collects each part's number and returned `ETag`, and sends them to backend $\rightarrow$ Lambda calls `CompleteMultipartUpload` to assemble the final object.

5. **Cost Trap Prevention:** Configure an S3 Lifecycle Rule with `AbortIncompleteMultipartUpload` (e.g., 7 days) to automatically purge incomplete uploads and avoid recurring storage charges for abandoned parts.

## Code Snippets / Examples

### 1. Generating Pre-signed S3 URL for Secure Client Upload (Node.js / SDK v3)

```typescript
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const s3Client = new S3Client({ region: process.env.AWS_REGION });

export async function getUploadPresignedUrl(fileName: string, fileType: string): Promise<string> {
  const command = new PutObjectCommand({
    Bucket: process.env.BUCKET_NAME,
    Key: `uploads/${Date.now()}-${fileName}`,
    ContentType: fileType,
  });

  // Client has 15 minutes to complete upload directly to S3
  return await getSignedUrl(s3Client, command, { expiresIn: 900 });
}
```

### 2. S3 Bucket Policy: Enforce HTTPS & Require SSE-KMS Encryption

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceTLSRequestsOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-secure-bucket",
        "arn:aws:s3:::my-secure-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::my-secure-bucket/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "aws:kms"
        }
      }
    }
  ]
}
```

## Related Topics

- [[AWS Identity and Access Management (IAM) - Identities, Policies, Roles & Best Practices]]

- [[System Design Scenarios - Payment Workflows, Webhooks, Idempotency & Large S3 Payloads]]

- [[Caching Architecture, Eviction Policies & Invalidation Pitfalls]]

- [[Client-Side Browser Storage. Mechanisms, Architecture & Security]]

- [[AWS VPC & Networking Scenarios - Subnets, Lambda VPC Integration, Endpoints & Security]]

## Tags

#fullstack #interview #aws #s3 #cloud-storage #security #system-design

## Revision Checklist

- [ ] Can explain in 60 seconds

- [ ] Can explain trade-offs

- [ ] Can give a real project example

- [ ] Can answer common follow-ups
