# VProfile — AWS Re-Architecture

Re-architected an existing multi-tier VProfile application from an EC2-based deployment model to an AWS managed-service architecture, integrating Amazon RDS, ElastiCache, Amazon MQ, Elastic Beanstalk, HTTPS, and CloudFront.

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/7d14b305-a3b9-41a2-9d46-519b012f3e6c" />

---

## Overview

The original VProfile environment relied heavily on self-managed services running on EC2.

This project re-architects the infrastructure around AWS managed services to reduce infrastructure management overhead while preserving the application's existing workload.

The resulting architecture separates the application tier from the backend services and introduces a global CDN in front of the application.

### Architecture transformation

| Original | Re-Architected |
|---|---|
| MySQL on EC2 | Amazon RDS |
| Memcached on EC2 | Amazon ElastiCache |
| RabbitMQ on EC2 | Amazon MQ |
| Manually managed application tier | Elastic Beanstalk |
| Direct application access | CloudFront → ALB |
| Manual application deployment | Elastic Beanstalk artifact deployment |

The project focuses on **cloud re-architecture and deployment**, rather than application development.

---

## Application Ownership Boundary

The VProfile application was used as an **existing workload** for this project.

I did **not** develop the VProfile application's Java business logic, authentication logic, or original application architecture.

My work focused on the AWS engineering surrounding the application:

- Provisioning the AWS infrastructure
- Replacing self-managed backend services with AWS managed services
- Configuring network and security boundaries
- Connecting the application to the managed backend services
- Configuring Elastic Beanstalk
- Building and deploying the application artifact
- Configuring HTTPS and DNS
- Integrating CloudFront
- Validating the resulting system

This distinction is intentional: the project demonstrates **DevOps/cloud infrastructure engineering around an existing application workload**.

---

## Architecture

```text
                         USER
                           │
                           ▼
                    Public DNS / URL
                           │
                           ▼
                     CloudFront
                    Global CDN
                           │
                           ▼
              Application Load Balancer
                    (Beanstalk)
                           │
                           ▼
                Elastic Beanstalk
                 Tomcat / EC2
                           │
             ┌─────────────┼─────────────┐
             │             │             │
             ▼             ▼             ▼
        Amazon RDS    ElastiCache     Amazon MQ
          MySQL        Memcached       RabbitMQ
```

The application tier is managed through **Elastic Beanstalk**, while the database, cache, and messaging tiers use AWS managed services.

CloudFront sits in front of the Beanstalk load balancer to provide globally distributed content delivery.

For the detailed architecture and traffic flows:

→ [Architecture](docs/architecture.md)

---

## My Engineering Contribution

### AWS Infrastructure

- Created the backend security boundary for the managed services.
- Provisioned Amazon RDS for MySQL.
- Provisioned Amazon ElastiCache for Memcached.
- Provisioned Amazon MQ for RabbitMQ.
- Configured private backend-service access.
- Established application-to-backend security-group connectivity.

### Application Platform

- Created the Elastic Beanstalk environment.
- Configured the application tier and load-balancer behavior.
- Configured the application health check to use `/login`.
- Configured rolling deployment behavior.

### Application Deployment

- Used the `awsrefactor` application branch supplied for the re-architected environment.
- Collected the managed-service endpoints.
- Updated the application configuration with the AWS service endpoints.
- Built the application using Maven.
- Produced the `vprofile-v2.war` deployment artifact.
- Deployed the artifact through Elastic Beanstalk.

### Security and Traffic

- Configured HTTPS using an ACM certificate.
- Configured the application domain.
- Created the CloudFront distribution with the Beanstalk load balancer as the origin.
- Updated DNS to route the application through CloudFront.

### Validation and Operations

- Validated application availability.
- Validated RDS-backed authentication and application data access.
- Validated ElastiCache connectivity.
- Validated HTTPS.
- Validated CloudFront routing through HTTP response headers.
- Performed dependency-aware AWS resource cleanup.

---

## Key Engineering Decisions

### Managed-Service Substitution

The core modernization pattern was to replace self-managed services with AWS managed equivalents:

```text
MySQL on EC2      → Amazon RDS
Memcached on EC2  → Amazon ElastiCache
RabbitMQ on EC2   → Amazon MQ
```

This moves infrastructure responsibilities such as service management, patching, and operational maintenance toward AWS-managed services.

### Platform Abstraction

