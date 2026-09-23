# AWS VPC & Networking Scenarios: Subnets, Lambda VPC Integration, Endpoints & Security

## Key Concepts

- Curated question bank covering VPC topology, private vs. public routing, VPC Peering, Transit Gateway, and NAT architectures.
    
      
    
- Evaluates Lambda-in-VPC mechanics, Hyperplane ENI behavior, internet egress, and multi-tier subnet isolation.
    
      
    
- Focuses heavily on private connectivity to AWS services: Gateway Endpoints (S3, DynamoDB) vs. Interface Endpoints (AWS PrivateLink), Endpoint Policies, and S3 Bucket Policies.
    
      
    
- No answers included—structured strictly for mock interview practice, whiteboard diagramming, and flashcard drilling.
    
      
    

## Common Interview Questions

### Lambda in VPC & Subnet Egress Architecture

- When you connect an AWS Lambda function to a VPC, why does it immediately lose direct internet access by default, even if you attach it to a **public subnet**?
    
      
    
- What is the step-by-step routing configuration required for a Lambda function in a private subnet to access both an internal RDS PostgreSQL database and a public third-party REST API?
    
      
    
- What are the differences between deploying a Lambda function across public vs. private vs. isolated (database-only) subnets?
    
      
    
- How did AWS Hyperplane ENI architecture solve the historical Lambda VPC cold-start penalty and private IP exhaustion problem?
    
      
    
- Why does placing a Lambda function across multiple Availability Zones (AZs) require a NAT Gateway per AZ, and what happens to cross-AZ egress costs and fault isolation if you only provision a single NAT Gateway?
    
      
    

### Accessing S3 Across Subnets, VPCs & Endpoints

- A Lambda function runs in a completely isolated private subnet (no NAT Gateway, no Internet Gateway). How can it read and write objects to Amazon S3?
    
      
    
- What is the difference between an **S3 Gateway Endpoint** and an **S3 Interface Endpoint (AWS PrivateLink)** regarding:
    
      
    - Routing mechanism (Route Table prefix list vs. ENI with private IP)
        
          
        
    - Cost (Free vs. Hourly charge + Data processing fees per GB)
        
          
        
    - Extended access (Can on-premises networks via Direct Connect/VPN access a Gateway Endpoint vs. an Interface Endpoint?)
        
          
        
    - Cross-region access (Can a Gateway Endpoint route to an S3 bucket in a different AWS region?)
        
          
        
- How do you configure a Route Table to associate an S3 Gateway Endpoint (`pl-xxxxxxxx`) with specific private subnets while preventing access from public subnets?
    
      
    
- How do you use **VPC Endpoint Policies** combined with **S3 Bucket Policies** (`aws:sourceVpce` vs. `aws:sourceVpc`) to enforce that an S3 bucket can _only_ be accessed from a designated VPC endpoint, blocking all public internet and console traffic?
    
      
    
- If a VPC has two subnets (Subnet A with a route to an S3 Gateway Endpoint and Subnet B with a route to a NAT Gateway), which path does outbound S3 traffic take if both routes exist, and how does the longest prefix match apply?
    
      
    

### VPC Endpoints: Gateway vs. Interface Endpoints (PrivateLink)

- Why are Amazon S3 and Amazon DynamoDB the only two services supported by VPC Gateway Endpoints?
    
      
    
- When should you use an Interface Endpoint for S3 instead of the free Gateway Endpoint?
    
      
    
- How does AWS PrivateLink work under the hood, and how does split-horizon Private DNS resolve public AWS service endpoints (e.g., `sqs.us-east-1.amazonaws.com`) to private VPC interface IPs?
    
      
    
- What happens if your Lambda function's Security Group blocks outbound HTTPS (Port 443) traffic when attempting to call an AWS service through an Interface Endpoint?
    
      
    

### Network Security: Security Groups vs. Network ACLs (NACLs)

- What is the difference between a Security Group and a Network ACL (NACL) regarding:
    
      
    - Layer of operation (Instance/ENI level vs. Subnet boundary level)
        
          
        
    - Statefulness (Stateful tracking vs. Stateless evaluation)
        
          
        
    - Rule evaluation order (All rules evaluated vs. Numbered sequential order)
        
          
        
    - Explicit Deny support (Can Security Groups have DENY rules?)
        
          
        
- Why do ephemeral client ports (ports 1024–65535) cause return-traffic drop issues when writing custom ingress/egress rules on Network ACLs?
    
      
    
- How do you configure Security Group self-referencing rules (ingress from `sg-xxxx` on port 5432) to allow microservices in the same security group to communicate without exposing static IP ranges?
    
      
    

### Routing, Peering, Transit Gateway & Hybrid Connectivity

- You have two VPCs (VPC A with CIDR `10.0.0.0/16` and VPC B with CIDR `10.1.0.0/16`). You establish a VPC Peering Connection. Why are instances in VPC A still unable to ping instances in VPC B? What are the missing routing and security layers?
    
      
    
