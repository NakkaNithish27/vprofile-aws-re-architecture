# VProfile — AWS Re-Architecture

[← Back to README](../README.md)

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/648cf8ea-c45b-4d00-abdf-c98b79acf217" />

---

## 1. Architecture Overview

This project re-architects an existing multi-tier VProfile application from a manually managed, EC2-based deployment model toward an AWS managed-service architecture.

The architectural goal is to move infrastructure responsibilities from individually managed EC2 instances and manually assembled application-tier components toward managed AWS services.

The transformation is:

```text
                 BEFORE
        EC2-Based / Lift & Shift
                 │
                 ▼
      ┌─────────────────────────┐
      │ Application Tier        │
      │ Tomcat on EC2           │
      │                         │
      │ Manually managed ALB    │
      │ Manually managed ASG    │
      │ Manual artifact flow    │
      └─────────────────────────┘
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
    MySQL    Memcached   RabbitMQ
    on EC2     on EC2     on EC2
```

becomes:

```text
                 AFTER
       AWS Managed-Service Model
                 │
                 ▼
              CloudFront
                 │
                 ▼
          Application Load
             Balancer
                 │
                 ▼
        Elastic Beanstalk
          ┌──────────────┐
          │ Tomcat / EC2 │
          │ Auto Scaling │
          │ CloudWatch   │
          │ S3 artifacts │
          └──────────────┘
                 │
       ┌─────────┼─────────┐
       ▼         ▼         ▼
     RDS     ElastiCache  Amazon MQ
    MySQL     Memcached   RabbitMQ
```

The key architectural idea is:

> **Keep the existing application workload while replacing much of the infrastructure underneath it with AWS-managed services.**

---

## 2. Architectural Transformation

The main service substitutions are:

| Previous Architecture | Re-Architected Architecture | Architectural Effect |
|---|---|---|
| MySQL on EC2 | Amazon RDS | Managed database |
| Memcached on EC2 | Amazon ElastiCache | Managed caching |
| RabbitMQ on EC2 | Amazon MQ for RabbitMQ | Managed message broker |
| Manually managed Tomcat EC2 | Elastic Beanstalk | Managed application platform |
| Manually managed ALB | Beanstalk-managed ALB | Platform-managed load balancing |
| Manually managed ASG | Beanstalk-managed Auto Scaling | Platform-managed scaling |
| Manual artifact handling | Beanstalk/S3 deployment flow | Simplified deployment |
| Direct origin access | CloudFront → ALB | CDN front layer |
| Manual application-tier monitoring/scaling configuration | Beanstalk/CloudWatch integration | More platform-managed operations |

The architecture therefore changes the **responsibility boundary**, not simply the location of the same software.

---

## 3. Final Architecture

```text
                           USERS
                             │
                             │ HTTPS
                             ▼
                    ┌─────────────────┐
                    │    DNS / URL    │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   CloudFront    │
                    │      CDN        │
                    │   HTTPS / SSL   │
                    └────────┬────────┘
                             │
                             │ Origin request
                             ▼
              ┌────────────────────────────────┐
              │     ELASTIC BEANSTALK          │
              │                                │
              │   ┌────────────────────────┐   │
              │   │ Application Load       │   │
              │   │ Balancer               │   │
              │   │ HTTPS :443             │   │
              │   └───────────┬────────────┘   │
              │               │                │
              │               ▼                │
              │   ┌────────────────────────┐   │
              │   │ EC2 / Tomcat           │   │
              │   │ Application Instances  │   │
              │   └───────────┬────────────┘   │
              │               │                │
              │   ┌───────────┴────────────┐   │
              │   │ Auto Scaling Group     │   │
              │   │ CloudWatch monitoring  │   │
              │   │ S3 artifact storage    │   │
              │   └────────────────────────┘   │
              └───────────────┬────────────────┘
                              │
                              │ Backend traffic
                              │ through SG rules
                              ▼
              ┌────────────────────────────────┐
              │     BACKEND SECURITY GROUP    │
              │                                │
              │  ┌──────────────┐              │
              │  │ Amazon RDS   │              │
              │  │ MySQL        │              │
              │  └──────────────┘              │
              │                                │
              │  ┌──────────────┐              │
              │  │ ElastiCache  │              │
              │  │ Memcached    │              │
              │  └──────────────┘              │
              │                                │
              │  ┌──────────────┐              │
              │  │ Amazon MQ    │              │
              │  │ RabbitMQ     │              │
              │  └──────────────┘              │
              └────────────────────────────────┘
```

