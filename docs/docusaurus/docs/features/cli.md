---
title: 'CLI Mode'
description: 'Use Infra Operator without Kubernetes - manage AWS infrastructure via command line'
sidebar_position: 2
---

## Overview

CLI mode allows you to use Infra Operator to create, manage, and delete AWS resources **without depending on a Kubernetes cluster**. Ideal for:

- Local development
- LocalStack testing
- Automation scripts
- Environments without Kubernetes available
- Rapid infrastructure prototyping

Resource state is stored locally in JSON files in the `~/.infra-operator/state/` directory.

## Installation

### Build from Source

**Command:**

```bash
# Clone the repository
git clone https://github.com/andrebassi/aws-infra-operator.git
cd aws-infra-operator

# Build the binary
CGO_ENABLED=0 go build -o infra-operator cmd/main.go

# Optional: move to PATH
sudo mv infra-operator /usr/local/bin/
```

### Download from Release (coming soon)

**Command:**

```bash
# Linux/macOS
curl -LO https://github.com/andrebassi/infra-operator-aws/releases/latest/download/infra-operator-$(uname -s)-$(uname -m)
chmod +x infra-operator-*
sudo mv infra-operator-* /usr/local/bin/infra-operator
```

## AWS Configuration

The CLI uses the standard AWS SDK credential chain:

### Option 1: Environment Variables

**Command:**

```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_REGION="us-east-1"

# Optional: for LocalStack
export AWS_ENDPOINT_URL="http://localhost:4566"
```

### Option 2: AWS Profile

**Command:**

```bash
# ~/.aws/credentials
[my-profile]
aws_access_key_id = AKIAXXXXXXXXXXXXXXXX
aws_secret_access_key = xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

# ~/.aws/config
[profile my-profile]
region = us-east-1

# Use the profile
export AWS_PROFILE=my-profile
```

### Option 3: IAM Role (EC2/ECS/Lambda)

In AWS environments, the CLI automatically uses the IAM Role associated with the instance/task/function.

### Option 4: AWSProvider in Manifest

**Example:**

```yaml
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: AWSProvider
metadata:
  name: localstack
spec:
  region: us-east-1
  endpoint: http://localhost:4566  # For LocalStack
```

## Commands

### help - Help

**Command:**

```bash
infra-operator help
```

**Output:**
```
Infra Operator AWS - CLI Mode

Usage:
  infra-operator apply  -f <file.yaml>  [flags]    Apply resources from YAML
  infra-operator plan   -f <file.yaml>  [flags]    Show execution plan
  infra-operator delete -f <file.yaml>  [flags]    Delete resources
  infra-operator get    [kind]                     List resources in state

Flags:
  -f, --file string        Path to manifest YAML file (can be repeated)
  --region string          AWS region (default: us-east-1 or env AWS_REGION)
  --endpoint string        AWS endpoint URL (for LocalStack)
  --state-dir string       State directory (default: ~/.infra-operator/state)
  --dry-run                Show what would be done without making changes
  -v, --verbose            Verbose output
```

### plan - Execution Plan

Shows what will be created/updated/deleted without making changes:

```bash
infra-operator plan -f manifest.yaml
```

**Output:**
```
=== Execution Plan ===

  + ComputeStack/dev-network (CREATE)
  + ComputeStack/staging-network (CREATE)
  ~ VPC/my-vpc (UPDATE)
  = Subnet/public-1 (NO CHANGE)

Plan: 2 to create, 1 to update, 1 unchanged
```

### apply - Apply Resources

Creates or updates AWS resources according to the manifest:

```bash
infra-operator apply -f manifest.yaml
```

**Output:**
```
=== Applying resources from 1 file(s) ===

  Creating VPC...
VPC: vpc-0f749cc7f23b5ba5b
  Creating Internet Gateway...
IGW: igw-0f05ef98fc069682b
  Creating Public Subnet...
Subnet: subnet-039966749cb323b64
  Creating Route Table...
Route Table: rtb-0bc8f81a94445bee6
  Creating Security Group...
Security Group: sg-05fcf88aeea9a2dcc
  Creating Bastion Instance...
Instance: i-01fc8bc746072f649
  Created ComputeStack/dev-network
vpcId: vpc-0f749cc7f23b5ba5b
internetGatewayId: igw-0f05ef98fc069682b
publicSubnetId: subnet-039966749cb323b64
```

