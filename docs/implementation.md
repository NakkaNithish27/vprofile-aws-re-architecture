# VProfile — AWS Re-Architecture Implementation

[← Back to README](../README.md) | [Architecture](architecture.md)

---

## 1. Implementation Overview

This document records the implementation sequence used to re-architect and deploy the existing VProfile application on AWS.

The implementation follows a dependency-aware sequence rather than creating resources in an arbitrary order.

The complete execution flow is:

```text
1. AWS account / region
        ↓
2. Key pair
        ↓
3. Backend security group
        ↓
4. Amazon RDS
        ↓
5. Amazon ElastiCache
        ↓
6. Amazon MQ
        ↓
7. Elastic Beanstalk
        ↓
8. Backend security-group update
        ↓
9. RDS database initialization
        ↓
10. Beanstalk health check
        ↓
11. HTTPS listener
        ↓
12. Application configuration
        ↓
13. Maven build
        ↓
14. WAR deployment
        ↓
15. CloudFront
        ↓
16. DNS
        ↓
17. End-to-end validation
        ↓
18. Dependency-aware cleanup
```

The ordering is important because later resources depend on identifiers, endpoints, or infrastructure created earlier.

---

# 2. Implementation Principles

The implementation follows several recurring engineering patterns.

### Managed-Service Substitution

```text
MySQL on EC2      → Amazon RDS
Memcached on EC2  → Amazon ElastiCache
RabbitMQ on EC2   → Amazon MQ
```

### Platform Abstraction

```text
EC2 + ALB + ASG + S3 + CloudWatch
                 ↓
        Elastic Beanstalk
```

### Deferred Security Binding

```text
Backend SG
    ↓
Backend services
    ↓
Beanstalk created
    ↓
Beanstalk instance SG discovered
    ↓
Backend SG updated
```

### Jump-Box Pattern

```text
Temporary EC2
      ↓
MySQL client
      ↓
Private RDS
```

### Endpoint-Driven Configuration

```text
Managed services
      ↓
Collect endpoints
      ↓
Update application.properties
      ↓
Maven build
      ↓
WAR
      ↓
Beanstalk
```

### CDN Front-Loading

```text
User
  ↓
CloudFront
  ↓
Beanstalk ALB
  ↓
Application
```

---

# 3. AWS Region and Initial Preparation

All resources must be created in the same AWS region so that the services can communicate through the same VPC/security-group model.

The practical material uses **North Virginia (`us-east-1`)** as the working region.

The initial preparation consists of:

```text
AWS Console
    ↓
Select project region
    ↓
Create key pair
    ↓
Create backend security group
```

---

# 4. Key Pair

A key pair is created before the Beanstalk environment.

### Purpose

Elastic Beanstalk launches EC2 instances automatically, but an SSH key provides emergency/troubleshooting access to those instances when required.

The key pair is therefore an operational access mechanism rather than a component of the application architecture.

```text
Key Pair
   ↓
Elastic Beanstalk
   ↓
EC2 application instances
```

---

# 5. Backend Security Group

A single backend security group is created for the managed backend services.

The documented name is:

```text
vprofile-rearch-backend-SG
```

The backend security group is intended for:

```text
Amazon RDS
Amazon ElastiCache
Amazon MQ
```

At creation time, the Beanstalk application security group does not yet exist.

Therefore, the backend security group is created before Beanstalk and its application-tier security group.

---

## 5.1 Initial Security-Group State

The initial sequence is:

```text
Create backend SG
       ↓
Generate SG ID
       ↓
Add internal/backend rule
       ↓
Attach SG to managed backend services
```

The security group is created first so that its own ID can subsequently be referenced in a self-referencing rule.

---

## 5.2 Why the Beanstalk Rule Is Delayed

The Beanstalk instance security group does not exist until the Beanstalk environment is created.

Therefore this cannot be done initially:

```text
Backend SG
    ↓
Allow from Beanstalk Instance SG
```

because the Beanstalk SG ID does not yet exist.

The implementation therefore intentionally defers this rule until after Beanstalk creation.

---

# 6. Amazon RDS Implementation

Amazon RDS replaces the self-managed MySQL service previously running on EC2.

```text
Before:
MySQL on EC2

After:
Amazon RDS
```

The RDS instance is created independently from Elastic Beanstalk.

This is important because the database lifecycle remains separate from the application environment lifecycle.

The Beanstalk configuration explicitly does **not** create the database because RDS has already been created separately.