---

## 4. Architectural Layers

The system can be understood as five logical layers.

```text
┌──────────────────────────────────────────┐
│  Layer 1 — Public Entry                 │
│  DNS → CloudFront                       │
├──────────────────────────────────────────┤
│  Layer 2 — Application Ingress          │
│  ALB → HTTPS                            │
├──────────────────────────────────────────┤
│  Layer 3 — Application Platform         │
│  Elastic Beanstalk → Tomcat / EC2       │
│  Auto Scaling + CloudWatch              │
├──────────────────────────────────────────┤
│  Layer 4 — Managed Backend Services     │
│  RDS + ElastiCache + Amazon MQ           │
├──────────────────────────────────────────┤
│  Layer 5 — Network Security             │
│  Security-group relationships            │
└──────────────────────────────────────────┘
```

---

# 5. Public Entry Layer

## DNS

Users do not interact directly with AWS-generated infrastructure endpoints.

The architecture uses a human-readable application domain.

The DNS flow is:

```text
Application Domain
       ↓
CloudFront Endpoint
```

The important architectural role of DNS is:

> **Provide a stable human-readable entry point for the application while allowing the underlying AWS endpoint to change.**

---

## CloudFront

CloudFront is the new front layer introduced by the re-architecture.

Its position is:

```text
User
  ↓
CloudFront
  ↓
Beanstalk ALB
```

CloudFront acts as the CDN layer and provides edge locations for content delivery.

Its architectural responsibilities are:

- Public CDN entry point
- Edge content delivery
- HTTPS for the public endpoint
- Origin forwarding to the Beanstalk load balancer

### Why it exists

The architecture introduces CloudFront to provide a global content-delivery layer and reduce latency for users geographically distant from the application's AWS region.

---

# 6. Application Ingress Layer

## Application Load Balancer

The Application Load Balancer is part of the Elastic Beanstalk environment.

The logical flow is:

```text
CloudFront
    ↓
ALB
    ↓
Application Instances
```

The ALB provides the application ingress point within the AWS environment.

It distributes incoming application requests to the Tomcat instances managed by the Beanstalk environment.

The HTTPS configuration adds a `443` listener to the Beanstalk load balancer.

The request path is:

```text
User
  │ HTTPS
  ▼
CloudFront
  │ HTTPS
  ▼
Beanstalk ALB
  │
  ▼
Tomcat
```

---

# 7. Application Platform Layer

## Elastic Beanstalk

Elastic Beanstalk is the central platform abstraction in the re-architected application tier.

Instead of manually assembling:

```text
EC2
+
ALB
+
Auto Scaling
+
S3 artifact handling
+
CloudWatch integration
+
Security groups
```

the project uses:

```text
Elastic Beanstalk Environment
```

as the managed application platform.

### Logical components managed through Beanstalk

```text
Elastic Beanstalk
      │
      ├── EC2 application instances
      │       └── Tomcat
      │
      ├── Application Load Balancer
      │
      ├── Auto Scaling Group
      │
      ├── S3 artifact storage
      │
      ├── CloudWatch integration
      │
      └── Application-tier security groups
```

The important architectural point is:

> **Beanstalk does not eliminate the underlying EC2/ALB/ASG resources; it moves their lifecycle and configuration under a managed application platform.**

---

## Auto Scaling

The Tomcat application instances are associated with an Auto Scaling Group created and managed as part of the Beanstalk environment.

```text
ALB
 ↓
Auto Scaling Group
 ├── EC2 / Tomcat
 ├── EC2 / Tomcat
 └── additional instances when scaling occurs
```

