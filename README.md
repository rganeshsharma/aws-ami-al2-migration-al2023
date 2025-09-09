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