---

## 6.1 RDS Configuration

The implementation requires:

- MySQL engine
- Database instance configuration
- Storage configuration
- Database credentials
- Backend security group
- RDS endpoint

The RDS instance is associated with the backend security group.

Once the instance becomes available, its endpoint is collected for later application configuration.

---

## 6.2 RDS Readiness

The expected AWS state is:

```text
RDS
 ↓
Status: Available
 ↓
Endpoint available
```

RDS provisioning can take significant time, so the implementation creates the backend services early and allows them to provision while other infrastructure work continues.

---

# 7. Amazon ElastiCache Implementation

Amazon ElastiCache replaces the self-managed Memcached service.

```text
Before:
Memcached on EC2

After:
ElastiCache / Memcached
```

The implementation uses the existing backend security group.

The resulting dependency is:

```text
Beanstalk Application
        ↓
ElastiCache endpoint
        ↓
Memcached
```

The ElastiCache endpoint is collected after the cluster becomes available because it is required later in `application.properties`.

---

## 7.1 Cache Configuration

The project uses **Memcached**, matching the application's caching requirement.

The important implementation values are:

```text
Engine:     Memcached
Port:       11211
Network:    Backend security group
Endpoint:   AWS-generated endpoint
```

The application later uses the endpoint and port to connect to the cache.

---

# 8. Amazon MQ Implementation

The completed practical uses **Amazon MQ for RabbitMQ**.

```text
Before:
RabbitMQ on EC2

After:
Amazon MQ / RabbitMQ
```

The dedicated Amazon MQ material explicitly selects RabbitMQ rather than ActiveMQ.

---

## 8.1 Broker Configuration

The practical uses:

```text
Broker engine:       RabbitMQ
Deployment:          Single-instance
Engine version:      3.x / 3.13
Access:              Private
Security group:      Backend SG
Encryption:          Enabled
```

The single-instance deployment is explicitly a learning-project choice; a production workload could require a clustered/high-availability configuration.

---

## 8.2 Application Compatibility

The `awsrefactor` application branch expects:

```text
RabbitMQ 3.x
Encrypted connection
Private VPC connectivity
Matching broker credentials
```

The broker configuration therefore needs to remain compatible with the application branch.

A mismatch in engine version, credentials, access mode, or encryption behavior can result in runtime connection failures.

---

# 9. Elastic Beanstalk IAM Preparation

Elastic Beanstalk requires IAM roles for the environment and its EC2 instances.

The practical distinguishes between:

```text
Service Role
     ↓
Beanstalk management plane

Instance Profile
     ↓
Beanstalk EC2 instances
```

This is an important management-plane/data-plane distinction.

---

## 9.1 Instance Profile

The practical creates a custom instance profile role named:

```text
vprofile-Rearch-bean-role
```

The documented role configuration includes Beanstalk-related policies such as:

```text
AdministratorAccess-AWSElasticBeanstalk
AWSElasticBeanstalkCustomPlatformforEC2Role
AWSElasticBeanstalkRoleSNS
AWSElasticBeanstalkWebTier
```

These permissions are broader than a production least-privilege design because the project is a learning implementation.

---

## 9.2 Service Role

The Beanstalk service role controls what the Elastic Beanstalk service itself can perform on behalf of the environment.

The implementation therefore keeps the two concepts separate:

```text
Beanstalk Service Role
        │
        ▼
AWS management operations

EC2 Instance Profile
        │
        ▼
Running application instances
```

---

# 10. Elastic Beanstalk Environment

Elastic Beanstalk is used as the application-platform layer.

The environment automatically provisions and manages:

```text
EC2 instances
Application Load Balancer
Auto Scaling Group
Security Groups
S3 artifact storage
CloudWatch integration
```

This replaces the manual assembly of these application-tier components.

The environment is created before the application artifact is deployed; initially, a sample application is used to verify the environment itself.

---

# 11. Beanstalk Platform Configuration

The documented platform configuration is:

```text
Platform:       Tomcat
Tomcat:         10
Java runtime:   Corretto 21
Platform code:  Recommended/default version
```

Corretto is Amazon's distribution of OpenJDK.

The sample application is initially deployed so that the Beanstalk infrastructure can be validated independently from the VProfile artifact.

---

# 12. Beanstalk Environment Configuration

The environment uses a load-balanced configuration rather than a single-instance configuration.

The documented configuration includes:

```text
Preset:          Custom
Environment:     Load balanced
Instance type:   t2.micro
Minimum:         2
Maximum:         4
Public IP:       Enabled
Load balancer:   ALB
Stickiness:      Enabled
Deployment:      Rolling
Batch size:      50%
Monitoring:      5-minute interval
Health:          Enhanced
Volume:          GP3
```

These are the project configuration choices.

---

# 13. GP3 Volume Configuration

The Beanstalk environment uses a GP3 root volume rather than leaving the volume as the container default.

This was an operational compatibility workaround associated with the launch-template/launch-configuration behavior described in the practical.

The configuration therefore uses:

```text
Root volume
    ↓
General Purpose 3 (GP3)
```

---

# 14. Application Capacity

The environment is configured with:

```text
Minimum instances: 2
Maximum instances: 4
```

The practical initially results in two running Beanstalk instances.

Conceptually:

```text
                 ALB
                  │
          ┌───────┴───────┐
          ▼               ▼
      Instance 1      Instance 2
       Tomcat           Tomcat
```

The capacity can later scale up to four instances according to the configured scaling behavior.

---

# 15. Load Balancer Configuration

The Beanstalk environment uses an **Application Load Balancer**.

The implementation chooses ALB because the workload is an HTTP/web application.

The initial listener configuration includes HTTP on port 80.

Later, HTTPS is added on port 443 after the ACM certificate and load balancer are available.

The practical also enables load-balancer stickiness for the VProfile application because the application uses local session authentication behavior.

---

# 16. Backend Security-Group Update

After the Beanstalk environment is successfully created, its automatically generated security groups can be discovered.

The critical security group is the one attached to the **Beanstalk EC2 application instances**.

The load-balancer SG must not be used for the backend rule.

The implementation is:

```text
Beanstalk Instance SG
          │
          │ source
          ▼
Backend Security Group
          │
    ┌─────┼─────┐
    ▼     ▼     ▼
   RDS  Cache   MQ
```

The backend SG must reference the instance SG so that application instances can reach the backend services.

---

# 17. Backend Self-Reference

The backend security group also contains an internal/self-reference rule.

The resulting security relationship is:

```text
Application Instance SG
          │
          ▼
Backend SG
          │
          └── Backend SG
```

This supports the intended internal backend communication pattern.

The completed backend SG should therefore contain rules representing both:

```text
Application tier → Backend tier
Backend tier     → Backend tier
```

---

# 18. RDS Database Initialization

Creating RDS creates the database engine, but the VProfile application still needs its schema and data.

RDS does not provide SSH/OS access to the underlying database host.

The implementation therefore uses a temporary EC2 jump box.

```text
Temporary EC2
      │
      │ MySQL client
      ▼
Amazon RDS
      │
      ▼
accounts database
```

The temporary instance is not part of the final architecture.

---

## 18.1 Temporary Client Security Group

A temporary security group is used for the MySQL client instance.

The RDS inbound rule references the EC2 client's security group rather than an IP address.

Conceptually:

```text
vpro-mysql-client-sg
        │
        │ source
        ▼
RDS inbound :3306
```

This is more resilient than hardcoding the client's IP because the rule remains valid if the instance's IP changes, provided the same security group is used.

---

## 18.2 Install Required Tools

The temporary EC2 instance requires:

```text
MySQL client
Git
```

The MySQL client is used to connect to RDS.

Git is used to retrieve the VProfile repository containing the database initialization SQL file.

---

## 18.3 Test RDS Connectivity

The documented connection pattern is:

```bash
mysql -h <rds-endpoint> -u admin -p accounts
```

The password should be entered interactively rather than embedded directly in shell history.

A successful connection confirms:

```text
Correct endpoint
        +
Correct credentials
        +
Correct security-group path
        =
RDS connectivity
```

---

## 18.4 Retrieve the Application Source

The source repository is cloned:

```bash
git clone https://github.com/hkhcoder/vprofile-project.git
```

The appropriate project branch is then selected for the database/application configuration.

The `awsrefactor` branch is the relevant branch for the re-architected application configuration.

---

## 18.5 Load the Database Schema

The initialization SQL file is:

```text
db_backup.sql
```

It is executed against the RDS `accounts` database.

Conceptually:

```text
db_backup.sql
      │
      ▼
MySQL client
      │
      ▼
RDS / accounts
      │
      ▼
Tables + required data
```

---

## 18.6 Verify the Schema

After loading the SQL file:

```sql
show tables;
```

A populated table list confirms that the schema was successfully loaded.

---

## 18.7 Terminate the Jump Box

The temporary EC2 instance is terminated after database initialization.

Its lifecycle is:

```text
Launch
  ↓
Connect
  ↓
Initialize
  ↓
Verify
  ↓
Terminate
```

The instance exists only for the initialization task and is removed afterward.

---

# 19. Beanstalk Health Check

The default Beanstalk health-check path is changed to:

```text
/login
```

The reason is that `/login` is the actual VProfile application entry point.

The implementation problem without this change is:

```text
Beanstalk checks /
       ↓
Application does not return expected response
       ↓
Instance marked unhealthy
       ↓
Auto Scaling may replace instance
```

The corrected behavior is:

```text
Beanstalk checks /login
       ↓
VProfile responds correctly
       ↓
Instance considered healthy
```

The health-check customization is required before relying on Beanstalk health status for deployment/scaling decisions.

---

# 20. HTTPS Configuration

HTTPS is added to the Beanstalk load balancer using an ACM certificate.

The implementation chain is:

```text
ACM certificate
      ↓
Beanstalk configuration
      ↓
ALB listener
      ↓
HTTPS :443
```

The listener configuration uses:

```text
Protocol:    HTTPS
Port:        443
Certificate: ACM certificate
Policy:      2021-06
```

Both saving the configuration and applying the changes are required.

---

# 21. Application Configuration

At this stage, all managed backend service endpoints should be available.

The application configuration file is:

```text
src/main/resources/application.properties
```

The project uses the `awsrefactor` branch for the AWS deployment configuration.

---

## 21.1 Backend Endpoint Mapping

The local configuration uses placeholder hostnames.

These are replaced with AWS service endpoints:

```text
db01
  ↓
RDS endpoint

rmq01
  ↓
Amazon MQ hostname

mc01
  ↓
ElastiCache endpoint
```

Endpoint replacement is a mandatory step before building the artifact.

---

## 21.2 Port Mapping

The important port mappings are:

| Service | Application Port |
|---|---:|
| MySQL / RDS | `3306` |
| Memcached / ElastiCache | `11211` |
| RabbitMQ / Amazon MQ | `5671` |

The RabbitMQ port is an important compatibility detail.

The local RabbitMQ configuration uses:

```text
5672
```

whereas the encrypted Amazon MQ connection uses:

```text
5671
```

Failing to change this value can result in a connection failure even when the hostname and credentials are correct.

---

## 21.3 Credentials

The database and message-broker credentials must also match the credentials configured for the AWS managed services.

The application configuration therefore represents:

```text
RDS
 ├── hostname
 ├── port
 ├── username
 └── password

ElastiCache
 ├── hostname
 └── port

Amazon MQ
 ├── hostname
 ├── port
 ├── username
 └── password
```

Secrets must **not** be committed to the public GitHub repository.

---

# 22. Maven Build

The application artifact is built locally after the AWS endpoint configuration is complete.

The documented build prerequisites are:

```text
Maven 3.9.9
Java 17+
```

The build command is:

```bash
mvn install
```

The resulting artifact is:

```text
target/vprofile-v2.war
```

The critical dependency is:

```text
Backend endpoints
       ↓
application.properties
       ↓
Maven build
       ↓
vprofile-v2.war
```

If the endpoints are wrong when the artifact is built, the deployed application will inherit those incorrect connection details.

---

# 23. Artifact Deployment

The WAR is deployed through the Elastic Beanstalk console.

The deployment mechanism is:

```text
Beanstalk Console
       ↓
Upload and deploy
       ↓
Select vprofile-v2.war
       ↓
Version label
       ↓
Deploy
```

Beanstalk then handles distribution of the artifact to the application instances.

This removes the need for:

```text
SSH
+
manual WAR copying
+
manual Tomcat restart
+
per-instance deployment
```

---

# 24. Rolling Deployment

The project uses:

```text
Deployment policy: Rolling
Batch size:        50%
```

With two initial instances:

```text
Instance 1
Instance 2
```

the effective deployment batch is:

```text
50% of 2 = 1 instance
```

The conceptual sequence is:

```text
Initial:
Instance 1 → Healthy
Instance 2 → Healthy

Deploy batch 1:
Instance 1 → Draining / deploying
Instance 2 → Healthy

Instance 1 → Healthy

Deploy batch 2:
Instance 2 → Draining / deploying

Instance 2 → Healthy

Deployment complete
```

