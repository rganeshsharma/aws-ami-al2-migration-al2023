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
