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

### Phase 3: Production Migration

#### 3.1 Update Auto Scaling Group (if applicable)
```bash
# Update Auto Scaling Group to use new launch template
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name "production-asg" \
    --launch-template LaunchTemplateName="al2023-migration-template",Version='$Latest'
```

#### 3.2 Rolling Instance Replacement
```bash
# Start instance refresh for gradual migration
aws autoscaling start-instance-refresh \
    --auto-scaling-group-name "production-asg" \
    --preferences '{
        "InstanceWarmup": 300,
        "MinHealthyPercentage": 90,
        "CheckpointPercentages": [20, 50, 100],
        "CheckpointDelay": 600
    }'
```

## Post-Migration Validation

### System Validation Checklist
- [ ] Verify OS version and kernel
- [ ] Confirm all services are running
- [ ] Test application functionality
- [ ] Validate log aggregation
- [ ] Check monitoring and alerting
- [ ] Verify backup processes
- [ ] Test security controls

### Automated Validation Script
```bash
#!/bin/bash
# post-migration-validation.sh

echo "=== AL2023 Migration Validation ==="

# Check OS version
echo "OS Version:"
cat /etc/os-release | grep PRETTY_NAME

# Check running services
echo -e "\nCritical Services Status:"
for service in sshd amazon-ssm-agent; do
    systemctl is-active $service
done

# Check IMDS v2 enforcement
echo -e "\nIMDS Configuration:"
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" 2>/dev/null)
if [ -n "$TOKEN" ]; then
    echo "IMDSv2: ENABLED"
else
    echo "IMDSv2: DISABLED/ERROR"
fi

# Application health check (customize as needed)
echo -e "\nApplication Health:"
# Add your application-specific health checks here

echo "=== Validation Complete ==="
```

## Rollback Procedures

### Emergency Rollback Steps

1. **Immediate Rollback**
```bash
# Revert Auto Scaling Group to previous launch template
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name "production-asg" \
    --launch-template LaunchTemplateName="original-al2-template",Version='$Latest'
```

2. **Cancel Instance Refresh**
```bash
# Cancel ongoing instance refresh
aws autoscaling cancel-instance-refresh \
    --auto-scaling-group-name "production-asg"
```

3. **Manual Instance Replacement**
```bash
# Terminate problematic instances (will be replaced by ASG)
aws ec2 terminate-instances --instance-ids i-problematic-instance
```

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: User Data Script Failures
**Symptoms**: Instance launches but applications don't start
**Solution**: 
- Check `/var/log/cloud-init-output.log`
- Verify script syntax and permissions
- Test user data script on test instance

#### Issue 2: Package Installation Failures
**Symptoms**: dnf commands fail in user data
**Solution**:
```bash
# Add retry logic to user data
for i in {1..3}; do
    dnf update -y && break
    sleep 30
done
```

#### Issue 3: IMDS Access Issues
**Symptoms**: Applications can't access instance metadata
**Solution**:
- Verify hop limit configuration
- Check security group rules
- Update application to use IMDSv2

#### Issue 4: Service Startup Failures
**Symptoms**: systemd services don't start automatically
**Solution**:
```bash
# Add explicit service management in user data
systemctl enable your-service
systemctl start your-service
systemctl status your-service
```

### Debug Commands
```bash
# Check cloud-init logs
sudo tail -f /var/log/cloud-init-output.log

# Verify instance metadata access
curl -H "X-aws-ec2-metadata-token: $(curl -X PUT http://169.254.169.254/latest/api/token -H 'X-aws-ec2-metadata-token-ttl-seconds: 21600')" http://169.254.169.254/latest/meta-data/

# Check system journal for errors
journalctl -xe

# Validate network connectivity
ping 8.8.8.8
nslookup amazon.com
```

## Best Practices

### Security Considerations
1. **Always enforce IMDSv2** with hop limit set to 2
2. **Use least privilege IAM roles** for instances
3. **Enable detailed monitoring** for better observability
4. **Implement proper tagging strategy** for resource management
5. **Use encrypted EBS volumes** for data at rest protection

### Performance Optimization
1. **Right-size instances** based on AL2023 performance characteristics
2. **Optimize user data scripts** to reduce boot time
3. **Use placement groups** for low-latency applications
4. **Configure appropriate EBS volume types** for workload requirements

### Operational Excellence
1. **Implement comprehensive logging** and monitoring
2. **Use Infrastructure as Code** (CloudFormation/Terraform) for reproducibility
3. **Establish automated testing** pipelines
4. **Document all customizations** and configurations
5. **Regular security updates** and patch management

### Cost Optimization
1. **Review instance types** for better price/performance ratio
2. **Implement auto-scaling policies** to match demand
3. **Use Spot instances** where appropriate
4. **Monitor and optimize EBS usage**

## Additional Resources

- [Amazon Linux 2023 User Guide](https://docs.aws.amazon.com/linux/al2023/ug/)
- [EC2 Launch Templates Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-launch-templates.html)
- [Instance Metadata Service Documentation](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
- [AWS CLI Reference](https://docs.aws.amazon.com/cli/latest/reference/)

## Change Log

| Version | Date | Changes | Author |
|---------|------|---------|---------|
| 1.0 | 2025-09-08 | Initial SOP creation | DevOps Team |

---

**Note**: This SOP should be tested in a non-production environment before implementation. Always ensure proper backups and rollback procedures are in place before beginning migration activities.