The deployment states can be monitored through Beanstalk Events and target-group health.

This should be described as the **configured rolling deployment strategy**, rather than as an unconditional production zero-downtime guarantee.

---

# 25. Custom Domain

A custom domain is mapped to the application.

The conceptual mapping is:

```text
Custom domain
      ↓
DNS record
      ↓
AWS application endpoint
```

The final project architecture places CloudFront between the public DNS entry and the Beanstalk origin.

---

# 26. CloudFront Implementation

CloudFront is introduced after the application is running through the Beanstalk load balancer.

The origin is:

```text
Beanstalk Load Balancer
```

The final request path becomes:

```text
User
  ↓
DNS
  ↓
CloudFront
  ↓
Beanstalk ALB
  ↓
Tomcat
```

The CloudFront distribution provides the public CDN layer and its own `*.cloudfront.net` endpoint.

---

## 26.1 CloudFront Origin

The Beanstalk load balancer is configured as the CloudFront origin.

Conceptually:

```text
CloudFront Distribution
          │
          │ origin
          ▼
Beanstalk ALB
```

This preserves the Beanstalk environment as the application origin while adding CloudFront as the public edge layer.

---

## 26.2 SSL

The public HTTPS architecture is:

```text
User
 │ HTTPS
 ▼
CloudFront
 │ HTTPS
 ▼
Beanstalk ALB :443
 │
 ▼
Tomcat
```

---

# 27. DNS Update

After CloudFront is created, the public DNS record is updated to point the application domain toward the CloudFront distribution.

Conceptually:

```text
vprofile.example.com
        ↓
CloudFront
        ↓
Beanstalk ALB
```

This allows users to use the application domain rather than the AWS-generated Beanstalk or CloudFront endpoint.

---

# 28. End-to-End Implementation State

At this point the deployed system should resemble:

```text
                         USER
                           │
                           │ HTTPS
                           ▼
                         DNS
                           │
                           ▼
                     CLOUDFRONT
                           │
                           ▼
                  BEANSTALK ALB
                      HTTPS :443
                           │
                           ▼
                 TOMCAT / EC2
                  ┌────────┴────────┐
                  │                 │
              Instance 1        Instance 2
                  │                 │
                  └────────┬────────┘
                           │
                 Backend security group
                           │
            ┌──────────────┼──────────────┐
            ▼              ▼              ▼
         Amazon RDS   ElastiCache      Amazon MQ
           MySQL        Memcached       RabbitMQ
```

The application instances are managed by Elastic Beanstalk, while the backend services remain independently managed AWS services.

---

# 29. Application Configuration → Infrastructure Dependency

One of the most important implementation relationships is:

```text
Infrastructure first
        ↓
Service endpoints exist
        ↓
Application configuration updated
        ↓
Artifact rebuilt
        ↓
Artifact deployed
```

This means the application artifact cannot be considered independent of the infrastructure configuration in this implementation.

The configuration is baked into the artifact during the Maven build.

Therefore:

```text
Change endpoint
      ↓
Change application.properties
      ↓
Rebuild WAR
      ↓
Redeploy WAR
```

A deployment without rebuilding after changing these values will not contain the updated configuration.

---

# 30. Troubleshooting Pattern

The project uses a repeatable:

```text
Fix
 ↓
Rebuild
 ↓
Redeploy
 ↓
Validate
```

cycle.

For example:

```text
Application cannot connect to backend
            ↓
Check application.properties
            ↓
Verify endpoint / port / credentials
            ↓
Correct configuration
            ↓
mvn install
            ↓
Upload and deploy
            ↓
Validate application
```

This is a reusable troubleshooting pattern for artifact-based deployments.

---

# 31. Implementation Verification Points

The implementation contains checkpoints at each major layer.

| Stage | Verification |
|---|---|
| Backend SG | Rules exist as expected |
| RDS | Status = Available; endpoint exists |
| ElastiCache | Cluster available; endpoint exists |
| Amazon MQ | Broker available; connection endpoint exists |
| Beanstalk | Environment healthy |
| Instances | Expected instances running |
| ALB | Load balancer exists |
| Security | Backend SG references Beanstalk instance SG |
| RDS initialization | `show tables;` returns application tables |
| Health check | `/login` returns expected application response |
| HTTPS | ALB has HTTPS listener on `443` |
| Build | `target/vprofile-v2.war` exists |
| Deployment | Beanstalk deployment completes successfully |
| Application | Login page loads |
| Database | Login/data operations work |
| Cache | Memcached verification succeeds |
| CloudFront | Request path can be proven through response headers |
| DNS | Public domain resolves to intended endpoint |

