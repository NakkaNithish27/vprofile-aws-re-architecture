# VProfile — AWS Re-Architecture: Limitations & Future Work

[← Back to README](../README.md) | [Architecture](architecture.md) | [Implementation](implementation.md) | [Validation](validation.md)

---

## 1. Purpose

This document defines the boundaries of the VProfile AWS Re-Architecture project and identifies the logical engineering capabilities that can be added in future iterations.

The purpose is to clearly distinguish between:

```text
What this project demonstrates
        ↓
What this project intentionally does not demonstrate
        ↓
What could be implemented next
```

The project should be understood as a hands-on AWS re-architecture and deployment project around an existing application workload, rather than as a complete production platform.

---

## 2. Current Project Scope

The project demonstrates the re-architecture of an existing VProfile application from a largely self-managed EC2-based infrastructure model toward AWS managed services.

The completed architecture includes:

```text
Existing VProfile Workload
          ↓
AWS Re-Architecture
          ↓
┌───────────────────────────────┐
│ Elastic Beanstalk             │
│ Amazon RDS                    │
│ Amazon ElastiCache            │
│ Amazon MQ                     │
│ Application Load Balancer     │
│ Auto Scaling                  │
│ CloudFront                    │
│ HTTPS / ACM                   │
│ Security Groups               │
└───────────────────────────────┘
          ↓
End-to-End Validation
```

The project therefore demonstrates:

- AWS managed-service substitution
- Application-tier deployment through Elastic Beanstalk
- Managed MySQL through Amazon RDS
- Managed Memcached through ElastiCache
- Managed RabbitMQ through Amazon MQ
- Application-to-backend security-group connectivity
- RDS initialization through a temporary jump box
- HTTPS configuration
- CloudFront integration
- DNS integration
- Artifact-based WAR deployment
- Rolling deployment configuration
- End-to-end application validation
- Dependency-aware infrastructure cleanup

The architectural transformation is:

```text
MySQL on EC2
      ↓
Amazon RDS

Memcached on EC2
      ↓
Amazon ElastiCache

RabbitMQ on EC2
      ↓
Amazon MQ

Manually managed application tier
      ↓
Elastic Beanstalk

Direct application access
      ↓
CloudFront → ALB
```

---

## 3. Application Ownership Limitation

The VProfile application is an existing workload used to demonstrate DevOps and cloud engineering.

This project does not claim ownership of:

- The VProfile Java business logic
- The application's original authentication implementation
- The original application architecture
- The underlying application development

The engineering work represented by this repository is the infrastructure and deployment work surrounding that workload.

```text
Existing Application
        │
        ▼
AWS Engineering Around Application
```

The repository should therefore be described as:

> Infrastructure and deployment re-architecture around an existing application workload.

It should not be presented as an application-development project.

---

## 4. Production-Readiness Boundary

The architecture demonstrates several production-oriented patterns, but it should not be described as a complete production architecture.

The project is primarily a learning and portfolio implementation.

It proves:

```text
Architecture
     ↓
Implementation
     ↓
Deployment
     ↓
Functional Validation
```

It does not prove every characteristic required for a production platform.

In particular, it does not establish:

```text
Production-grade security
Production-grade observability
Multi-region failover
Complete infrastructure automation
Complete CI/CD
Enterprise disaster recovery
Production performance characteristics
```

Therefore:

> The project demonstrates a working AWS re-architecture, not production certification of the resulting platform.

---

## 5. Amazon MQ High-Availability Limitation

The project uses a single-instance RabbitMQ broker.

The learning configuration is intentionally simplified for the project and does not demonstrate a production-grade clustered RabbitMQ deployment.

Current model:

```text
Application
     │
     ▼
Amazon MQ
     │
     └── Single RabbitMQ broker
```

This means the project does not demonstrate:

- RabbitMQ cluster deployment
- Broker-node redundancy
- Broker failover
- Production messaging-service resilience

### Future improvement

A production-oriented implementation could evolve toward:

```text
Application
     │
     ▼
Amazon MQ
     │
     ├── Broker Node
     ├── Broker Node
     └── Failover / HA
```