- What is the "Transitive Routing" limitation of VPC Peering, and why can't VPC A talk to VPC C through VPC B in a peering topology?
    
      
    
- How does **AWS Transit Gateway (TGW)** solve the full-mesh complexity of VPC Peering when interconnecting dozens or hundreds of VPCs across multiple AWS accounts?
    
      
    
- Can you peer two VPCs that have overlapping or identical CIDR blocks (e.g., both use `10.0.0.0/16`)? How do you resolve this using Private NAT Gateway or AWS PrivateLink?
    
      
    

### Debugging, Observability & DNS Resolution

- A service running in a private subnet fails to connect to an external API. How do you troubleshoot the connectivity path using:
    
      
    - Route Tables and default gateway routes (`0.0.0.0/0` $\to$ `nat-xxxx`)
        
          
        
    - Subnet NACL outbound and inbound rules
        
          
        
    - Security Group outbound egress rules
        
          
        
    - NAT Gateway allocation state, Elastic IP, and route targets
        
          
        
    - VPC Flow Logs (identifying `REJECT` vs. `ACCEPT` records on ENIs)
        
          
        
- How does the default VPC DNS Resolver (Amazon Route 53 Resolver / "+2" address, e.g., `10.0.0.2`) work, and what happens if `enableDnsHostnames` or `enableDnsSupport` is disabled on a VPC?
    
      
    
- How do VPC Flow Logs capture rejected traffic, and why don't flow logs display DNS queries handled directly by the `.2` resolver?
    
      
    

## Strong Answers / Talking Points

- _Note: Reference individual topic notes for full technical breakdown and implementation architecture._
    
      
    - See [[AWS Lambda Core Architecture & Execution Model|AWS Lambda Execution Context and Lifecycle]] for Hyperplane ENI behavior and VPC cold starts.
        
          
        
    - See [[AWS Identity and Access Management (IAM) - Identities, Policies, Roles & Best Practices|AWS Cross-Account IAM and S3 Bucket Policies]] for `aws:sourceVpce` and `aws:sourceVpc` condition keys.
        
          
        
    - See [[AWS VPC & Networking Scenarios - Subnets, Lambda VPC Integration, Endpoints & Security|AWS VPC Gateway Endpoints vs Interface Endpoints Architecture]] for cost vs. hybrid connectivity comparisons.
        
          
        
    - See [[AWS VPC & Networking Scenarios - Subnets, Lambda VPC Integration, Endpoints & Security|Network Security: Security Groups, NACLs, and VPC Flow Logs Analysis]] for ephemeral port mechanics.
        
          
        
    - See [[AWS VPC & Networking Scenarios - Subnets, Lambda VPC Integration, Endpoints & Security|AWS Transit Gateway vs VPC Peering at Enterprise Scale]] for transitive routing topologies.
        
          
        

## Code Snippets / Examples

### Securing S3 Access via VPC Gateway Endpoint Policy & S3 Bucket Policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccessOnlyFromSpecificVpcEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::production-secure-data-bucket",
        "arn:aws:s3:::production-secure-data-bucket/*"
      ],
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0123456789abcdef0"
        }
      }
    }
  ]
}
```

### AWS SAM: Lambda Function Attached to Multi-AZ Private Subnets

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: Lambda function configured with Multi-AZ VPC attachment and private routing.

Resources:
  # Lambda Execution Security Group
  LambdaVpcSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: "Security group for Lambda VPC access"
      VpcId: !Ref TargetVpcId
      SecurityGroupEgress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0 # Allows HTTPS egress to NAT Gateway or VPC Endpoints
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          CidrIp: 10.0.0.0/16 # Internal database access within VPC

  # Lambda Function
  VpcBoundFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: src/
      Handler: app.handler
      Runtime: nodejs20.x
      Timeout: 15
      MemorySize: 512
      VpcConfig:
        SecurityGroupIds:
          - !Ref LambdaVpcSecurityGroup
        SubnetIds:
          # Multi-AZ placement across private subnets (routes point to NAT Gateway or VPC Endpoints)
          - !Ref PrivateSubnetAZ1
          - !Ref PrivateSubnetAZ2
      Policies:
        - AWSLambdaVPCAccessExecutionRole # Required IAM managed policy for ENI provisioning
```

## Related Topics


- [[AWS Lambda Core Architecture & Execution Model]]

- [[Network Protocols. Transport, Security & Application Layers]]

- [[TCP vs. UDP]]

- [[AWS Identity and Access Management (IAM) - Identities, Policies, Roles & Best Practices]]

- [[AWS Serverless Interview Scenarios - Advanced System Design & Debugging]]


## Tags

#fullstack #interview #aws #vpc #networking #lambda #s3 #security #system-design

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups