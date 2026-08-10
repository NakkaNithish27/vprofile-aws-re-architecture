# VProfile — AWS Re-Architecture Validation

[← Back to README](../README.md) | [Architecture](architecture.md) | [Implementation](implementation.md)

---

## 1. Validation Overview

Validation is the final engineering stage of the VProfile AWS re-architecture.

The objective is not simply to confirm that AWS resources show a **running**, **available**, or **healthy** state.

The objective is to prove that:

1. The application is reachable.
2. The application tier is healthy.
3. The application can communicate with its managed backend services.
4. HTTPS is functioning.
5. The public DNS path reaches CloudFront.
6. CloudFront is actually participating in the request path.
7. The complete architecture works end-to-end.

The core principle is:

> **Do not validate infrastructure only by looking at resource status. Prove that the application can traverse the architecture and successfully use its dependencies.**

---

# 2. Validation Strategy

Validation follows the architecture from the outside inward:

```text
PUBLIC ACCESS
      ↓
DNS
      ↓
CloudFront
      ↓
HTTPS
      ↓
Application Load Balancer
      ↓
Elastic Beanstalk
      ↓
Tomcat / EC2
      ↓
Managed Backend Services
      ├── RDS
      ├── ElastiCache
      └── Amazon MQ
```

The validation strategy therefore has two dimensions:

### Functional validation

Does the application actually work?

### Infrastructure-path validation

Can we prove that requests are passing through the intended infrastructure layers?

This distinction is particularly important for CloudFront because the application may look exactly the same whether the browser reaches the ALB directly or reaches the ALB through CloudFront.

---

# 3. Final Validation Model

The complete validation model is:

```text
                 USER
                   │
                   ▼
             Public URL
                   │
                   ▼
                  DNS
                   │
                   ▼
             CloudFront
                   │
              Via header
                   │
                   ▼
          Beanstalk ALB
                   │
                   ▼
        Tomcat / EC2 Instance
                   │
        ┌──────────┼──────────┐
        ▼          ▼          ▼
       RDS      ElastiCache  Amazon MQ
      MySQL      Memcached    RabbitMQ
```

Each layer needs a corresponding validation signal.

| Layer | What must be proven |
|---|---|
| DNS | Public hostname resolves to the intended CloudFront entry point |
| CloudFront | Request actually passes through CloudFront |
| HTTPS | Public application is reachable securely |
| ALB | Application requests reach the Beanstalk origin |
| Beanstalk | Environment and application instances are healthy |
| Tomcat | Application responds correctly |
| RDS | Application can authenticate/use database-backed functionality |
| ElastiCache | Application can use the managed cache |
| Amazon MQ | Application can communicate with RabbitMQ |
| End-to-end | The complete request/dependency chain works |

---

# 4. Pre-Validation Infrastructure Checks

Before testing the application, confirm that the underlying environment is in the expected state.

### Expected state

```text
Elastic Beanstalk
    ↓
Healthy

Application instances
    ↓
Healthy

ALB
    ↓
Available

RDS
    ↓
Available

ElastiCache
    ↓
Available

Amazon MQ
    ↓
Available

CloudFront
    ↓
Enabled / Available

DNS
    ↓
Public record configured
```

These checks do not constitute the final validation.

They only establish that the infrastructure is ready for functional testing.

---

# 5. Elastic Beanstalk Health Validation

Elastic Beanstalk must report a healthy environment before application-level validation proceeds.

The configured application health-check path is:

```text
/login
```

This is important because the VProfile application uses `/login` as its actual application entry point.

### Validation

Confirm:

```text
Environment
    ↓
Healthy

Instances
    ↓
Healthy

Health check
    ↓
/login
```

### Expected result

The environment should remain healthy and the application instances should pass the configured health check.

---

# 6. Application Availability Validation

The first functional test is reaching the application through its public URL.

The intended request path is:

```text
Browser
   ↓
Public DNS
   ↓
CloudFront
   ↓
Beanstalk ALB
   ↓
Tomcat
   ↓
VProfile
```

The application should display the VProfile login page.

### Expected result

```text
VProfile login page
        ↓
Successfully rendered
```

This proves basic application availability, but it does **not yet prove** that every backend dependency or CloudFront layer has been correctly validated.

---

# 7. HTTPS Validation

The public application is intended to be accessed through HTTPS.

The expected path is:

```text
Browser
   │ HTTPS
   ▼
CloudFront
   │ HTTPS
   ▼
Beanstalk ALB :443
   │
   ▼
Application
```

### Validation checks

Confirm:

- Browser uses `https://`.
- No certificate warning is displayed.
- The public domain matches the configured certificate.
- The Beanstalk ALB has the HTTPS listener.
- CloudFront can communicate with the configured origin.

### Expected result

```text
HTTPS URL
    ↓
Application loads successfully
```

---

# 8. DNS Validation

The public domain should resolve to the CloudFront distribution rather than directly exposing the Beanstalk load balancer.

The intended resolution path is:

```text
Application domain
       ↓
CloudFront distribution
       ↓
Nearest CloudFront edge
       ↓
ALB origin
```

### Validation objective

Prove that the public URL is using the intended CloudFront entry point.

### Failure indicators

If the application is accessible but CloudFront validation fails, possible causes include:

- DNS still points directly to the ALB.
- DNS propagation has not completed.
- The old load-balancer record still exists.
- The CloudFront distribution is not yet fully available.
- The wrong CloudFront distribution is being referenced.

---

# 9. CloudFront Validation

CloudFront is the most important infrastructure-path validation in the final stage.

The application can look identical whether traffic comes from:

```text
Browser → ALB
```

or:

```text
Browser → CloudFront → ALB
```

Therefore, simply seeing the application load does not prove CloudFront participation.

The validation must inspect the HTTP response metadata.

---

## 9.1 Why the Browser Must Be Clean

Use one of:

- Private/incognito browser
- Different browser
- Cleared browser cache

The reason is to avoid relying on previously cached content.

---

## 9.2 Open Developer Tools

Open browser developer tools:

```text
F12
  ↓
Developer Tools
  ↓
Network / Console
```

Then access the HTTPS application URL.

---

## 9.3 Find the GET Request

Locate the GET request corresponding to the application page.

Conceptually:

```text
Network
   ↓
GET request
   ↓
Headers
   ↓
Response headers
```

---

## 9.4 Inspect the `Via` Header

Look through the response headers for:

```text
Via
```

The expected value contains:

```text
*.cloudfront.net
```

### Expected evidence

```text
Via: <CloudFront-related value ending in .cloudfront.net>
```

### What this proves

```text
Browser
   ↓
CloudFront
   ↓
ALB
   ↓
Application Instance
```

Therefore the CloudFront layer is not merely configured — it is demonstrably participating in the request path.

---

# 10. CloudFront Validation Evidence

The highest-signal CloudFront evidence is:

```text
Browser Developer Tools
        ↓
GET request
        ↓
Response Headers
        ↓
Via
        ↓
*.cloudfront.net
```

This is stronger than a screenshot showing the CloudFront distribution exists.

Why?

Because:

```text
CloudFront exists
       ≠
Traffic is using CloudFront
```

The `Via` header provides evidence of the latter.

This is a reusable infrastructure-validation principle:

> **When an intermediary is introduced into a traffic path, validate the intermediary by inspecting evidence that it actually handled the request.**

---

# 11. CloudFront Failure Diagnosis

If the expected `Via` evidence is missing, investigate the traffic path rather than immediately assuming the application is broken.

Possible causes include:

```text
No Via header
     │
     ├── CloudFront origin incorrectly configured
     │
     ├── DNS not pointing to CloudFront
     │
     ├── Browser accessing ALB directly
     │
     └── DNS propagation / transition issue
```

The first diagnostic question should therefore be:

> **Am I actually accessing the application through the public CloudFront-backed domain?**

---

# 12. Complete Traffic-Flow Validation

The intended request lifecycle is:

```text
1. User enters HTTPS URL
             ↓
2. DNS resolves the public domain
             ↓
3. Request reaches CloudFront
             ↓
4. CloudFront checks its cache
             ↓
5. Cache hit → content can be served at edge
             │
             └── Cache miss → request forwarded to ALB
                                      ↓
                                 Healthy instance
                                      ↓
                                   Tomcat
                                      ↓
                                  Application
                                      ↓
                                 Response
                                      ↓
                             CloudFront response
```

---

# 13. Cache-Behavior Validation

CloudFront should not be interpreted as caching every piece of application behavior.

The architecture distinguishes between cacheable/static content and dynamic/user-specific content.