CloudWatch metrics and alarms are integrated into the Beanstalk-managed environment and can participate in scaling decisions.

---

## Application Health

The application health check is aligned with the actual application entry point.

The configured health-check path is:

```text
/login
```

A health check must test a path that actually produces the expected successful response from the deployed application.

Conceptually:

```text
Wrong health path
      ↓
Health check fails
      ↓
Instance marked unhealthy
      ↓
Auto Scaling may replace instance
      ↓
New instance fails same check
      ↓
Potential replacement loop
```

---

# 8. Backend Service Layer

The backend architecture replaces software running on EC2 with AWS-managed services.

```text
                    APPLICATION
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
       RDS          ElastiCache      Amazon MQ
       MySQL          Memcached       RabbitMQ
```

These services are logically grouped as the backend tier.

---

## Amazon RDS

Amazon RDS replaces the MySQL database that previously ran on EC2.

```text
Before:
Application → MySQL on EC2

After:
Application → Amazon RDS
```

The application receives the RDS endpoint and uses it in its configuration.

The RDS instance remains a backend service rather than becoming part of the Beanstalk application environment.

This separation is important:

```text
Elastic Beanstalk
     ≠
RDS
```

RDS has its own lifecycle and remains independent of the application platform.

The database is initialized separately using the provided database backup through a temporary EC2 client.

---

## ElastiCache

Amazon ElastiCache provides the managed Memcached layer.

```text
Before:
Application → Memcached on EC2

After:
Application → Amazon ElastiCache
```

The application receives the ElastiCache endpoint and uses it in its configuration.

The cache remains a backend dependency outside the Beanstalk application platform.

---

## Amazon MQ

The completed practical uses **Amazon MQ with RabbitMQ**.

```text
Before:
Application → RabbitMQ on EC2

After:
Application → Amazon MQ / RabbitMQ
```

The implemented broker configuration uses RabbitMQ with private access and encrypted connectivity.

### Source-material clarification

Some introductory material refers to Amazon MQ with **ActiveMQ**, but the dedicated Amazon MQ practical explicitly creates **RabbitMQ** and the later application configuration is built around RabbitMQ.

For this repository architecture, the implemented architecture is therefore represented as:

> **Amazon MQ for RabbitMQ**

rather than ActiveMQ.

---

# 9. Backend Security Boundary

The backend services share a manually created backend security group.

Conceptually:

```text
             BEANSTALK INSTANCE SG
                      │
                      │ allowed inbound
                      ▼
             BACKEND SECURITY GROUP
                      │
          ┌───────────┼───────────┐
          ▼           ▼           ▼
         RDS      ElastiCache   Amazon MQ
```

The backend security group contains the managed backend services.

The application instances are permitted to initiate the required backend connections through security-group rules.

---

# 10. Security-Group Dependency

One of the most important architectural dependencies in the project is temporal.

At the beginning:

```text
Backend SG
    │
    ├── RDS
    ├── ElastiCache
    └── Amazon MQ

Beanstalk SG
    └── DOES NOT EXIST YET
```

Therefore, the backend SG cannot initially contain a rule referencing the Beanstalk instance SG.

The sequence becomes:

```text
1. Create backend SG
          ↓
2. Create RDS
          ↓
3. Create ElastiCache
          ↓
4. Create Amazon MQ
          ↓
5. Create Elastic Beanstalk
          ↓
6. Beanstalk creates instance SG
          ↓
7. Add Beanstalk instance SG
   as allowed source in backend SG
```

This is the **deferred security binding** pattern.

---

## Instance SG vs Load Balancer SG

Beanstalk creates separate security groups for:

```text
Load Balancer
      │
      ▼
Application Instances
```

The backend SG must reference the **application instance security group**, not the load-balancer security group.

The reason is traffic ownership:

```text
ALB
 ↓
Application Instance
 ↓
Backend Service
```

The backend service receives the connection from the application instance.

Therefore:

```text
Backend SG
   ← allow from →
Beanstalk Instance SG
```

not:

```text
Backend SG
   ← allow from →
Beanstalk ALB SG
```

---

# 11. Internal Backend Security

The backend security group also contains an internal/self-reference pattern.

Conceptually:

```text
Backend SG
    │
    └── Backend SG
          │
          ├── RDS
          ├── ElastiCache
          └── Amazon MQ
```

This allows internal backend communication according to the configured security-group rules.

---

# 12. Application-to-Backend Connectivity

The application does not discover the managed services through manually assigned EC2 private IP addresses.

Instead, each managed service provides an endpoint.

The dependency chain is:

```text
RDS
  ↓
RDS endpoint

ElastiCache
  ↓
ElastiCache endpoint

Amazon MQ
  ↓
RabbitMQ endpoint
```

These values are then injected into the application's configuration.

```text
Managed Services
      │
      ├── RDS endpoint
      ├── ElastiCache endpoint
      └── Amazon MQ endpoint
              │
              ▼
       application.properties
              │
              ▼
          Maven build
              │
              ▼
           WAR artifact
              │
              ▼
       Elastic Beanstalk
```

The important build dependency is:

> All required backend endpoints must be known before the application artifact is built.

---

# 13. Application Deployment Architecture

The deployment flow is:

```text
Existing application source
          │
          │ awsrefactor branch
          ▼
application.properties
          │
          │ backend endpoints
          ▼
        Maven
          │
          ▼
   vprofile-v2.war
          │
          ▼
 Elastic Beanstalk
          │
          ▼
Application instances
          │
          ▼
       Tomcat
```

Beanstalk then distributes the artifact to the application instances and manages the deployment process.

The project uses artifact deployment rather than manually copying the WAR to individual EC2 instances.

---

# 14. Rolling Deployment Architecture

The application tier is configured for rolling deployment.

The conceptual process is:

```text
Current state

Instance A   Instance B
   │            │
   └──── Healthy ┘
          │
          ▼
      Deploy batch
          │
          ▼
Update one portion
          │
          ▼
Validate health
          │
          ▼
Update remaining portion
```

The project configuration uses a 50% batch size for a two-instance environment, resulting in incremental deployment rather than replacing the entire application tier simultaneously.

This should be understood as the **deployment strategy configured for the practical**, not as a blanket claim of production-grade zero-downtime behavior.

---

# 15. HTTPS Architecture

The public request path uses HTTPS.

```text
User
  │
  │ HTTPS
  ▼
CloudFront
  │
  │ HTTPS
  ▼
Beanstalk ALB :443
  │
  ▼
Tomcat
```

The project therefore has two relevant TLS boundaries:

1. Public connection to CloudFront
2. CloudFront-to-origin connection through the Beanstalk load balancer

---

# 16. Complete Request Flow

The complete user request path can be reconstructed as:

```text
                         USER
                           │
                           │ HTTPS
                           ▼
                     PUBLIC DNS
                           │
                           ▼
                      CLOUDFRONT
                           │
                           │ origin request
                           ▼
                 APPLICATION LOAD
                    BALANCER
                           │
                           │ HTTP/HTTPS
                           ▼
                   TOMCAT / EC2
                           │
                           │
          ┌────────────────┼────────────────┐
          │                │                │
          ▼                ▼                ▼
       AMAZON RDS    ELASTICACHE        AMAZON MQ
         MySQL        Memcached          RabbitMQ
```

The application tier is managed by Elastic Beanstalk:

```text
CloudFront
    ↓
ALB
    ↓
Beanstalk
    ↓
Auto Scaling Group
    ↓
Tomcat EC2 instances
```

---

# 17. Backend Request Flows

Different application operations use different backend dependencies.

## Authentication / Database

```text
User
  ↓
CloudFront
  ↓
ALB
  ↓
Tomcat
  ↓
RDS MySQL
```

The login operation therefore provides a practical validation of application-to-database connectivity.

---

## Caching

