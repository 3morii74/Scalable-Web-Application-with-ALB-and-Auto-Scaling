# Project 1: Scalable Web Application with ALB and Auto Scaling

**Track:** AWS Solutions Architect – Associate (Graduation Project)
**Architecture type:** EC2-based

## 1. Overview

Deploy a production-grade web application on AWS using EC2 instances inside a properly architected VPC, with public and private subnets across **two Availability Zones**. High availability and scalability are achieved with an Application Load Balancer (ALB), an Auto Scaling Group (ASG), and a CloudFront distribution that caches static assets. A Multi-AZ RDS instance is the database backend, and **all compute runs in private subnets**.

## 2. Key AWS Services

| Service              | Role in the solution                                            |
| -------------------- | --------------------------------------------------------------- |
| **VPC**              | Public and private subnets, NAT Gateway, Security Groups, NACLs |
| **EC2 + ASG**        | Launch Template, target tracking scaling policies               |
| **ALB + WAF**        | Layer 7 routing, WAF rules for the OWASP Top 10                 |
| **CloudFront**       | Caches static assets, reduces latency                           |
| **RDS Multi-AZ**     | MySQL/PostgreSQL with automated failover                        |
| **Route 53**         | Alias record pointing to the CloudFront/ALB, health checks      |
| **Systems Manager**  | Session Manager for secure, bastion-free instance access        |
| **CloudWatch + SNS** | Dashboards, alarms, and notifications                           |

## 3. Solution Architecture Diagram

High-level view of the full solution:

![Project 1 AWS architecture diagram](project-1-architecture-diagram.png)

## 4. Network Layout (VPC, Subnets, Route Tables)

Three subnet tiers per Availability Zone, each with its own route table logic.

```mermaid
flowchart LR
    subgraph VPC["VPC 10.0.0.0/16"]
        subgraph AZA["Availability Zone A"]
            PA["Public subnet A<br/>10.0.1.0/24<br/>ALB node, NAT GW"]
            AA["Private app subnet A<br/>10.0.11.0/24<br/>EC2 instances"]
            DA["Private data subnet A<br/>10.0.21.0/24<br/>RDS primary"]
        end
        subgraph AZB["Availability Zone B"]
            PB["Public subnet B<br/>10.0.2.0/24<br/>ALB node, NAT GW"]
            AB["Private app subnet B<br/>10.0.12.0/24<br/>EC2 instances"]
            DB2["Private data subnet B<br/>10.0.22.0/24<br/>RDS standby"]
        end
    end

    IGW["Internet Gateway"]
    PA -- "0.0.0.0/0" --> IGW
    PB -- "0.0.0.0/0" --> IGW
    AA -- "0.0.0.0/0 via NAT-A" --> PA
    AB -- "0.0.0.0/0 via NAT-B" --> PB
    DA -. "no internet route" .- DA
    DB2 -. "no internet route" .- DB2
```

| Route table | Associated subnets | Routes                                                |
| ----------- | ------------------ | ----------------------------------------------------- |
| `rt-public` | Public A, Public B | `10.0.0.0/16` → local, `0.0.0.0/0` → Internet Gateway |
| `rt-app-a`  | Private app A      | `10.0.0.0/16` → local, `0.0.0.0/0` → NAT GW A         |
| `rt-app-b`  | Private app B      | `10.0.0.0/16` → local, `0.0.0.0/0` → NAT GW B         |
| `rt-data`   | Private data A, B  | `10.0.0.0/16` → local only                            |

## 5. Security Groups (Layered Access)

Each tier only accepts traffic from the tier directly in front of it.

```mermaid
flowchart LR
    Internet(["Internet / CloudFront"]) -- "HTTPS 443" --> SGALB["sg-alb"]
    SGALB -- "HTTP 80/8080" --> SGAPP["sg-app<br/>EC2 instances"]
    SGAPP -- "3306 / 5432" --> SGDB["sg-db<br/>RDS"]
    SSM["Systems Manager<br/>Session Manager"] -. "outbound 443 only, no inbound SSH" .-> SGAPP
```

| Security group | Inbound rule                          | Source                                        |
| -------------- | ------------------------------------- | --------------------------------------------- |
| `sg-alb`       | TCP 443 (and 80 redirect)             | CloudFront managed prefix list or `0.0.0.0/0` |
| `sg-app`       | TCP 80/8080                           | `sg-alb`                                      |
| `sg-db`        | TCP 3306 (MySQL) or 5432 (PostgreSQL) | `sg-app`                                      |

> **No port 22 is open anywhere.** Admin access uses Session Manager through the SSM agent and an IAM instance profile.

## 6. Request Flow

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant R as Route 53
    participant C as CloudFront
    participant S as S3 static bucket
    participant W as WAF
    participant A as ALB
    participant E as EC2 in ASG
    participant D as RDS Multi-AZ

    U->>R: Resolve www.example.com
    R-->>U: CloudFront alias
    U->>C: HTTPS request
    alt Static asset (CSS, JS, images)
        C->>S: Fetch on cache miss
        S-->>C: Object
        C-->>U: Cached response
    else Dynamic request
        C->>W: Forward to origin
        W->>A: Allowed request
        A->>E: Route to healthy target
        E->>D: Query
        D-->>E: Result
        E-->>A: Response
        A-->>C: Response
        C-->>U: Response
    end