Conceptually:

```text
Static / cacheable content
        ↓
CloudFront edge cache
        ↓
Fast edge response
```

while:

```text
Dynamic / non-cacheable content
        ↓
CloudFront
        ↓
Origin / ALB
        ↓
Application
```

Therefore, the presence of CloudFront should be validated through traffic-path evidence rather than assuming every request is served from cache.

---

# 14. Application-to-RDS Validation

RDS replaces the MySQL service that previously ran on EC2.

The application must therefore successfully communicate with:

```text
Tomcat
   ↓
RDS endpoint
   ↓
MySQL
```

The primary application-level proof is successful functionality that depends on the database.

### Validation model

```text
User
 ↓
Application
 ↓
Database lookup
 ↓
RDS
 ↓
Result
 ↓
Application response
```

### Expected result

The login operation succeeds and the application can retrieve the expected application data.

This demonstrates:

```text
Application → RDS
```

is functioning.

---

# 15. RDS Validation Boundary

RDS status alone is not sufficient.

These are different claims:

```text
RDS status = Available
```

and:

```text
Application successfully uses RDS
```

The second is the meaningful application-level validation.

The database initialization should also be validated separately using the database schema, but the final end-to-end validation should prove that the deployed application can actually use the database.

---

# 16. Amazon MQ Validation

The implemented architecture uses:

```text
Amazon MQ
    ↓
RabbitMQ
```

The application must be able to communicate with the private broker.

The expected dependency is:

```text
Tomcat
   ↓
Amazon MQ endpoint
   ↓
RabbitMQ
```

### Validation objective

Prove that the application can successfully perform the messaging-related operation that depends on RabbitMQ.

### Expected result

The application-level messaging test succeeds.

This demonstrates:

```text
Application
    ↓
Network/security configuration
    ↓
Amazon MQ
    ↓
RabbitMQ
```

rather than merely confirming that the broker resource exists.

---

# 17. ElastiCache Validation

The application uses:

```text
ElastiCache
    ↓
Memcached
```

The dependency is:

```text
Tomcat
   ↓
ElastiCache endpoint
   ↓
Memcached
```

### Validation objective

Confirm that the application's cache functionality works through the managed ElastiCache service.

---

# 18. End-to-End Application Validation

The final functional validation combines the application and its backend dependencies.

The expected validation sequence is:

```text
Public URL
    ↓
Login page loads
    ↓
Login succeeds
    ↓
Database-backed functionality works
    ↓
Messaging functionality works
    ↓
Cache functionality works
```

At the infrastructure level:

```text
Public URL
    ↓
DNS
    ↓
CloudFront
    ↓
ALB
    ↓
Beanstalk / Tomcat
    ├── RDS
    ├── ElastiCache
    └── Amazon MQ
```

The objective is to validate the complete refactored stack through the public URL and verify application, database, cache, and messaging functionality.

---

# 19. Validation Matrix

| Validation Area | Test | Expected Evidence |
|---|---|---|
| Beanstalk | Environment health | Healthy environment |
| Application | Open public URL | VProfile login page |
| HTTPS | Open HTTPS URL | Valid HTTPS connection |
| DNS | Resolve public domain | Domain reaches intended public endpoint |
| CloudFront | Inspect response headers | `Via` contains `.cloudfront.net` |
| ALB/Application | Application loads | Successful application response |
| RDS | Login/data operation | Database-backed functionality succeeds |
| Amazon MQ | Messaging operation | RabbitMQ functionality succeeds |
| ElastiCache | Cache operation | Cache functionality succeeds |
| End-to-end | Complete user flow | All layers operate together |

---

# 20. Evidence Strategy

The repository should contain **high-signal evidence**, not screenshots of every action.

Recommended evidence categories:

```text
evidence/
└── screenshots/
    ├── architecture/
    ├── infrastructure/
    ├── deployment/
    └── validation/
```

---

## 20.1 Architecture Evidence

Useful evidence may include:

- Final AWS architecture view
- CloudFront distribution configuration
- Beanstalk environment configuration
- Backend managed-service configuration

The evidence should support architectural claims rather than duplicate the documentation.

---

## 20.2 Infrastructure Evidence

Useful evidence may include:

- RDS available state
- ElastiCache available state
- Amazon MQ available state
- Beanstalk healthy state
- Backend security-group rules

---

