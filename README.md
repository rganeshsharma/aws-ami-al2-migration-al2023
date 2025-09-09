# Standard Operating Procedure: AWS AMI Migration from Amazon Linux 2 to Amazon Linux 2023

## Overview

This document outlines the standard operating procedure for migrating AWS EC2 instances from Amazon Linux 2 (AL2) to Amazon Linux 2023 (AL2023) using launch templates with enhanced security configurations.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Migration Strategy](#migration-strategy)
- [Launch Template Configuration](#launch-template-configuration)
- [Step-by-Step Migration Process](#step-by-step-migration-process)
- [Post-Migration Validation](#post-migration-validation)
- [Rollback Procedures](#rollback-procedures)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)


## Prerequisites

### Technical Requirements
- AWS CLI configured with appropriate permissions
- Access to EC2, VPC, and IAM services
- Understanding of current AL2 instance configurations
- Backup strategy in place

### Required Permissions
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateLaunchTemplate",
                "ec2:ModifyLaunchTemplate",
                "ec2:DescribeLaunchTemplates",
                "ec2:DescribeImages",
                "ec2:DescribeInstances",
                "ec2:RunInstances",
                "ec2:TerminateInstances",
                "ec2:CreateTags",
                "iam:PassRole"
            ],
            "Resource": "*"
        }
    ]
}
```

### Pre-Migration Checklist
- [ ] Document current AL2 instance configurations
- [ ] Identify application dependencies and compatibility with AL2023
- [ ] Prepare custom user data scripts
- [ ] Verify security group configurations
- [ ] Ensure IAM roles and policies are compatible
- [ ] Schedule maintenance window
- [ ] Notify stakeholders

## Migration Strategy

### Key Differences Between AL2 and AL2023

| Component | Amazon Linux 2 | Amazon Linux 2023 |
|-----------|-----------------|-------------------|
| Kernel | 4.14/5.x | 6.1+ |
| Package Manager | yum | dnf |
| Init System | systemd | systemd |
| Python Default | Python 2.7/3.7 | Python 3.9+ |
| Container Runtime | Docker | Podman (default) |
| SELinux | Disabled by default | Enabled by default |

### Migration Approaches
1. **Blue-Green Deployment** (Recommended)
2. **Rolling Update**
3. **In-Place Migration** (Not recommended for production)

## Launch Template Configuration
### Using AWS Console 
AWS Console > EC2> Launch templates > Create Launch template 

Launch templates name : al2023_launch_template
Click Browser AMI's > al2023 for x86_64 AL 2023 kernel-6.x AMI
If you want you can add custom Instance type, Keypair, Network settings, Storage for the launch template

Advanced details > Metadata response hop limit = 2

Provide User data consisting your cluster details 

Refer for example user data: https://repost.aws/knowledge-center/custom-user-eks-2023 

```config
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="BOUNDARY"

--BOUNDARY
Content-Type: application/node.eks.aws

---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    apiServerEndpoint: API_SERVER_ENDPOINT
    certificateAuthority: CERTIFICATE
    cidr: SERVICE_IPv4_RANGE
    name: CLUSTER_NAME
  kubelet:
    config:
      maxPods: 17 
    flags:
    - "--node-labels=key=value" 

--BOUNDARY
Content-Type: text/x-shellscript;

#!/bin/bash
  set -o xtrace
  yum install htop -y

--BOUNDARY--
```

Use this new template that has hop limit set to 2 and add choose this launch template during new AL2023 node group creation.


## Launch Template Configuration

### Core Configuration Parameters

```bash
# Get latest AL2023 AMI ID
AL2023_AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=al2023-ami-*-x86_64" \
              "Name=state,Values=available" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)
```

### Instance Metadata Service (IMDS) Configuration
- **Hop Limit**: Set to 2 for enhanced security
- **IMDS Version**: Enforce IMDSv2
- **Response Hop Limit**: Controls metadata access through NAT/proxy

## Step-by-Step Migration Process

### Phase 1: Preparation

#### 1.1 Inventory Current Configuration
```bash
# Document existing instance details
aws ec2 describe-instances \
    --instance-ids i-1234567890abcdef0 \
    --query 'Reservations[].Instances[].[InstanceId,InstanceType,ImageId,SecurityGroups,IamInstanceProfile]' \
    --output table
```

#### 1.2 Create Launch Template

```bash
# Create launch template with AL2023 and security configurations
aws ec2 create-launch-template \
    --launch-template-name "al2023-migration-template" \
    --launch-template-data '{
        "ImageId": "'$AL2023_AMI_ID'",
        "InstanceType": "t3.medium",
        "SecurityGroupIds": ["sg-xxxxxxxxxx"],
        "IamInstanceProfile": {
            "Name": "EC2-Instance-Profile"
        },
        "MetadataOptions": {
            "HttpTokens": "required",
            "HttpPutResponseHopLimit": 2,
            "HttpEndpoint": "enabled",
            "InstanceMetadataTags": "enabled"
        },
        "UserData": "'$(base64 -w 0 userdata.sh)'",
        "TagSpecifications": [
            {
                "ResourceType": "instance",
                "Tags": [
                    {
                        "Key": "Name",
                        "Value": "AL2023-Migrated-Instance"
                    },
                    {
                        "Key": "Environment",
                        "Value": "Production"
                    },
                    {
                        "Key": "Migration",
                        "Value": "AL2-to-AL2023"
                    }
                ]
            }
        ]
    }'
```

#### 1.3 Sample User Data Script (userdata.sh)

```bash
#!/bin/bash

# AL2023 User Data Script for Migration
# Set system timezone
timedatectl set-timezone UTC

# Update system packages
dnf update -y

# Install essential packages
dnf install -y \
    htop \
    curl \
    wget \
    git \
    unzip \
    python3-pip \
    awscli

# Configure CloudWatch agent (if required)
if [ -f "/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json" ]; then
    /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
        -a fetch-config \
        -m ec2 \
        -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
        -s
fi

# Application-specific configurations
# Replace with your application setup commands

# Enable and start services
systemctl enable amazon-ssm-agent
systemctl start amazon-ssm-agent

# Custom application deployment
# Add your application-specific commands here

# Signal completion
/opt/aws/bin/cfn-signal -e $? --stack ${AWS::StackName} --resource AutoScalingGroup --region ${AWS::Region}

# Log completion
echo "AL2023 instance initialization completed at $(date)" >> /var/log/migration.log
```
### Phase 2: Testing and Validation

#### 2.1 Launch Test Instance
```bash
# Launch instance from template for testing
aws ec2 run-instances \
    --launch-template LaunchTemplateName="al2023-migration-template",Version='$Latest' \
    --min-count 1 \
    --max-count 1 \
    --subnet-id subnet-xxxxxxxxxx
```

#### 2.2 Validate Instance Configuration
```bash
# Connect to instance and verify configurations
ssh -i your-key.pem ec2-user@instance-ip

# Check OS version
cat /etc/os-release

# Verify IMDS configuration
curl -s http://169.254.169.254/latest/meta-data/instance-id

# Check hop limit setting
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/)/network-card/

# Verify user data execution
cat /var/log/cloud-init-output.log
```