```

## 7. Auto Scaling Behaviour

```mermaid
flowchart TD
    LT["Launch Template<br/>AMI, instance type, user data, IAM profile"] --> ASG
    ASG["Auto Scaling Group<br/>min 2, desired 2, max 6<br/>spans 2 AZs"]
    CW["CloudWatch metric<br/>Avg CPU or ALB RequestCountPerTarget"]
    POL{"Target tracking policy<br/>Target: 50 percent CPU"}
    OUT["Scale out<br/>Launch new instances"]
    IN["Scale in<br/>Terminate surplus instances"]
    REG["Register with ALB target group"]
    HC{"Health check passed?"}
    REP["ASG replaces unhealthy instance"]
    NOTIF["SNS notification"]

    CW --> POL
    POL -- "above target" --> OUT --> ASG
    POL -- "below target" --> IN --> ASG
    ASG --> REG --> HC
    HC -- "yes" --> Serve["Serve traffic"]
    HC -- "no" --> REP --> ASG
    ASG -. "launch / terminate events" .-> NOTIF
```

## 8. High Availability and Failover

```mermaid
flowchart LR
    subgraph Normal["Normal operation"]
        direction TB
        N1["ALB"] --> N2["EC2 AZ-a"]
        N1 --> N3["EC2 AZ-b"]
        N2 --> N4[("RDS Primary AZ-a")]
        N3 --> N4
        N4 -. "sync replication" .-> N5[("RDS Standby AZ-b")]
    end

    subgraph Failure["AZ-a failure"]
        direction TB
        F1["ALB"] -- "health check fails for AZ-a" --> F3["EC2 AZ-b"]
        F3 --> F5[("RDS Standby promoted in AZ-b<br/>same DNS endpoint")]
        F6["ASG launches replacement instances in AZ-b"]
    end

    Normal ==> Failure
```

- **Compute:** the ALB stops routing to failed targets, and the ASG launches replacements in the healthy AZ.
- **Database:** RDS Multi-AZ promotes the standby automatically (typically 60–120 seconds); the application keeps using the same endpoint.
- **Egress:** one NAT Gateway per AZ prevents a single AZ failure from cutting off outbound internet for the other.

## 9. Monitoring and Alerting

```mermaid
flowchart LR
    ALB["ALB metrics<br/>5xx, latency, healthy hosts"] --> CW
    EC2["EC2 / ASG metrics<br/>CPU, network, status checks"] --> CW
    RDS["RDS metrics<br/>CPU, connections, free storage"] --> CW
    WAF["WAF metrics<br/>blocked requests"] --> CW
    CW["CloudWatch<br/>Dashboards + Alarms"] --> SNS["SNS topic"]
    SNS --> Email["Email"]
    SNS --> SMS["SMS / on-call"]
```

Suggested alarms:

| Alarm                           | Threshold        | Action                    |
| ------------------------------- | ---------------- | ------------------------- |
| ALB `HTTPCode_Target_5XX_Count` | > 10 in 5 min    | SNS                       |
| ALB `UnHealthyHostCount`        | ≥ 1 for 3 min    | SNS                       |
| ASG average CPU                 | > 70% for 10 min | SNS (plus scaling policy) |
| RDS `CPUUtilization`            | > 80% for 10 min | SNS                       |
| RDS `FreeStorageSpace`          | < 10 GB          | SNS                       |
| WAF blocked requests spike      | anomaly          | SNS                       |

## 10. Implementation Steps

1. **Network:** create the VPC, 6 subnets (public, private-app, private-data × 2 AZs), Internet Gateway, 2 NAT Gateways, and route tables.
2. **Security:** create `sg-alb`, `sg-app`, `sg-db` and NACLs; create an IAM role with `AmazonSSMManagedInstanceCore` for EC2.
3. **Database:** create a DB subnet group and an RDS Multi-AZ instance (MySQL or PostgreSQL) in the private data subnets, with encryption enabled.
4. **Compute:** create a Launch Template (AMI, user data to install and start the app, instance profile) and an Auto Scaling Group across the private app subnets with a target tracking policy.
5. **Load balancing:** create the ALB in the public subnets, a target group with a health check path (e.g. `/health`), and HTTPS listener with an ACM certificate.
6. **Edge protection:** attach a WAF web ACL (AWS Managed Rules: Core rule set, SQLi, Known bad inputs) to the ALB.
7. **CDN:** create an S3 bucket for static assets and a CloudFront distribution with two origins (S3 via Origin Access Control and the ALB), with cache behaviors per path.
8. **DNS:** create a Route 53 alias record pointing to CloudFront and optional health checks.
9. **Operations:** configure Session Manager, CloudWatch dashboard and alarms, and an SNS topic with email subscription.
10. **Test:** load test to trigger scale out, terminate an instance, and reboot RDS with failover to validate resilience.