### get - List Resources

Lists resources managed by the CLI:

```bash
# List all resources
infra-operator get

# List by type
infra-operator get ComputeStack
infra-operator get VPC
infra-operator get EC2Instance
```

**Output:**
```
KIND                 NAME                           AWS ID                                   STATUS
----------------------------------------------------------------------------------------------------
ComputeStack         dev-network                    vpc-0f749cc7f23b5ba5b                    Ready
ComputeStack         staging-network                vpc-0a123bc4d56e78901                    Ready
```

### delete - Delete Resources

Removes AWS resources and cleans up local state:

```bash
infra-operator delete -f manifest.yaml
```

**Output:**
```
=== Deleting resources from 1 file(s) ===

  Terminating bastion instance i-01fc8bc746072f649...
  Deleting security group sg-05fcf88aeea9a2dcc...
  Deleting route table rtb-0bc8f81a94445bee6...
  Deleting subnet subnet-039966749cb323b64...
  Detaching IGW igw-0f05ef98fc069682b...
  Deleting IGW igw-0f05ef98fc069682b...
  Deleting VPC vpc-0f749cc7f23b5ba5b...
  Deleted ComputeStack/dev-network
```

## Practical Examples

### Example 1: Create Complete VPC with Bastion

**File: `bastion-stack.yaml`**:

```yaml
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: AWSProvider
metadata:
  name: aws-prod
spec:
  region: us-east-1
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: ComputeStack
metadata:
  name: bastion-stack
spec:
  vpcCIDR: "10.0.0.0/16"
  providerRef:
    name: aws-prod
  bastionInstance:
    enabled: true
    instanceType: t3.micro
    publicIP: true
    userData: |
      #!/bin/bash
      yum update -y
      yum install -y htop vim
```

**Execute:**

```bash
# View plan
infra-operator plan -f bastion-stack.yaml

# Apply
infra-operator apply -f bastion-stack.yaml

# Verify
infra-operator get ComputeStack
```

### Example 2: Multiple Environments in Same File

**File: `multi-env.yaml`**:

```yaml
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: AWSProvider
metadata:
  name: aws-default
spec:
  region: us-east-1
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: ComputeStack
metadata:
  name: dev-network
spec:
  vpcCIDR: "10.10.0.0/16"
  providerRef:
    name: aws-default
  bastionInstance:
    enabled: true
    instanceType: t3.micro
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: ComputeStack
metadata:
  name: staging-network
spec:
  vpcCIDR: "10.20.0.0/16"
  providerRef:
    name: aws-default
  bastionInstance:
    enabled: true
    instanceType: t3.small
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: ComputeStack
metadata:
  name: prod-network
spec:
  vpcCIDR: "10.30.0.0/16"
  providerRef:
    name: aws-default
  bastionInstance:
    enabled: true
    instanceType: t3.medium
```

### Example 3: Using with LocalStack

**Start LocalStack:**

```bash
docker run -d \
  --name localstack \
  -p 4566:4566 \
  -e SERVICES=ec2,vpc,iam,sts \
  localstack/localstack
```

**File: `localstack-test.yaml`**:

```yaml
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: AWSProvider
metadata:
  name: localstack
spec:
  region: us-east-1
  endpoint: http://localhost:4566
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: ComputeStack
metadata:
  name: local-test
spec:
  vpcCIDR: "10.100.0.0/16"
  providerRef:
    name: localstack
  bastionInstance:
    enabled: true
    instanceType: t3.micro
```

**Execute:**

```bash
# Configure fake credentials for LocalStack
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test

# Apply
infra-operator apply -f localstack-test.yaml --endpoint http://localhost:4566

# Verify
infra-operator get
```

### Example 4: CloudInit with Complex Scripts

**Example:**

