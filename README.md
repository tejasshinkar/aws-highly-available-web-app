# AWS Highly Available Web Application

A hands-on AWS project demonstrating a highly available, self-healing web application using:

- Application Load Balancer (ALB)
- EC2
- Auto Scaling Group
- Launch Template
- Target Groups & Health Checks
- CloudWatch
- Multi-AZ architecture

## Architecture

```text
                         Internet
                            │
                            ▼
                    ┌───────────────┐
                    │      ALB      │
                    └───────┬───────┘
                            │
                     Target Group
                      Health Checks
                       /         \
                      ▼           ▼
                  EC2 AZ-1a   EC2 AZ-1b
                      \           /
                       \         /
                      Auto Scaling
                           │
                    Launch Template
                           │
                       CloudWatch
```

## What I Implemented

- Created EC2 instances across multiple Availability Zones.
- Configured an Application Load Balancer and Target Group.
- Configured health checks for the web servers.
- Created a Launch Template with automated web-server setup.
- Configured an Auto Scaling Group with min/max/desired capacity.
- Added Target Tracking based on average CPU utilization.
- Verified ALB traffic distribution across instances.
- Simulated instance failure by terminating an active instance.
- Verified Auto Scaling launched a replacement instance.
- Verified the replacement instance passed health checks and joined the Target Group.

## Failure & Self-Healing Test

An active EC2 instance was deliberately terminated.

```text
EC2 instance terminated
        ↓
Target becomes unhealthy
        ↓
Auto Scaling detects capacity change
        ↓
Replacement EC2 launched
        ↓
Launch Template User Data runs
        ↓
Apache starts
        ↓
Target Group health check passes
        ↓
Application becomes available again
```

## Key AWS Services

| Service | Purpose |
|---|---|
| EC2 | Web-server compute |
| ALB | Distributes incoming HTTP traffic |
| Target Group | Registers and health-checks EC2 instances |
| Auto Scaling | Maintains desired capacity and replaces failed instances |
| Launch Template | Defines how replacement instances are launched |
| CloudWatch | Provides metrics used for scaling |

---

## GitHub Repository Structure

```text
aws-highly-available-web-app/
│
├── README.md
│
├── screenshots/
│   ├── ...
│
└── docs/
    ├── Project-Documentation.pdf
    └── Project-Documentation.docx
```

### Uploading the Project

On the GitHub **Add file → Upload files** page, you can upload the prepared project contents together.

Drag the contents of the local project folder into the GitHub upload area:

```text
aws-highly-available-web-app/
│
├── README.md
├── screenshots/
│   ├── ...
│
└── docs/
    ├── PDF
    └── DOCX
```

GitHub should preserve the folder structure.

**Important:** Upload the contents of the project folder so that `README.md`, `screenshots/`, and `docs/` appear directly in the repository root. Do not create an unnecessary extra `aws-highly-available-web-app/` folder inside the repository.