## 20.3 Deployment Evidence

Useful evidence may include:

- Beanstalk deployment event
- Uploaded WAR/version
- Rolling deployment progress
- Healthy application instances after deployment

---

## 20.4 Validation Evidence

The strongest validation evidence includes:

### Application

```text
VProfile login page
```

### Successful application operation

```text
Successful login / application dashboard
```

### CloudFront

```text
Developer Tools
    ↓
GET request
    ↓
Response Headers
    ↓
Via: *.cloudfront.net
```

### Backend functionality

Evidence showing successful application interaction with:

```text
RDS
Amazon MQ
ElastiCache
```

---

# 21. Validation Evidence vs Infrastructure Status

A useful distinction is:

| Evidence type | What it proves |
|---|---|
| RDS = Available | RDS resource is operational |
| MQ = Available | Broker resource is operational |
| ElastiCache = Available | Cache resource is operational |
| Beanstalk = Healthy | Application platform considers environment healthy |
| Login succeeds | Application can successfully use required database-backed functionality |
| Messaging test succeeds | Application can communicate with RabbitMQ |
| Cache test succeeds | Application can communicate with Memcached |
| `Via: *.cloudfront.net` | Request passed through CloudFront |

The strongest validation combines both infrastructure state and application behavior.

---

# 22. Failure Isolation Model

Validation should also help isolate failures.

A useful diagnostic flow is:

```text
Public page fails
      ↓
Check DNS / CloudFront / ALB / Beanstalk
```

If:

```text
Page loads
```

but:

```text
Login fails
```

then investigate:

```text
RDS
application database configuration
credentials
security-group connectivity
```

If:

```text
Login works
```

but:

```text
Messaging fails
```

then investigate:

```text
Amazon MQ
RabbitMQ configuration
credentials
port
security-group connectivity
```

If:

```text
Messaging works
```

but:

```text
Cache functionality fails
```

then investigate:

```text
ElastiCache
Memcached endpoint
port
security-group connectivity
```

This creates a dependency-aware troubleshooting sequence rather than debugging the entire architecture simultaneously.

---

# 23. Failure-Signature Matrix

| Symptom | Primary Investigation |
|---|---|
| Public URL does not resolve | DNS |
| HTTPS certificate error | ACM certificate / CloudFront domain association |
| Application page unavailable | CloudFront → ALB → Beanstalk |
| Beanstalk unhealthy | `/login` health check / application instance |
| Login fails | RDS endpoint / credentials / SG |
| Messaging fails | Amazon MQ / RabbitMQ endpoint / port / credentials / SG |
| Cache fails | ElastiCache endpoint / port / SG |
| CloudFront `Via` missing | DNS / CloudFront origin / direct ALB access |
| Old application path still appears | DNS propagation / old record / browser cache |
| CloudFront origin fails | Incorrect ALB or region/origin configuration |

---

# 24. Deployment Validation

The deployment strategy is configured as:

```text
Rolling
50% batch size
```

With two application instances:

```text
2 instances
   ↓
50% batch
   ↓
1 instance at a time
```

The documented deployment flow is:

```text
Instance 1
    ↓
Deploy
    ↓
Draining / unhealthy during update
    ↓
Healthy
    ↓
Proceed to Instance 2
    ↓
Deploy
    ↓
Healthy
```

Beanstalk Events and target-group health provide observable evidence of the deployment progression.

---

# 25. Deployment Acceptance Criteria

The deployment can be considered successfully validated when:

```text
✓ Beanstalk environment healthy
✓ Expected application instances healthy
✓ Deployment completed
✓ Application login page loads
✓ Login succeeds
✓ Database-backed functionality works
✓ Messaging functionality works
✓ Cache functionality works
✓ HTTPS works
✓ Public DNS works
✓ CloudFront `Via` evidence is present
```

---

# 26. Complete Validation Procedure

The entire validation process can be compressed into:

```text
1. Confirm infrastructure is ready
        ↓
2. Confirm Beanstalk is healthy
        ↓
3. Open public HTTPS URL
        ↓
4. Confirm login page loads
        ↓
5. Login successfully
        ↓
6. Confirm database-backed functionality
        ↓
7. Confirm messaging functionality
        ↓
8. Confirm cache functionality
        ↓
9. Open Developer Tools
        ↓
10. Inspect GET response headers
        ↓
11. Find Via header
        ↓
12. Confirm *.cloudfront.net
        ↓
13. Record high-signal evidence
```