Elastic Beanstalk abstracts much of the application-tier infrastructure, including the application instances, load balancer, scaling configuration, artifact handling, and health monitoring.

### Security-Group Dependency

The backend security group was created before the Beanstalk environment because the Beanstalk security group did not yet exist.

After Beanstalk created its application-tier security group, the backend security group was updated to allow traffic from that security group.

This demonstrates a practical dependency-ordering pattern when infrastructure resources reference one another.

### Endpoint-Driven Application Configuration

The application configuration had to be updated with the actual AWS service endpoints before building the deployment artifact.

```text
AWS managed services
        ↓
Collect endpoints
        ↓
Update application.properties
        ↓
Maven build
        ↓
WAR artifact
        ↓
Elastic Beanstalk deployment
```

### CDN Front-Loading

CloudFront was placed in front of the application load balancer:

```text
User
  ↓
CloudFront
  ↓
ALB
  ↓
Beanstalk / EC2
```

This allows CloudFront edge locations to serve cacheable content closer to users while the Beanstalk environment remains the application origin.

---

## Validation

The deployed system was validated at multiple layers.

### Application

- Application login page loads.
- Login succeeds against the RDS-backed application database.
- Application data can be retrieved.
- Memcached functionality is verified.

### Deployment

- Elastic Beanstalk environment reaches a healthy state.
- Application instances pass the configured health check.
- Rolling deployment progresses across instances.

### HTTPS

- ACM certificate is attached to the HTTPS listener.
- Application is accessible over HTTPS.

### CloudFront

CloudFront integration was validated by inspecting the HTTP response headers in browser developer tools.

A `Via` header containing a `.cloudfront.net` value provides evidence that the request passed through CloudFront.

```text
Browser
   ↓
CloudFront
   ↓
ALB
   ↓
Application Instance
```

For the complete validation methodology:

→ [Validation](docs/validation.md)

---

## Technologies

### AWS

- Amazon RDS
- Amazon ElastiCache
- Amazon MQ
- Elastic Beanstalk
- EC2
- Application Load Balancer
- Auto Scaling
- Amazon S3
- CloudWatch
- CloudFront
- AWS Certificate Manager
- Security Groups
- DNS

### Application / Deployment

- Java
- Apache Maven
- WAR deployment
- Git
- `application.properties`

---

## Project Boundaries

This project demonstrates AWS infrastructure re-architecture and application deployment around an existing workload.

It does **not** demonstrate:

- Development of the VProfile Java application
- Development of the VProfile business logic
- Terraform or other Infrastructure as Code implementation
- A complete CI/CD pipeline
- Production-grade WAF implementation
- Multi-region application deployment
- Production-grade high-availability architecture for every backend service
- A complete production observability platform

The project should therefore be understood as a **hands-on AWS re-architecture and deployment project**, not as a claim of building the underlying application or a complete production platform.

For the detailed boundaries and potential next steps:

→ [Limitations & Future Work](docs/limitations-and-future-work.md)

---

## Documentation

| Document | Purpose |
|---|---|
| [Architecture](docs/architecture.md) | Complete architecture, relationships, traffic flows, and security boundaries |
| [Implementation](docs/implementation.md) | Infrastructure assembly, configuration, build, and deployment process |
| [Validation](docs/validation.md) | Validation strategy, checks, and evidence mapping |
| [Limitations & Future Work](docs/limitations-and-future-work.md) | Current boundaries and potential architectural evolution |

---

## Evidence

High-signal evidence from the completed environment is maintained separately from the core documentation.

→ [Evidence](evidence/screenshots/)

Evidence is intended to support important implementation and validation claims without turning the repository into a collection of screenshots.

---

## Architecture Diagram

![VProfile AWS Re-Architecture](architecture.png)

---

## Project Summary

```text
Existing VProfile Workload
          │
          ▼
AWS Re-Architecture
          │
          ├── Elastic Beanstalk
          ├── Amazon RDS
          ├── Amazon ElastiCache
          ├── Amazon MQ
          ├── HTTPS / ACM
          └── CloudFront
          │
          ▼
Deployment
          │
          ▼
End-to-End Validation
          │
          ▼
Dependency-Aware Cleanup
```

**Core engineering capability demonstrated:**

> Re-architecting and deploying a multi-tier application on AWS by replacing self-managed infrastructure with managed services, establishing secure service connectivity, deploying the application through Elastic Beanstalk, integrating CloudFront, and validating the resulting end-to-end system.