The exact production topology would depend on workload requirements and the Amazon MQ deployment options appropriate for the application.

---

## 6. Database High-Availability Limitation

The project demonstrates Amazon RDS as the managed database replacement, but it does not establish a complete production database high-availability and disaster-recovery strategy.

The project validates that:

```text
Application
     ↓
RDS
     ↓
Database-backed functionality
```

works successfully.

However, that is different from proving:

```text
Database failure
     ↓
Automatic recovery
     ↓
Application continues operating
```

The project does not include a comprehensive demonstration of:

- Database failover testing
- Disaster-recovery procedures
- Recovery-time objectives
- Recovery-point objectives
- Cross-region database recovery
- Production backup/restore exercises

### Future improvement

A later iteration could introduce and validate:

```text
Primary Database
       ↓
High-Availability Configuration
       ↓
Failover
       ↓
Recovery Validation
```

---

## 7. Infrastructure as Code Limitation

The AWS architecture is implemented through AWS console-driven configuration and documented implementation steps.

The project does not provide a complete Infrastructure as Code representation such as Terraform.

Current model:

```text
AWS Console
     ↓
Resource Configuration
     ↓
Working Environment
```

Future model:

```text
Terraform
     ↓
Infrastructure Definition
     ↓
AWS Resources
```

### Future improvement

The complete architecture could eventually be represented declaratively:

```text
Terraform
   │
   ├── Networking
   ├── Security Groups
   ├── RDS
   ├── ElastiCache
   ├── Amazon MQ
   ├── Elastic Beanstalk
   ├── CloudFront
   ├── ACM
   └── DNS
```

This would make infrastructure creation:

- Repeatable
- Version-controlled
- Reviewable
- Reproducible
- Easier to modify consistently

---

## 8. Configuration Management Limitation

The current project configures the application through its configuration file and AWS service configuration rather than implementing a dedicated configuration-management platform.

The project does not demonstrate a complete Ansible-based configuration-management workflow.

A future architecture could separate:

```text
Infrastructure provisioning
        ↓
Terraform
        ↓
Configuration management
        ↓
Ansible
```

This creates a clearer distinction between:

```text
Where infrastructure exists
```

and:

```text
How software is configured
```

Configuration management is therefore a logical future capability rather than a completed part of this project.

---

## 9. CI/CD Limitation

The application deployment flow is artifact-based.

The current model is:

```text
Application Source
       ↓
Maven Build
       ↓
WAR
       ↓
Elastic Beanstalk
       ↓
Deployment
       ↓
Validation
```

The project does not implement a complete automated CI/CD pipeline.

It does not currently demonstrate:

- Automated build triggering
- Automated unit testing
- Automated integration testing
- Artifact repository management
- Automated deployment promotion
- Automated rollback
- Deployment approval gates
- Security scanning
- Automated post-deployment validation

### Future improvement

A future delivery pipeline could become:

```text
Git Push
   ↓
Build
   ↓
Unit Tests
   ↓
Integration Tests
   ↓
Security Scan
   ↓
Package WAR
   ↓
Artifact Repository
   ↓
Deploy
   ↓
Validate
   ↓
Promote / Roll Back
```

Possible platforms could include Jenkins, GitHub Actions, or GitLab CI/CD.

The important future capability is the automated software-delivery process, not merely adding another tool.

---

## 10. Security Hardening Limitation

The project establishes network-level security using security groups and private backend access.

However, it does not establish a complete production security platform.

The CloudFront configuration used for the learning project does not enable WAF.

Current conceptual security model:

```text
Security Groups
      ↓
Network / port-level control
```

Future production-oriented model:

```text
Internet
    ↓
CloudFront
    ↓
WAF
    ↓
ALB
    ↓
Application
    ↓
Private Backend Services
```

Potential future security improvements include:

- AWS WAF
- Least-privilege IAM
- Secrets management
- Secure credential injection
- Stronger network segmentation
- More restrictive security-group rules
- Security scanning
- Centralized audit logging
- Automated security validation

These capabilities are future work unless independently implemented and evidenced.

---

## 11. Secrets Management Limitation

The application configuration requires credentials for services such as:

```text
RDS
Amazon MQ
```

The project demonstrates endpoint-driven application configuration, but it does not establish a complete production secrets-management architecture.

A production implementation should avoid treating credentials as ordinary application configuration.

A future design could use:

```text
AWS Secrets Manager
        ↓
Application configuration
        ↓
Runtime credential retrieval
```

The future objective would be:

```text
Credentials
     ↓
Secure storage
     ↓
Controlled access
     ↓
Runtime injection
```

rather than:

```text
Credentials
     ↓
Static configuration
     ↓
Application artifact
```

---

## 12. Observability Limitation

The project uses AWS health and monitoring capabilities, but it does not establish a complete enterprise observability platform.

Current validation primarily focuses on:

```text
Infrastructure state
        +
Application behavior
        +
Traffic-path evidence
```

It does not provide a comprehensive:

```text
Logs
Metrics
Traces
        ↓
Centralized Observability
        ↓
Dashboards
        ↓
Alerts
        ↓
Incident Response
```

### Future improvement

A production-oriented observability implementation could add:

- Centralized application logs
- Structured logging
- Metrics dashboards
- Application performance monitoring
- Distributed tracing
- Alerting
- Error-rate monitoring
- Latency monitoring
- Infrastructure dashboards

This would extend the project from:

```text
"Does the system work?"
```

toward:

```text
"How is the system behaving over time?"
```

---

## 13. Performance and Load-Testing Limitation

The project validates functional behavior.

It does not establish production performance characteristics.

The validation proves that the system can successfully execute the intended application flow, but it does not establish:

- Maximum throughput
- Maximum concurrent users
- Response-time targets
- Load-test results
- Stress-test results
- Saturation behavior
- Scaling thresholds
- Performance under failure

### Future improvement

A later performance-validation stage could introduce:

```text
Baseline
   ↓
Load Test
   ↓
Measure
   ↓
Identify Bottleneck
   ↓
Tune
   ↓
Retest
```

Potential measurements include:

- Requests per second
- Average latency
- P95/P99 latency
- CPU utilization
- Memory utilization
- Database latency
- Cache hit ratio
- Message-processing latency

These should only be claimed after being actually measured.

---

## 14. Failure-Injection Limitation

The project validates successful operation but does not comprehensively demonstrate failure injection or chaos testing.

For example, the project does not establish a complete test such as:

```text
Application Instance Failure
        ↓
Auto Scaling Detection
        ↓
Replacement
        ↓
Application Recovery
```

or:

```text
Backend Failure
        ↓
Application Behavior
        ↓
Detection
        ↓
Recovery
```

### Future improvement

Future validation could deliberately introduce controlled failures and measure recovery.

Examples:

```text
Terminate application instance
        ↓
Observe replacement
        ↓
Validate application

Broker failure
        ↓
Observe application behavior
        ↓
Validate recovery

Database failover
        ↓
Observe application behavior
        ↓
Validate recovery
```

This would turn availability claims into evidence-backed resilience claims.

---

## 15. Multi-Region Limitation

The application infrastructure is deployed in a single AWS region.

CloudFront provides a global distribution layer, but this should not be confused with multi-region application deployment.

The current model is:

```text
Global Users
     ↓
CloudFront
     ↓
Single AWS Region
     ↓
Application + Backend Services
```

This provides global edge distribution but does not provide:

```text
Region A
     +
Region B
     ↓
Application Failover
```

### Future improvement

A multi-region architecture could eventually look like:

```text
                 CloudFront
                     │
             ┌───────┴───────┐
             ▼               ▼
          Region A        Region B
             │               │
          ALB/App          ALB/App
             │               │
          Backend          Backend
             │               │
             └───────┬───────┘
                     ▼
              Global Routing
```

The exact implementation would depend on database replication, state management, DNS/routing, consistency, and disaster-recovery requirements.

---

## 16. Disaster-Recovery Limitation

The project does not establish a complete disaster-recovery strategy.

It demonstrates infrastructure creation and successful operation, but not a formal:

```text
Failure
   ↓
Recovery Plan
   ↓
Recovery Execution
   ↓
Recovery Measurement
```

It does not establish:

- Defined RTO
- Defined RPO
- Cross-region recovery
- Automated recovery orchestration
- Full disaster-recovery drills
- Documented recovery runbooks

### Future improvement

A later project iteration could define:

```text
Business Requirement
        ↓
RTO / RPO
        ↓
Backup Strategy
        ↓
Replication Strategy
        ↓
Recovery Automation
        ↓
DR Test
        ↓
Measured Recovery
```

---

## 17. CloudFront Configuration Limitation

CloudFront is successfully integrated as the public edge layer, but the project uses a relatively simple learning-oriented distribution configuration.

The configuration uses learning-oriented defaults and allows the application methods required by the dynamic VProfile workload.

This means the project does not establish a fully optimized production CDN configuration.

### Future improvement

A production-oriented CloudFront implementation could evaluate:

- Cache policies
- Origin request policies
- Cache-key design
- TTL tuning
- Compression
- Static/dynamic path separation
- HTTPS-only viewer policy
- WAF integration
- Origin protection
- Application-specific caching behavior

The goal would be to optimize the CDN based on actual application traffic rather than relying primarily on learning-project defaults.

---

## 18. Application-Configuration Limitation

The implementation uses managed-service endpoints to update `application.properties` before building the WAR.

The resulting relationship is:

```text
AWS Infrastructure
       ↓
Service Endpoints
       ↓
application.properties
       ↓
Maven Build
       ↓
WAR
       ↓
Deployment
```

This works for the learning project, but it creates a coupling between:

```text
Infrastructure endpoint values
        and
Application artifact
```

A future implementation could separate deployment artifacts from environment-specific configuration.

For example:

```text
Same Artifact
      ↓
Environment Configuration
      ├── Development
      ├── Test
      └── Production
```

This would make the deployment model more portable across environments.

---

## 19. Automated Testing Limitation

The validation strategy is primarily functional and infrastructure-oriented.

It does not establish a comprehensive automated test suite.

The project therefore does not claim:

- Complete unit-test coverage
- Automated integration-test coverage
- Automated regression testing
- Automated security testing
- Automated performance testing

### Future improvement

A stronger delivery lifecycle could become:

```text
Commit
  ↓
Unit Tests
  ↓
Integration Tests
  ↓
Security Tests
  ↓
Build
  ↓
Deploy
  ↓
Smoke Tests
  ↓
End-to-End Tests
```

This would connect software quality directly to the CI/CD workflow.

---

## 20. Operational Automation Limitation

The project reduces operational overhead by moving responsibilities to managed AWS services.

However, the project does not automate the complete infrastructure lifecycle through code.

The current model is substantially:

```text
AWS Console
     ↓
Configure
     ↓
Deploy
     ↓
Validate
     ↓
Clean Up
```

A future platform could evolve toward:

```text
Git
 ↓
Infrastructure Code
 ↓
Plan
 ↓
Review
 ↓
Apply
 ↓
Deploy
 ↓
Validate
 ↓
Destroy / Lifecycle Management
```

This would make the architecture itself a version-controlled engineering artifact.

---

## 21. Future Work Roadmap

The logical evolution from the current project is:

```text
CURRENT PROJECT
AWS Re-Architecture
        ↓
Infrastructure as Code
        ↓
Configuration Management
        ↓
CI/CD
        ↓
Automated Testing
        ↓
Observability
        ↓
Security Hardening
        ↓
High Availability
        ↓
Disaster Recovery
        ↓
Multi-Region Architecture
```

Each stage builds on the previous engineering capability.

---

## 22. Future Work — Infrastructure as Code

### Goal

Represent the AWS architecture declaratively.

```text
Terraform
    ↓
AWS Infrastructure
```

Potential scope:

- Security groups
- RDS
- ElastiCache
- Amazon MQ
- Elastic Beanstalk
- CloudFront
- ACM
- DNS

### Capability gained

```text
Manual Infrastructure
        ↓
Version-Controlled Infrastructure
```

---

## 23. Future Work — Configuration Management

### Goal

