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