---

# 27. Final Validation Checklist

```text
PUBLIC ACCESS
□ Public domain resolves
□ HTTPS URL loads
□ VProfile login page appears

APPLICATION
□ Beanstalk environment healthy
□ Application instances healthy
□ Login succeeds

DATABASE
□ RDS is available
□ Application can use RDS-backed functionality

MESSAGING
□ Amazon MQ is available
□ Application messaging test succeeds

CACHE
□ ElastiCache is available
□ Application cache test succeeds

CLOUDFRONT
□ Distribution is available
□ Public DNS points to CloudFront
□ Developer Tools show Via header
□ Via contains *.cloudfront.net

DEPLOYMENT
□ WAR deployment completed
□ Rolling deployment completed
□ Instances return to healthy state

EVIDENCE
□ Application evidence captured
□ Backend validation evidence captured
□ Beanstalk/deployment evidence captured
□ CloudFront header evidence captured
```

---

# 28. Final Validation Result

The architecture is considered **end-to-end validated** when the following chain is proven:

```text
                         USER
                           │
                           ▼
                    PUBLIC HTTPS URL
                           │
                           ▼
                          DNS
                           │
                           ▼
                     CLOUDFRONT
                           │
                    Via: *.cloudfront.net
                           │
                           ▼
                    BEANSTALK ALB
                           │
                           ▼
                    TOMCAT / EC2
                           │
             ┌─────────────┼─────────────┐
             ▼             ▼             ▼
            RDS        ELASTICACHE     AMAZON MQ
           MySQL        Memcached       RabbitMQ
             │             │             │
             └─────────────┼─────────────┘
                           │
                           ▼
                 WORKING VPROFILE APP
```

The important distinction is:

```text
Infrastructure exists
        ↓
Infrastructure is healthy
        ↓
Application works
        ↓
Backend dependencies work
        ↓
Traffic path is proven
        ↓
ARCHITECTURE VALIDATED
```

---

# 29. Validation as an Engineering Skill

The final validation stage demonstrates an important DevOps principle:

> **A system should be validated through observable evidence, not assumptions.**

For this project:

```text
CloudFront exists
        ↓
not enough

CloudFront is configured as origin
        ↓
not enough

DNS points to CloudFront
        ↓
stronger

HTTP response contains CloudFront evidence
        ↓
traffic path proven
```

Likewise:

```text
RDS is Available
        ↓
not enough

Application successfully uses RDS
        ↓
dependency proven
```

And:

```text
Beanstalk is Healthy
        ↓
not enough

Application successfully serves users
        ↓
application behavior proven
```

This distinction between **resource state** and **system behavior** is one of the most reusable lessons from the project.

---

# 30. Validation Mental Model

The entire validation process can be remembered as:

```text
PROVE THE OUTSIDE
        ↓
Public URL
        ↓
DNS
        ↓
CloudFront
        ↓
Via header
        ↓
PROVE THE APPLICATION
        ↓
ALB
        ↓
Beanstalk
        ↓
Tomcat
        ↓
PROVE THE DEPENDENCIES
        ↓
RDS
        ↓
Amazon MQ
        ↓
ElastiCache
        ↓
PROVE THE WHOLE PATH
        ↓
End-to-End User Flow
        ↓
ARCHITECTURE VALIDATED
```

Or, more simply:

```text
URL
 ↓
CloudFront
 ↓
ALB
 ↓
Beanstalk
 ↓
Application
 ├── RDS
 ├── Amazon MQ
 └── ElastiCache
 ↓
SUCCESS
```

---

# 31. Validation → Documentation → Cleanup

The project completion sequence is:

```text
BUILD
  ↓
VALIDATE
  ↓
DOCUMENT
  ↓
CLEAN UP
```

Validation proves that the architecture works before the environment is destroyed.

The resulting evidence should preserve enough proof to explain:

```text
What was built?
        ↓
How did traffic flow?
        ↓
How did the application use its dependencies?
        ↓
How was CloudFront proven?
        ↓
What evidence demonstrates success?
```

---

## Navigation

- [← Back to README](../README.md)
- [Architecture](architecture.md)
- [Implementation](implementation.md)
- [Limitations & Future Work](limitations-and-future-work.md)