Separate infrastructure provisioning from software configuration.

```text
Terraform
    ↓
Infrastructure
    ↓
Ansible
    ↓
Configuration
```

Potential scope:

- Application configuration
- Runtime configuration
- Operational configuration
- Environment-specific settings

---

## 24. Future Work — CI/CD

### Goal

Automate software delivery.

```text
Git Push
   ↓
Build
   ↓
Test
   ↓
Package
   ↓
Deploy
   ↓
Validate
```

Potential capabilities:

- Automated Maven builds
- Automated tests
- Artifact management
- Elastic Beanstalk deployment
- Deployment gates
- Rollback
- Post-deployment validation

---

## 25. Future Work — Security

### Goal

Move from basic network-level security toward a broader application-security model.

Potential architecture:

```text
Internet
   ↓
CloudFront
   ↓
WAF
   ↓
ALB
   ↓
Application
   ↓
Private Backend Services
```

Potential improvements:

- WAF
- Secrets Manager
- Least-privilege IAM
- Automated security scanning
- Stronger network segmentation
- Secure configuration delivery
- Centralized audit logging

---

## 26. Future Work — Observability

### Goal

Move from validation to continuous operational visibility.

```text
Application
    │
    ├── Logs
    ├── Metrics
    └── Traces
            │
            ▼
      Observability
            │
            ├── Dashboards
            └── Alerts
```

Potential capabilities:

- Centralized logs
- Metrics
- Tracing
- Dashboards
- Alerts
- SLO/SLI monitoring
- Incident investigation

---

## 27. Future Work — High Availability

### Goal

Strengthen resilience across application and backend layers.

Potential improvements:

```text
Application
   ↓
Multiple Instances
   ↓
Load Balancer
```

and:

```text
Backend Services
   ↓
HA Configuration
   ↓
Failover
```

Areas for future work include:

- Application-instance redundancy
- Database high availability
- RabbitMQ broker redundancy
- Backend failure recovery
- Automated replacement
- Failure testing

The project already uses a load-balanced Beanstalk environment, but high availability should not be generalized to every backend service because the Amazon MQ deployment is explicitly single-instance.

---

## 28. Future Work — Disaster Recovery

### Goal

Prove that the system can recover from major infrastructure failure.

Future work could introduce:

```text
Backup
   ↓
Replication
   ↓
Recovery Automation
   ↓
DR Environment
   ↓
Recovery Test
```

The important transition is from:

```text
"We have backups"
```

to:

```text
"We have tested recovery and measured it."
```

---

## 29. Future Work — Multi-Region

### Goal

Provide application-level resilience beyond a single AWS region.

Potential architecture:

```text
                CloudFront
                    │
          ┌─────────┴─────────┐
          ▼                   ▼
      Region A             Region B
          │                   │
       ALB/App              ALB/App
          │                   │
       Backend              Backend
```

Potential engineering challenges would include:

- Database replication
- State management
- Global routing
- Failover
- Data consistency
- Recovery procedures
- Regional health detection

CloudFront alone does not provide this application-level multi-region failover.

---

## 30. Future Work — Performance Engineering

### Goal

Establish measurable performance characteristics.

Future workflow:

```text
Baseline
   ↓
Load Test
   ↓
Observe
   ↓
Identify Bottleneck
   ↓
Tune
   ↓
Retest
```

Potential areas:

- Application CPU
- Application memory
- Database performance
- Cache performance
- Message-broker performance
- ALB behavior
- CloudFront cache behavior
- Scaling response

---

## 31. Future Work — Automated Validation

The current validation process proves the architecture through functional checks and HTTP-level evidence.

A future implementation could automate those checks:

```text
Deployment
    ↓
Automated Smoke Test
    ↓
Login Test
    ↓
Backend Dependency Test
    ↓
CloudFront Verification
    ↓
Pass / Fail
```

This would allow validation to become a repeatable part of the delivery pipeline rather than a primarily manual activity.

---

## 32. Future Evolution Map

The current project can therefore be viewed as one stage in a larger DevOps progression:

```text
┌───────────────────────────────────┐
│ AWS Re-Architecture               │
│                                   │
│ RDS + ElastiCache + Amazon MQ     │
│ Elastic Beanstalk + CloudFront    │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ Infrastructure as Code            │
│                                   │
│ Terraform                         │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ Configuration Management          │
│                                   │
│ Ansible                           │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ CI/CD                             │
│                                   │
│ Build → Test → Deploy → Validate  │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ Observability + Security          │
│                                   │
│ Metrics + Logs + Traces + WAF     │
└────────────────┬──────────────────┘
                 │
                 ▼
┌───────────────────────────────────┐
│ Resilience                        │
│                                   │
│ HA + DR + Multi-Region            │
└───────────────────────────────────┘
```

---

## 33. What This Project Should Not Claim

Until separately implemented and supported by evidence, this repository should not claim:

- Development of the VProfile application
- Production-ready cloud architecture
- Complete Terraform/IaC implementation
- Complete Ansible configuration management
- Complete CI/CD implementation
- Production-grade WAF implementation
- Enterprise observability
- Production-grade high availability across every backend service
- Multi-region application failover
- Disaster-recovery certification
- Automated performance testing
- Chaos engineering
- Zero-downtime production guarantees
- Comprehensive security hardening

These are either outside the current project scope or represent future engineering capabilities.

---

## 34. Completed vs Future Capability Matrix

| Capability | Current Project | Future Work |
|---|---:|---:|
| AWS re-architecture | ✓ | |
| Amazon RDS | ✓ | |
| ElastiCache | ✓ | |
| Amazon MQ | ✓ | |
| Elastic Beanstalk | ✓ | |
| ALB | ✓ | |
| CloudFront | ✓ | |
| HTTPS / ACM | ✓ | |
| Application deployment | ✓ | |
| End-to-end validation | ✓ | |
| Terraform | | ✓ |
| Ansible | | ✓ |
| CI/CD | | ✓ |
| Automated testing | | ✓ |
| WAF | | ✓ |
| Secrets management | | ✓ |
| Enterprise observability | | ✓ |
| Production MQ HA | | ✓ |
| Database DR | | ✓ |
| Multi-region | | ✓ |
| Performance testing | | ✓ |
| Failure injection | | ✓ |
| Automated validation | | ✓ |

---

## 35. Final Perspective

The value of this project is not that it represents a complete production platform.

Its value is that it demonstrates a meaningful transition:

```text
Self-Managed Infrastructure
        ↓
AWS Managed Services
        ↓
Managed Application Platform
        ↓
Secure Service Connectivity
        ↓
Application Deployment
        ↓
CDN Integration
        ↓
End-to-End Validation
```

The project demonstrates the important engineering idea that cloud re-architecture is not simply moving an application to another location.

It changes the operational responsibility boundary:

```text
Before
Engineer
   ↓
Provision
Patch
Configure
Scale
Monitor
Maintain
```

toward:

```text
After
AWS Managed Services
   ↓
Provisioning abstraction
Patching abstraction
Operational abstraction
Scaling abstraction
Managed service lifecycle
```

The tradeoff is that the engineer gives up some low-level infrastructure control in exchange for reduced operational overhead. The project demonstrates that tradeoff through RDS, ElastiCache, Amazon MQ, and Elastic Beanstalk.

---

## 36. Final Mental Model

The entire limitations and future-work story can be remembered as:

```text
WHAT I BUILT
      ↓
AWS Re-Architecture
      ↓
WHAT I PROVED
      ↓
Deployment + Validation
      ↓
WHAT I DID NOT BUILD
      ↓
IaC + CI/CD + WAF + Observability
      ↓
WHAT COMES NEXT
      ↓
Automation
      ↓
Security
      ↓
Observability
      ↓
HA
      ↓
DR
      ↓
Multi-Region
```

The key principle is:

> **Do not hide the limitations of the current project. Use them to define exactly where the next stage of engineering begins.**

This keeps the repository technically honest while making the project's evolution easy to understand.

---

## Navigation

- [← Back to README](../README.md)
- [Architecture](architecture.md)
- [Implementation](implementation.md)
- [Validation](validation.md)