---

# 32. Cleanup Implementation

The project also documents dependency-aware cleanup.

Cleanup cannot simply be performed in reverse creation order because some AWS resources have dependency relationships.

The documented cleanup sequence is:

```text
1. Disable CloudFront
        ↓
2. Delete RDS
        ↓
3. Delete ElastiCache
        ↓
4. Delete Amazon MQ
        ↓
5. Remove backend SG cross-reference
        ↓
6. Terminate Elastic Beanstalk
        ↓
7. Delete CloudFront
        ↓
8. Remove DNS record
        ↓
9. Delete backend security group
```

CloudFront must first be disabled before it can be deleted.

The backend security-group reference to the Beanstalk instance SG must be removed before Beanstalk termination because that external dependency can prevent clean deletion.

---

# 33. Final Resource State

The expected post-cleanup state is:

```text
CloudFront        → deleted
RDS               → deleted
ElastiCache       → deleted
Amazon MQ         → deleted
Beanstalk         → terminated
DNS record        → removed
Backend SG        → deleted
```

The cleanup procedure is therefore part of the implementation rather than an afterthought.

---

# 34. Complete Implementation Sequence

The entire implementation can be compressed into:

```text
AWS
 │
 ├── Key Pair
 │
 ├── Backend SG
 │      │
 │      ├── RDS
 │      ├── ElastiCache
 │      └── Amazon MQ
 │
 ├── IAM Roles
 │
 └── Elastic Beanstalk
        │
        ├── EC2
        ├── ALB
        ├── ASG
        ├── S3
        └── CloudWatch

        ↓

Backend SG Update
        ↓
RDS Initialization
        ↓
Health Check → /login
        ↓
HTTPS :443
        ↓
Collect AWS endpoints
        ↓
application.properties
        ↓
Maven
        ↓
vprofile-v2.war
        ↓
Beanstalk Upload & Deploy
        ↓
Rolling 50%
        ↓
CloudFront
        ↓
DNS
        ↓
End-to-End Validation
        ↓
Dependency-Aware Cleanup
```

---

# 35. Implementation Mental Model

The implementation can be remembered as:

```text
BUILD THE FOUNDATION
        ↓
Security Group
        ↓
Managed Backend Services
        ↓
BUILD THE PLATFORM
        ↓
Elastic Beanstalk
        ↓
CONNECT THE TIERS
        ↓
Backend SG ← Beanstalk Instance SG
        ↓
INITIALIZE STATE
        ↓
RDS schema + data
        ↓
MAKE THE PLATFORM APPLICATION-AWARE
        ↓
Health check /login
        ↓
HTTPS :443
        ↓
BUILD THE APPLICATION
        ↓
AWS endpoints
        ↓
application.properties
        ↓
Maven → WAR
        ↓
DEPLOY
        ↓
Rolling deployment
        ↓
PUT THE CDN IN FRONT
        ↓
CloudFront
        ↓
DNS
        ↓
PROVE THE SYSTEM
        ↓
End-to-end validation
        ↓
CLEAN UP
```

---

# 36. Implementation Patterns Demonstrated

| Pattern | Implementation |
|---|---|
| Managed-service substitution | RDS, ElastiCache, Amazon MQ |
| Platform abstraction | Elastic Beanstalk |
| Management/data-plane separation | Beanstalk service role vs instance profile |
| Deferred security binding | Backend SG updated after Beanstalk creation |
| Jump box | Temporary EC2 for RDS initialization |
| Endpoint-driven configuration | AWS endpoints injected before Maven build |
| Health-check alignment | `/login` |
| Progressive deployment | Rolling, 50% |
| CDN front-loading | CloudFront before ALB |
| Sample-first platform validation | Beanstalk sample application before real WAR |
| Decoupled stateful service | RDS lifecycle independent of Beanstalk |
| Fix-rebuild-redeploy | Configuration correction followed by new WAR deployment |
| Dependency-aware cleanup | Remove references before terminating dependent infrastructure |

---

## Navigation

- [← Back to README](../README.md)
- [Architecture](architecture.md)
- [Validation](validation.md)
- [Limitations & Future Work](limitations-and-future-work.md)
