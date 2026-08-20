---
title: "AWS Highly Available Web Application — ALB, Auto Scaling & Self-Healing"
description: "A hands-on AWS project implementing a multi-AZ web application with an Application Load Balancer, Auto Scaling, health checks, User Data automation, and self-healing."
tags: aws, ec2, alb, autoscaling, devops
---

# AWS Highly Available Web Application

### ALB + EC2 + Auto Scaling + Multi-AZ + Self-Healing

A hands-on AWS project that deploys an Apache-based web application across multiple Availability Zones and demonstrates load balancing, health checks, dynamic scaling, and automatic instance replacement.

---

## 1. Project Overview

The application runs on Amazon EC2 instances managed by an Auto Scaling Group.

An internet-facing Application Load Balancer receives HTTP traffic and forwards requests to healthy EC2 instances through a Target Group.

The EC2 instances are created from a Launch Template, which uses User Data to automatically install and configure Apache and generate a webpage containing instance-specific information.

The environment was tested under normal operation, scale-in, and deliberate instance termination to verify automatic replacement and recovery.

### Architecture

```text
                         Internet
                            │
                         HTTP :80
                            │
                            ▼
                  ┌───────────────────┐
                  │ Application Load  │
                  │ Balancer (ALB)    │
                  │   web-app-alb     │
                  └─────────┬─────────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   Target Group    │
                  │   web-server-tg   │
                  │   HTTP Health     │
                  │      Checks       │
                  └─────────┬─────────┘
                            │
                  ┌─────────┴─────────┐
                  ▼                   ▼
          ┌─────────────┐     ┌─────────────┐
          │    EC2      │     │    EC2      │
          │ ap-south-1a │     │ ap-south-1b │
          └─────────────┘     └─────────────┘
                  ▲                   ▲
                  └─────────┬─────────┘
                            │
                  ┌───────────────────┐
                  │ Auto Scaling Group │
                  │    web-app-asg     │
                  └─────────┬─────────┘
                            │
                  ┌───────────────────┐
                  │  Launch Template  │
                  │   web-server-lt   │
                  └───────────────────┘
                            │
                         User Data
                            │
                      Apache + HTML
```

---

## 2. AWS Components

| Component | Configuration / Role |
|---|---|
| Amazon EC2 | Runs Apache and the web application |
| Launch Template | Reusable EC2 configuration blueprint |
| User Data | Installs/configures Apache and generates the application page |
| Security Groups | Controls ALB and EC2 traffic |
| Target Group | Maintains EC2 targets and performs HTTP health checks |
| Application Load Balancer | Public HTTP entry point and traffic distribution |
| Auto Scaling Group | Maintains and replaces EC2 capacity |
| CloudWatch | Supplies CPU utilization metrics for Target Tracking |
| Availability Zones | `ap-south-1a` and `ap-south-1b` |

---

## 3. Network & Security Configuration

Two Security Groups were used.

### SG-ALB

```text
Inbound:
HTTP :80 → 0.0.0.0/0

Outbound:
All traffic
```

### SG-EC2

```text
Inbound:
HTTP :80 → SG-ALB

Outbound:
All traffic
```

Application traffic therefore follows:

```text
Internet
    │
    │ HTTP :80
    ▼
 SG-ALB
    │
    │ HTTP :80
    ▼
 SG-EC2
    │
    ▼
   EC2
```

This keeps HTTP access to the EC2 instances restricted to traffic originating from the ALB.

For instance-level testing, SSH was temporarily allowed:

```text
SSH :22
Source: <public-IP>/32
```

The SSH rule was removed during cleanup.

---

## 4. Launch Template & User Data

### Launch Template

| Setting | Value |
|---|---|
| Name | `web-server-lt` |
| AMI | Amazon Linux 2023 |
| Architecture | 64-bit x86 |
| Instance type | `t3.micro` |
| Key pair | `lab-ec2` |
| Security Group | `SG-EC2` |

The initial Launch Template version had an AMI configuration issue. Version 2 was created with the correct Amazon Linux 2023 AMI and made the default version used by the ASG.

### User Data

User Data performs the initial EC2 configuration automatically:

```text
Launch EC2
    │
    ├── Update packages
    ├── Install httpd
    ├── Start httpd
    ├── Enable httpd
    ├── Retrieve instance metadata
    │      ├── Instance ID
    │      ├── Availability Zone
    │      └── Region
    │
    └── Generate application HTML
```

The application displays:

```text
AWS Highly Available Web Application

Built by Tejas Shinkar

ELB + Auto Scaling + EC2 + CloudWatch

Instance ID: ...
Availability Zone: ...
Region: ap-south-1
```

The instance-specific information makes backend changes visible when requests are served through the ALB.

---

## 5. Target Group & Health Checks

### Target Group

| Setting | Value |
|---|---|
| Name | `web-server-tg` |
| Target type | Instances |
| Protocol | HTTP |
| Port | 80 |
| Health check protocol | HTTP |
| Path | `/` |
| Port | Traffic port |
| Interval | 30 seconds |
| Timeout | 5 seconds |
| Healthy threshold | 2 |
| Unhealthy threshold | 2 |
| Success code | 200 |

The Target Group was initially created with **0 targets intentionally**. Targets were not manually registered because the Auto Scaling Group was configured to register its instances automatically.

### Health check flow

```text
Target Group
      │
      │ HTTP GET /
      ▼
   EC2 :80
      │
      │ HTTP 200
      ▼
   Healthy
```

An EC2 instance being in a `running` state does not by itself make it a healthy application target. The Target Group checks the HTTP application response.

---

## 6. Application Load Balancer

### ALB Configuration

| Setting | Value |
|---|---|
| Name | `web-app-alb` |
| Type | Application Load Balancer |
| Scheme | Internet-facing |
| IP address type | IPv4 |
| VPC | Default VPC |
| Availability Zones | `ap-south-1a`, `ap-south-1b` |
| Security Group | `SG-ALB` |

### Listener

```text
Protocol: HTTP
Port: 80
Default action: Forward to web-server-tg
```

### Request flow

```text
Browser
   │
   ▼
ALB :80
   │
   ▼
web-server-tg
   │
   ▼
Healthy EC2 :80
   │
   ▼
Apache
   │
   ▼
Application response
```

With two healthy instances, repeated requests to the ALB endpoint displayed different Instance IDs and Availability Zones, confirming that requests were reaching different healthy backends.

---

## 7. Auto Scaling Group

### Configuration

| Setting | Value |
|---|---|
| Name | `web-app-asg` |
| Launch Template | `web-server-lt` |
| Desired capacity | 2 |
| Minimum capacity | 1 |
| Maximum capacity | 4 |
| Availability Zones | `ap-south-1a`, `ap-south-1b` |
| Target Group | `web-server-tg` |
| Health check grace period | 300 seconds |

The ASG launches instances using the Launch Template and automatically registers them with the Target Group.

The 300-second grace period allows a new instance to boot, execute User Data, install/start Apache, and become ready for application health checks.

### Initial deployment

```text
Auto Scaling Group
       │
       ├──────────────┐
       ▼              ▼
     EC2 #1         EC2 #2
   ap-south-1a    ap-south-1b
       │              │
       └──────┬───────┘
              ▼
        Target Group
              │
              ▼
             ALB
```

---

## 8. Target Tracking & Scale-In

A Target Tracking policy was configured after the initial deployment was verified.

| Setting | Value |
|---|---|
| Policy | Target Tracking |
| Metric | Average CPU Utilization |
| Target | 50% |
| Instance warmup | 300 seconds |
| Scale in | Enabled |
| Scale out | Enabled |

```text
EC2 CPU Utilization
        │
        ▼
    CloudWatch
        │
        ▼
Target Tracking
        │
        ▼
Auto Scaling Group
      /   \
     ▼     ▼
Scale Out  Scale In
   │          │
 Add EC2    Remove EC2
```

### Observed scale-in

The workload remained low after the policy was enabled, so the ASG scaled in.

The instance being removed entered the Target Group's **Draining** state before termination.

```text
Healthy EC2
     │
     ▼
ASG decides to scale in
     │
     ▼
Target Group → Draining
     │
     ▼
Instance terminates
```

The environment eventually reached:

```text
Desired capacity: 1
Healthy targets: 1
```

At this point, repeated ALB refreshes naturally showed the same Instance ID because only one healthy backend remained.

---

## 9. Application Verification

### EC2 verification

SSH was temporarily enabled for instance-level verification.

Apache was checked with:

```bash
sudo systemctl status httpd
```

Result:

```text
active (running)
```

The application was then verified locally:

```bash
curl localhost
```

The command returned the custom HTML page.

### ALB verification

The application was accessed through the ALB DNS endpoint.

The page displayed:

```text
Instance ID
Availability Zone
Region
```

This verified the complete path:

```text
Browser
  ↓
ALB
  ↓
Target Group
  ↓
EC2
  ↓
Apache
  ↓
Application
```