```text
User
  ↓
CloudFront
  ↓
ALB
  ↓
Tomcat
  ↓
ElastiCache / Memcached
```

Application behavior that depends on caching provides evidence that the application can reach the managed cache.

---

## Messaging

```text
User
  ↓
CloudFront
  ↓
ALB
  ↓
Tomcat
  ↓
Amazon MQ / RabbitMQ
```

Messaging functionality depends on the application being able to reach the private RabbitMQ broker.

---

# 18. RDS Initialization Architecture

RDS does not provide operating-system-level access in the same way as an EC2 database server.

The project therefore uses a temporary EC2 client as a database initialization mechanism.

```text
Temporary EC2
     │
     │ private network access
     ▼
Amazon RDS
     │
     ▼
db_backup.sql
     │
     ▼
Schema + application data
```

The temporary instance functions as a **jump box/client**, not as part of the final application architecture.

Its lifecycle is therefore:

```text
Create
  ↓
Connect to RDS
  ↓
Initialize database
  ↓
Validate
  ↓
Terminate
```

This is an operational implementation pattern rather than a permanent architecture component.

---

# 19. Artifact and Infrastructure Dependencies

The architecture has several important dependencies.

```text
Backend SG
    ↓
Backend services
    ↓
Service endpoints
    ↓
Application configuration
    ↓
WAR build
    ↓
Beanstalk deployment
    ↓
CloudFront origin
    ↓
DNS
    ↓
End-to-end validation
```

More precisely:

```text
Create backend SG
        ↓
RDS / ElastiCache / Amazon MQ
        ↓
collect endpoints
        ↓
Beanstalk environment
        ↓
update backend SG with Beanstalk instance SG
        ↓
initialize RDS
        ↓
configure health check
        ↓
configure HTTPS
        ↓
build WAR with endpoints
        ↓
deploy WAR
        ↓
CloudFront
        ↓
DNS
        ↓
validation
```

---

# 20. Major Architectural Decisions

## Decision 1 — Managed-Service Substitution

### Choice

Replace self-managed backend services with:

```text
RDS
ElastiCache
Amazon MQ
```

### Reason

Reduce infrastructure-management responsibility while retaining the application's logical backend functions.

### Result

The application interacts with managed service endpoints instead of services installed directly on EC2.

---

## Decision 2 — Elastic Beanstalk for the Application Tier

### Choice

Use Elastic Beanstalk instead of individually managing:

```text
EC2
ALB
ASG
S3 artifact deployment
CloudWatch integration
```

### Reason

Provide a managed application platform that coordinates these resources.

### Result

The application tier becomes a single managed deployment environment.

---

## Decision 3 — Separate Backend Security Boundary

### Choice

Place:

```text
RDS
ElastiCache
Amazon MQ
```

behind a backend security group.

### Reason

Keep backend services separated from the public/application ingress path and control application-to-backend access through security-group rules.

### Result

The backend services are not directly exposed as public application endpoints.

---

## Decision 4 — Deferred Security Binding

### Choice

Create the backend security group before Beanstalk, then add the Beanstalk instance security-group reference afterward.

### Reason

The Beanstalk-generated security group does not exist until the environment is created.

### Result

The infrastructure sequence must respect this temporal dependency.

---

## Decision 5 — CloudFront Before the ALB

### Choice

Place CloudFront before the Beanstalk load balancer.

### Reason

Provide CDN edge delivery and a public distribution layer.

### Result

The public traffic path becomes:

```text
User
 ↓
CloudFront
 ↓
ALB
 ↓
Application
```

---

## Decision 6 — Health Check `/login`

### Choice

Use `/login` as the application health-check path.

### Reason

Align the infrastructure health probe with the application's actual reachable entry point.

### Result

Beanstalk/ALB health status reflects actual application behavior rather than an unsuitable endpoint.

---

## Decision 7 — Endpoint-Driven Configuration

### Choice

Build the application only after all managed backend endpoints are available.

### Reason

The application configuration requires those endpoints.

### Result

The build dependency becomes:

```text
Provision
   ↓
Discover endpoints
   ↓
Configure
   ↓
Build
   ↓
Deploy
```

---

# 21. Architecture Responsibility Boundaries

The final architecture has clear responsibility boundaries.

| Layer | Primary responsibility |
|---|---|
| DNS | Resolve the public application name |
| CloudFront | Global CDN/public edge layer |
| ALB | Application request distribution |
| Elastic Beanstalk | Application platform lifecycle |
| EC2/Tomcat | Run the application workload |
| Auto Scaling | Adjust application instance capacity |
| CloudWatch | Monitoring/scaling signals within the managed environment |
| RDS | Relational database |
| ElastiCache | Caching |
| Amazon MQ | Messaging |
| Security Groups | Network access control |

The important distinction is:

```text
Application workload
        │
        ▼
Runs on infrastructure
        │
        ▼
Infrastructure is increasingly managed by AWS
```

The project therefore demonstrates a change in **operational responsibility** rather than a rewrite of the application.

---

# 22. Architecture vs Application Ownership

The VProfile application remains the workload.

The architecture work concerns the environment around it:

```text
                 EXISTING APPLICATION
                         │
                         ▼
             ┌─────────────────────┐
             │ AWS ARCHITECTURE    │
             │                     │
             │ CloudFront          │
             │ ALB                 │
             │ Beanstalk           │
             │ RDS                 │
             │ ElastiCache         │
             │ Amazon MQ           │
             │ Security Groups     │
             │ HTTPS / DNS         │
             └─────────────────────┘
```

Therefore this project should be understood as:

> **Infrastructure and deployment re-architecture around an existing workload.**

It should not be interpreted as an application-development project.

---

# 23. Architectural Limitations

The architecture demonstrates the managed-service pattern, but it does not establish every characteristic expected from a production platform.

Important boundaries include:

### Amazon MQ availability

The practical uses a **single-instance RabbitMQ broker** for learning rather than a production HA cluster.

### Infrastructure as Code

The practical does not demonstrate a complete Terraform/IaC representation of the architecture.

### CI/CD

The deployment flow is artifact-based through Elastic Beanstalk rather than a complete automated CI/CD pipeline.

### Security hardening

The architecture does not establish a complete production security platform such as a fully implemented WAF strategy.

### Observability

The architecture uses Beanstalk/CloudWatch capabilities but does not establish a complete enterprise observability stack.

### Multi-region architecture

The project demonstrates a regional AWS deployment with CloudFront at the global distribution layer; it does not establish multi-region application failover.

These boundaries are documented further in:

[Limitations & Future Work](limitations-and-future-work.md)

---

# 24. Architecture Mental Model

The entire architecture can be compressed into one model:

```text
                    PUBLIC
                      │
                      ▼
                   DNS
                      │
                      ▼
                CloudFront
              Global Edge Layer
                      │
                      ▼
                    ALB
              Application Ingress
                      │
                      ▼
             Elastic Beanstalk
                      │
          ┌───────────┴───────────┐
          ▼                       ▼
     Auto Scaling              CloudWatch
          │
          ▼
      Tomcat / EC2
          │
          │ secure backend access
          ▼
   ┌──────┼────────┐
   ▼      ▼        ▼
  RDS   Cache      MQ
 MySQL Memcached RabbitMQ
```

And the architectural transformation can be remembered as:

```text
SELF-MANAGED EC2
       ↓
MANAGED AWS SERVICES

MySQL       → RDS
Memcached   → ElastiCache
RabbitMQ    → Amazon MQ
Tomcat/EC2  → Elastic Beanstalk
Direct ALB  → CloudFront → ALB
```

The central engineering principle is:

> **Replace infrastructure that the team previously managed directly with AWS-managed services while preserving the application's logical workload and integrating the resulting services through explicit network, security, configuration, and deployment dependencies.**

---

## Navigation

- [← Back to README](../README.md)
- [Implementation](implementation.md)
- [Validation](validation.md)
- [Limitations & Future Work](limitations-and-future-work.md)