```yaml
---
apiVersion: aws-infra-operator.runner.codes/v1alpha1
kind: ComputeStack
metadata:
  name: cloudinit-docker
spec:
  vpcCIDR: "10.50.0.0/16"
  providerRef:
    name: aws-prod
  bastionInstance:
    enabled: true
    instanceType: t3.medium
    publicIP: true
    userData: |
      #!/bin/bash
      set -e

      # Update system
      yum update -y

      # Install Docker
      amazon-linux-extras install docker -y
      systemctl start docker
      systemctl enable docker
      usermod -aG docker ec2-user

      # Install Docker Compose
      curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
      chmod +x /usr/local/bin/docker-compose

      # Create marker file
      echo "Setup complete: $(date)" > /home/ec2-user/setup-complete.txt
```

## Global Flags

| Flag | Description | Example |
|------|-------------|---------|
| `-f, --file` | YAML file (can repeat) | `-f vpc.yaml -f ec2.yaml` |
| `--region` | AWS region | `--region us-west-2` |
| `--endpoint` | Custom AWS endpoint | `--endpoint http://localhost:4566` |
| `--state-dir` | State directory | `--state-dir /opt/infra/state` |
| `--dry-run` | Simulation mode | `--dry-run` |
| `-v, --verbose` | Verbose output | `-v` |

## State Structure

The CLI stores state in `~/.infra-operator/state/`:

```
~/.infra-operator/
└── state/
├── ComputeStack/
│   └── default/
│       ├── dev-network.json
│       └── staging-network.json
├── VPC/
│   └── default/
│       └── my-vpc.json
└── EC2Instance/
        └── default/
            └── bastion.json
```

**State file format:**

```json
{
  "apiVersion": "aws-infra-operator.runner.codes/v1alpha1",
  "kind": "ComputeStack",
  "name": "dev-network",
  "spec": {
"vpcCIDR": "10.10.0.0/16",
"providerRef": { "name": "aws-prod" },
"bastionInstance": { "enabled": true, "instanceType": "t3.micro" }
  },
  "status": {
"phase": "Ready",
"ready": true,
"vpc": { "id": "vpc-0f749cc7f23b5ba5b", "state": "available" }
  },
  "awsResources": {
"vpcId": "vpc-0f749cc7f23b5ba5b",
"internetGatewayId": "igw-0f05ef98fc069682b",
"publicSubnetId": "subnet-039966749cb323b64",
"routeTableId": "rtb-0bc8f81a94445bee6",
"bastionSecurityGroupId": "sg-05fcf88aeea9a2dcc",
"bastionInstanceId": "i-01fc8bc746072f649"
  },
  "createdAt": "2025-11-26T12:06:01.995731-03:00",
  "updatedAt": "2025-11-26T12:06:01.995731-03:00"
}
```

## Resource Order

The CLI applies resources respecting dependencies:

1. AWSProvider
2. VPC
3. InternetGateway
4. Subnet
5. ElasticIP
6. NATGateway
7. RouteTable
8. SecurityGroup
9. IAMRole
10. KMSKey
11. SecretsManagerSecret
12. S3Bucket
13. ECRRepository
14. RDSInstance
15. DynamoDBTable
16. ElastiCacheCluster
17. SQSQueue
18. SNSTopic
19. LambdaFunction
20. EC2Instance, EC2KeyPair
21. ALB, NLB
22. EKSCluster, ECSCluster
23. APIGateway
24. Certificate
25. CloudFront
26. Route53HostedZone
27. Route53RecordSet
28. ComputeStack (high-level)

## Troubleshooting

### Error: "no valid credentials found"

**Command:**

```bash
# Verify credentials
aws sts get-caller-identity

# Or export explicitly
export AWS_ACCESS_KEY_ID="..."
export AWS_SECRET_ACCESS_KEY="..."
```

### Error: "resource not found in state"

The resource may have been manually deleted in AWS. Use `--verbose` to see details:

```bash
infra-operator delete -f manifest.yaml -v
```

### Error: "no resources found in files"

Verify the YAML is correct and contains `apiVersion` and `kind`:

```bash
cat manifest.yaml | head -20
```

### Clear Local State

**Command:**

```bash
# Remove state for a type
rm -rf ~/.infra-operator/state/ComputeStack/

# Remove all state
rm -rf ~/.infra-operator/state/
```

## Next Steps

- [ComputeStack](/services/compute/ec2#computestack) - High-level resource for complete infrastructure
- [Drift Detection](/features/drift-detection) - Detect manual changes
- [Prometheus Metrics](/features/prometheus-metrics) - Infrastructure monitoring