### Load-balancing verification

When two backend instances were healthy, repeated ALB requests displayed different Instance IDs/AZs.

This provided a simple visual verification that the ALB was distributing traffic across healthy targets.

---

## 10. Failure Simulation & Self-Healing

An active EC2 instance managed by the ASG was deliberately terminated.

### Recovery flow

```text
Active EC2
    │
    ▼
Manual termination
    │
    ▼
ASG detects capacity loss
    │
    ▼
Replacement EC2 launched
    │
    ▼
Launch Template applied
    │
    ▼
User Data executes
    │
    ▼
Apache configured
    │
    ▼
Target Group registration
    │
    ▼
HTTP health check
    │
    ▼
Healthy
    │
    ▼
ALB can route traffic
```

The replacement instance was launched automatically using the Launch Template.

Because User Data was part of the launch configuration, the replacement instance automatically received the Apache configuration and application page.

The replacement was then automatically registered with `web-server-tg` and became **Healthy**.

This verified the self-healing behavior of the architecture.

---

## 11. Troubleshooting

### Launch Template AMI configuration

The initial Launch Template version did not contain the expected AMI configuration.

**Fix:** Created Version 2 with Amazon Linux 2023 and made it the default version.

### SSH connectivity

The initial SSH connection failed because port 22 was not allowed by `SG-EC2`.

**Fix:** Temporarily added:

```text
SSH :22
Source: <public-IP>/32
```

SSH then worked and allowed:

```bash
sudo systemctl status httpd
curl localhost
```

The temporary SSH rule was removed during cleanup.

### Same Instance ID after refresh

After Target Tracking was enabled, low workload caused the ASG to scale from 2 instances down to 1.

Therefore, subsequent ALB refreshes showed the same Instance ID.

The correct troubleshooting path was to check:

```text
ASG capacity
      +
Target Group health
      +
Registered targets
```

before assuming the ALB was not distributing traffic.

---

## 12. Production Considerations

This project intentionally used a simplified setup for demonstration.

For a production deployment, the architecture would typically be extended with:

- HTTPS/TLS on the ALB instead of HTTP.
- A deliberately designed VPC and subnet architecture instead of the Default VPC.
- IAM roles for EC2 rather than relying on credentials.
- CloudWatch alarms and operational monitoring.
- More deliberate access controls and administrative access patterns.
- A real application instead of the demonstration HTML page.

---

## 13. Cleanup

After testing, all project-specific resources were removed.

The cleanup sequence was:

```text
1. Auto Scaling Group
        ↓
2. EC2 instances
        ↓
3. Application Load Balancer
        ↓
4. Target Group
        ↓
5. Launch Template
        ↓
6. Security Groups
        ↓
7. Final resource verification
```

Final project resources:

```text
web-app-asg  → Deleted
EC2          → Terminated
web-app-alb  → Deleted
web-server-tg → Deleted
web-server-lt → Deleted
SG-ALB       → Deleted
SG-EC2       → Deleted
```

No project-specific running resources were left behind after cleanup.

---

## 14. Project Result

The completed project demonstrated the following end-to-end behavior:

```text
                         ┌─────────────────────┐
                         │      Internet       │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │        ALB          │
                         └──────────┬──────────┘
                                    │
                                    ▼
                         ┌─────────────────────┐
                         │    Target Group     │
                         │   Health Checks     │
                         └──────────┬──────────┘
                                    │
                         ┌──────────┴──────────┐
                         ▼                     ▼
                    ┌──────────┐          ┌──────────┐
                    │   EC2    │          │   EC2    │
                    │   AZ-A   │          │   AZ-B   │
                    └────┬─────┘          └────┬─────┘
                         ▲                     ▲
                         └──────────┬──────────┘
                                    │
                         ┌─────────────────────┐
                         │    Auto Scaling     │
                         │       Group         │
                         └──────────┬──────────┘
                                    │
                         ┌─────────────────────┐
                         │  Launch Template    │
                         │  + User Data        │
                         └─────────────────────┘

                  CloudWatch → Target Tracking
```

### Verified

- Multi-AZ EC2 deployment.
- ALB-based application access.
- Target Group health checks.
- Automatic target registration through ASG.
- User Data-based server configuration.
- CPU Target Tracking.
- Scale-in and target draining.
- Automatic replacement after EC2 termination.
- Replacement target becoming healthy.
- ALB routing to the recovered backend.
- Complete resource cleanup.

---

*AWS Cloud + DevOps — Project: Highly Available Web Application*
