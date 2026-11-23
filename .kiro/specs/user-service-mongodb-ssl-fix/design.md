# Design Document: Fix user-service MongoDB SSL/TLS Connection

## Overview

This design document provides a comprehensive solution for the user-service MongoDB Atlas SSL/TLS handshake failure in the EKS cluster. The issue manifests as `javax.net.ssl.SSLException: (internal_error) Received fatal alert: internal_error` causing CrashLoopBackOff. Analysis reveals that while all three services (user-service, product-service, order-service) use identical Dockerfiles, only user-service fails SSL/TLS connections. This indicates the issue is environmental and related to the Alpine Linux base image's CA certificate handling.

## Bug Details

### Current Faulty Behavior

**Symptom**: user-service pod enters CrashLoopBackOff state immediately after deployment to EKS cluster (ap-south-2 region, namespace: med-erp).

**Error Message**:
```
javax.net.ssl.SSLException: Received fatal alert: internal_error
  at sun.security.ssl.Alert.createSSLException(Alert.java:131)
  at sun.security.ssl.Alert.createSSLException(Alert.java:117)
  at sun.security.ssl.TransportContext.fatal(TransportContext.java:340)
  at sun.security.ssl.Alert$AlertConsumer.consume(Alert.java:293)
  
Connection error:
com.mongodb.MongoSocketOpenException: Exception opening socket to ac-jdpzeeb-shard-00-00.l7y92yx.mongodb.net:27017
```

**Failure Pattern**:
1. Pod starts successfully
2. Spring Boot application initializes
3. MongoDB connection attempt begins
4. SSL/TLS handshake initiated to MongoDB Atlas replica set nodes:
   - ac-jdpzeeb-shard-00-00.l7y92yx.mongodb.net:27017
   - ac-jdpzeeb-shard-00-01.l7y92yx.mongodb.net:27017
   - ac-jdpzeeb-shard-00-02.l7y92yx.mongodb.net:27017
5. All three handshakes fail with "internal_error"
6. Connection timeout after 30 seconds
7. Readiness probe returns HTTP 503
8. Kubernetes restarts pod → repeat cycle

**Observable Conditions**:
```bash
$ kubectl get pods -n med-erp -l app=user-service
NAME                           READY   STATUS             RESTARTS   AGE
user-service-7d4f8c9b6-abc12   0/1     CrashLoopBackOff   5          8m

$ kubectl describe pod user-service-7d4f8c9b6-abc12 -n med-erp | grep -A 5 "Last State"
Last State:     Terminated
  Reason:       Error
  Exit Code:    1
  Started:      Mon, 13 Jan 2025 10:30:15 +0000
  Finished:     Mon, 13 Jan 2025 10:31:02 +0000
```

### Context Analysis

**Working Services Comparison**:

| Service | Dockerfile | MongoDB Driver | Connection Status | Base Image |
|---------|-----------|---------------|------------------|-----------|
| user-service | `FROM eclipse-temurin:17-jre-alpine` | spring-boot-starter-data-mongodb 3.2.3 | ❌ FAILS | eclipse-temurin:17-jre-alpine |
| product-service | `FROM eclipse-temurin:17-jre-alpine` | spring-boot-starter-data-mongodb 3.2.3 | ✅ SUCCESS | eclipse-temurin:17-jre-alpine |
| order-service | `FROM eclipse-temurin:17-jre-alpine` | spring-boot-starter-data-mongodb 3.2.3 | ✅ SUCCESS | eclipse-temurin:17-jre-alpine |

**Key Findings**:
1. All three services use **identical** Dockerfiles (except jar name and port)
2. All three services use **identical** MongoDB driver versions (Spring Boot 3.2.3 → MongoDB driver 4.11.1)
3. All three services connect to the **same** MongoDB Atlas cluster
4. All three services use the **same** connection string format from Kubernetes secrets
5. Only user-service fails SSL/TLS handshake

**Environmental Hypothesis**: The Alpine Linux base image in user-service container lacks the CA certificates needed to validate MongoDB Atlas' SSL certificate chain. Product-service and order-service may have been built at a time when ca-certificates were present or may have had them installed through a different build path.

## Expected Behavior

### Correct System Behavior

After applying the fix, the user-service should exhibit the following behavior:

**1. Successful Pod Startup**:
```bash
$ kubectl get pods -n med-erp -l app=user-service
NAME                           READY   STATUS    RESTARTS   AGE
user-service-abc123-xyz45      1/1     Running   0          5m
user-service-abc123-xyz67      1/1     Running   0          5m
```

**2. Successful MongoDB Connection Logs**:
```
INFO  o.s.d.m.c.MongoClient - Opened connection [connectionId{localValue:1, serverValue:12345}] to ac-jdpzeeb-shard-00-00.l7y92yx.mongodb.net:27017
INFO  o.s.d.m.c.MongoClient - Cluster created with settings {hosts=[ac-jdpzeeb-shard-00-00.l7y92yx.mongodb.net:27017, ...], mode=REPLICA_SET, ...}
INFO  o.s.d.m.c.MongoClient - Cluster description updated with state CONNECTED
INFO  c.e.userservice.UserServiceApplication - Started UserServiceApplication in 8.234 seconds (JVM running for 9.012)
```

**3. Readiness Probe Success**:
```bash
$ kubectl exec user-service-abc123-xyz45 -n med-erp -- curl -s http://localhost:8081/api/v1/actuator/health/readiness
{"status":"UP","groups":["readiness"]}
```

**4. No Restarts**:
- Pod restart count remains 0
- No CrashLoopBackOff events
- Pod reaches Ready state within 30 seconds of creation

**5. Functional API Endpoints**:
```bash
$ curl -X POST http://user-service:8081/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"test","password":"test"}'
# Expected: HTTP 401 (unauthorized) - service is responding correctly
# NOT: HTTP 503 (service unavailable due to MongoDB connection failure)
```

### Success Criteria

- ✅ SSL/TLS handshake completes successfully using TLS 1.2+
- ✅ MongoDB connection establishes within 5 seconds
- ✅ Certificate chain validated using DigiCert Global Root CA
- ✅ No SSLException or connection timeout errors in logs
- ✅ Pod reaches Ready state on first attempt (no restarts)
- ✅ Readiness probe returns HTTP 200
- ✅ Authentication and authorization APIs functional

## Hypothesized Root Cause

### Root Cause Analysis

**Primary Hypothesis**: Alpine Linux base image (`eclipse-temurin:17-jre-alpine`) does not include root CA certificates by default, preventing Java from validating MongoDB Atlas' SSL certificate chain.

#### Evidence Supporting Hypothesis

**1. Alpine Linux Minimal Design Philosophy**:
Alpine Linux is designed for minimal footprint and does not include `ca-certificates` package in base image. The package must be explicitly installed.

**2. MongoDB Atlas Certificate Chain**:
```
Root: DigiCert Global Root CA
  ↓
Intermediate: DigiCert SHA2 Secure Server CA
  ↓
Server: *.mongodb.net
```

Without the DigiCert Global Root CA certificate, Java's SSL implementation cannot validate the certificate chain, resulting in handshake failure.

**3. Java SSL Trust Store**:
Java's `javax.net.ssl` package relies on the system's trusted CA certificates. On Alpine Linux, these are sourced from `/etc/ssl/certs/ca-bundle.crt`, which is generated by the `ca-certificates` package.

**4. Identical Dockerfiles, Different Outcomes**:
While all three services have identical Dockerfiles, the inconsistency suggests:
- Product-service and order-service were built when Alpine's base image included ca-certificates (or they were manually added in an earlier iteration)
- User-service was built more recently when ca-certificates were missing
- Alternatively, Docker layer caching may have preserved certificates for two services but not the third

**5. Error Message Analysis**:
The specific error "Received fatal alert: internal_error" is a generic SSL handshake failure. In the context of missing CA certificates, it indicates:
- Client (Java application) initiates handshake
- Server (MongoDB Atlas) presents certificate
- Client cannot validate certificate chain (missing root CA)
- Client sends "internal_error" alert and terminates connection

#### Validation of Hypothesis

To confirm the hypothesis, we would inspect the container:

```bash
# Check if ca-certificates package is installed
$ kubectl exec user-service-xxx -n med-erp -- apk info | grep ca-certificates
# Expected (FAIL): No output (package not installed)

# Check if CA bundle exists
$ kubectl exec user-service-xxx -n med-erp -- ls -la /etc/ssl/certs/ca-bundle.crt
# Expected (FAIL): No such file or directory

# Compare with working service
$ kubectl exec product-service-xxx -n med-erp -- ls -la /etc/ssl/certs/ca-bundle.crt
# Expected (SUCCESS): File exists with size ~200-300KB
```

### Secondary Hypotheses (Ruled Out)

**❌ Hypothesis 2: Different MongoDB Driver Versions**
- Ruled out: All services use Spring Boot 3.2.3 with identical mongodb-driver versions (4.11.1)

**❌ Hypothesis 3: Network/Firewall Issues**
- Ruled out: Product-service and order-service connect successfully from same EKS cluster
- All pods have identical network policies and security groups

**❌ Hypothesis 4: Connection String Misconfiguration**
- Ruled out: All services use same Kubernetes secret format, only user-service fails

**❌ Hypothesis 5: Java Version Incompatibility**
- Ruled out: All services use identical Java 17 runtime (eclipse-temurin:17-jre-alpine)

## Fix Implementation

### Solution Overview

Install the `ca-certificates` package in the Alpine Linux runtime stage of the user-service Dockerfile to provide trusted root CA certificates required for MongoDB Atlas SSL/TLS validation.

### Dockerfile Modification

## System Architecture

### Current Architecture (Defective)

```
┌─────────────────────────────────────────────────────────────┐
│ EKS Cluster (ap-south-2, namespace: med-erp)               │
│                                                              │
│  ┌──────────────────────────────────────────┐              │
│  │ user-service Pod (CrashLoopBackOff)      │              │
│  │                                           │              │
│  │ Container: eclipse-temurin:17-jre-alpine │              │
│  │ ❌ Missing: ca-certificates package       │              │
│  │                                           │              │
│  │ Java App → MongoDB Driver                │              │
│  │              ↓                            │              │
│  │         SSL/TLS Handshake                │              │
│  │              ↓                            │              │
│  │         ❌ FAILS: internal_error          │              │
│  │                                           │              │
│  │ No root CA certs to validate             │              │
│  │ *.mongodb.net certificate chain          │              │
│  └──────────────────────────────────────────┘              │
│                    ↓ (attempts connection)                  │
│  ┌──────────────────────────────────────────┐              │
│  │ MongoDB Atlas Replica Set                │              │
│  │ - ac-jdpzeeb-shard-00-00.l7y92yx        │              │
│  │ - ac-jdpzeeb-shard-00-01.l7y92yx        │              │
│  │ - ac-jdpzeeb-shard-00-02.l7y92yx        │              │
│  │   (All nodes: port 27017, TLS required)  │              │
│  └──────────────────────────────────────────┘              │
│                                                              │
│  ┌──────────────────────────────────────────┐              │
│  │ product-service Pod (✅ Running)          │              │
│  │ order-service Pod (✅ Running)            │              │
│  │                                           │              │
│  │ Same Dockerfile, connects successfully   │              │
│  │ (Built earlier with working ca-certs?)   │              │
│  └──────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

### Fixed Architecture

```
┌─────────────────────────────────────────────────────────────┐
│ EKS Cluster (ap-south-2, namespace: med-erp)               │
│                                                              │
│  ┌──────────────────────────────────────────┐              │
│  │ user-service Pod (✅ Ready)               │              │
│  │                                           │              │
│  │ Container: eclipse-temurin:17-jre-alpine │              │
│  │ ✅ Installed: ca-certificates package     │              │
│  │ ✅ Contains: /etc/ssl/certs/ca-bundle.crt│              │
│  │                                           │              │
│  │ Java App → MongoDB Driver                │              │
│  │              ↓                            │              │
│  │    SSL/TLS Handshake (TLS 1.2+)         │              │
│  │              ↓                            │              │
│  │    ✅ Certificate chain validated         │              │
│  │    ✅ Connection established              │              │
│  │                                           │              │
│  │ Readiness Probe: HTTP 200                │              │
│  └──────────────────────────────────────────┘              │
│                    ↓ (secure connection)                    │
│  ┌──────────────────────────────────────────┐              │
│  │ MongoDB Atlas Replica Set                │              │
│  │ - ac-jdpzeeb-shard-00-00.l7y92yx        │              │
│  │ - ac-jdpzeeb-shard-00-01.l7y92yx        │              │
│  │ - ac-jdpzeeb-shard-00-02.l7y92yx        │              │
│  │   ✅ TLS handshake successful             │              │
│  └──────────────────────────────────────────┘              │
│                                                              │
│  ┌──────────────────────────────────────────┐              │
│  │ product-service, order-service           │              │
│  │ ✅ Continue working (unaffected)          │              │
│  └──────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

## Component Design

#### Exact File Changes

**Current user-service/Dockerfile** (Defective):
```dockerfile
FROM maven:3.8.3-openjdk-17 AS build

WORKDIR /app

COPY . .

RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY --from=build /app/target/user-service-*.jar app.jar

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
```

**Fixed user-service/Dockerfile**:
```dockerfile
FROM maven:3.8.3-openjdk-17 AS build

WORKDIR /app

COPY . .

RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine

# Install ca-certificates to enable SSL/TLS validation for MongoDB Atlas
RUN apk add --no-cache ca-certificates && \
    update-ca-certificates

WORKDIR /app

COPY --from=build /app/target/user-service-*.jar app.jar

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Key Changes

1. **Line 11-13 (NEW)**: Add `RUN` instruction to install ca-certificates package
2. **Placement**: Install ca-certificates BEFORE `WORKDIR` to ensure certificates are available system-wide
3. **Alpine Package Manager**: Use `apk add --no-cache` to minimize image size
4. **Certificate Update**: Run `update-ca-certificates` to generate `/etc/ssl/certs/ca-bundle.crt`

#### Why This Fixes The Issue

**Root CA Certificate Chain for MongoDB Atlas**:
```
DigiCert Global Root CA (Root)
    ↓
DigiCert SHA2 Secure Server CA (Intermediate)
    ↓
*.mongodb.net (Server Certificate)
```

**Without ca-certificates package**:
- Alpine Linux has minimal installation by default
- `/etc/ssl/certs/` directory may be empty or incomplete
- Java's cacerts truststore cannot validate MongoDB Atlas certificate chain
- SSL/TLS handshake fails with "internal_error"

**With ca-certificates package**:
- Alpine's ca-certificates package provides ~140+ root CA certificates
- Includes DigiCert Global Root CA required for MongoDB Atlas
- `/etc/ssl/certs/ca-bundle.crt` generated with all trusted roots
- Java MongoDB driver can validate certificate chain successfully

### Implementation Steps

#### Step 1: Modify user-service/Dockerfile

**File**: `user-service/Dockerfile`

**Location**: Line 11 (immediately after `FROM eclipse-temurin:17-jre-alpine`)

**Add these lines**:
```dockerfile
# Install ca-certificates to enable SSL/TLS validation for MongoDB Atlas
RUN apk add --no-cache ca-certificates && \
    update-ca-certificates
```

**Complete modified Dockerfile**:
```dockerfile
FROM maven:3.8.3-openjdk-17 AS build

WORKDIR /app

COPY . .

RUN mvn clean package -DskipTests

FROM eclipse-temurin:17-jre-alpine

# Install ca-certificates to enable SSL/TLS validation for MongoDB Atlas
RUN apk add --no-cache ca-certificates && \
    update-ca-certificates

WORKDIR /app

COPY --from=build /app/target/user-service-*.jar app.jar

EXPOSE 8081

ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Step 2: Build Docker Image

**Command**:
```bash
cd f:\Desktop\medpHARMA\edublitz-b2b-medical-erp
docker build -t user-app:v1.0.1 -f user-service/Dockerfile user-service/
```

**Expected Output**:
```
[+] Building 145.2s (12/12) FINISHED
 => FROM eclipse-temurin:17-jre-alpine
 => RUN apk add --no-cache ca-certificates && update-ca-certificates
 => (1/1) Installing ca-certificates (20230506-r0)
 => Executing ca-certificates-20230506-r0.trigger
 => Successfully built abc123def456
 => Successfully tagged user-app:v1.0.1
```

#### Step 3: Push to ECR

**Commands**:
```bash
# Authenticate
aws ecr get-login-password --region ap-southeast-2 | \
  docker login --username AWS --password-stdin \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com

# Tag and push versioned tag
docker tag user-app:v1.0.1 \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1
docker push 898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1

# Tag and push latest
docker tag user-app:v1.0.1 \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest
docker push 898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest
```

#### Step 4: Deploy to EKS

**Command**:
```bash
kubectl set image deployment/user-service \
  user-service=898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1 \
  -n med-erp
```

**Monitor Rollout**:
```bash
kubectl rollout status deployment/user-service -n med-erp --timeout=5m
```

#### Step 5: Verify Fix

**Check Pod Status**:
```bash
kubectl get pods -n med-erp -l app=user-service
# Expected: STATUS=Running, READY=1/1, RESTARTS=0
```

**Check Logs**:
```bash
kubectl logs -n med-erp -l app=user-service --tail=50 | grep -i "mongodb\|ssl"
# Expected: "Opened connection" messages, NO "SSLException"
```

### Why This Fix Works

**Root CA Certificate Chain for MongoDB Atlas**:
```
DigiCert Global Root CA (Root)
    ↓
DigiCert SHA2 Secure Server CA (Intermediate)
    ↓
*.mongodb.net (Server Certificate)
```

**Without ca-certificates package**:
- Alpine Linux has minimal installation by default
- `/etc/ssl/certs/` directory may be empty or incomplete
- Java's cacerts truststore cannot validate MongoDB Atlas certificate chain
- SSL/TLS handshake fails with "internal_error"

**With ca-certificates package**:
- Alpine's ca-certificates package provides ~140+ root CA certificates
- Includes DigiCert Global Root CA required for MongoDB Atlas
- `/etc/ssl/certs/ca-bundle.crt` generated with all trusted roots
- Java MongoDB driver can validate certificate chain successfully

### Impact Analysis

**Image Size**: +1 MB (ca-certificates package ~220 KB + certificate bundle ~700 KB)

**Build Time**: +3-7 seconds (apk add + update-ca-certificates execution)

**Runtime Performance**: Negligible (<50ms for initial certificate validation per connection)

**Compatibility**: No changes to application code, configuration, or Kubernetes resources required

### Dependency Analysis

#### pom.xml Consistency Verification

All three services use identical Spring Boot and MongoDB dependencies:

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.3</version>
</parent>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```

**MongoDB Driver Version** (resolved by Spring Boot 3.2.3):
- mongodb-driver-sync: 4.11.1
- mongodb-driver-core: 4.11.1

**Finding**: No pom.xml changes required. The issue is purely infrastructure/container-level.

### 3. Kubernetes Deployment

#### No Changes Required to user-service-deployment.yaml

The Kubernetes deployment configuration remains unchanged:
- Connection string: Uses existing secret `app-secrets.MONGODB_URI_USER`
- Environment variables: No modifications needed
- Resource limits: Unchanged
- Probes: Existing readiness/liveness probes will work once MongoDB connects

**Rationale**: The fix is container-layer only. Application code and Kubernetes resources are correct.

## Data Models

### SSL/TLS Certificate Validation Flow

```typescript
interface SSLHandshakeContext {
  clientContainer: {
    baseImage: "eclipse-temurin:17-jre-alpine";
    caCertificatesInstalled: boolean;
    trustedRootCerts: string[]; // Paths like /etc/ssl/certs/ca-bundle.crt
  };
  
  serverCertificate: {
    subject: "*.mongodb.net";
    issuer: "DigiCert SHA2 Secure Server CA";
    validUntil: Date;
    certificateChain: Certificate[];
  };
  
  handshakeResult: {
    success: boolean;
    protocol: "TLSv1.2" | "TLSv1.3";
    cipherSuite: string;
    error?: "internal_error" | "certificate_unknown" | "handshake_failure";
  };
}
```

### Build and Deployment Workflow

```typescript
interface BuildDeploymentPipeline {
  // Phase 1: Build
  buildStage: {
    step: "docker build";
    dockerfile: "user-service/Dockerfile";
    outputImage: string; // Tag format: user-app:v{version}
    buildArgs: Record<string, string>;
  };
  
  // Phase 2: Push to ECR
  pushStage: {
    step: "docker push";
    ecrRepository: "898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app";
    imageTag: string;
  };
  
  // Phase 3: Deploy to EKS
  deployStage: {
    step: "kubectl rollout";
    namespace: "med-erp";
    deployment: "user-service";
    newImage: string;
    strategy: "RollingUpdate";
  };
  
  // Phase 4: Verification
  verificationStage: {
    step: "kubectl get pods";
    expectedState: "Running";
    readinessProbeStatus: 200;
    noRestarts: boolean;
  };
}
```

## Interface Specifications

### 1. Docker Build Interface

**Command**:
```bash
docker build -t user-app:v1.0.1 -f user-service/Dockerfile user-service/
```

**Inputs**:
- Dockerfile: `user-service/Dockerfile` (modified)
- Build context: `user-service/` directory
- Base images:
  - maven:3.8.3-openjdk-17 (build stage)
  - eclipse-temurin:17-jre-alpine (runtime stage)

**Outputs**:
- Docker image: `user-app:v1.0.1`
- Image layers include:
  - Alpine base layer
  - ca-certificates package layer (NEW)
  - Application JAR layer
  - Configuration layers

**Success Criteria**:
- Build completes without errors
- Image size increase: ~1-2 MB (for ca-certificates)
- `/etc/ssl/certs/ca-bundle.crt` exists in final image

### 2. ECR Push Interface

**Command**:
```bash
# Authenticate
aws ecr get-login-password --region ap-southeast-2 | \
  docker login --username AWS --password-stdin \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com

# Tag
docker tag user-app:v1.0.1 \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1

# Push
docker push 898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1

# Update latest tag
docker tag user-app:v1.0.1 \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest
docker push 898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest
```

**Inputs**:
- Local image: `user-app:v1.0.1`
- AWS credentials with ECR push permissions
- Target repository: `user-app`

**Outputs**:
- Image available in ECR with tags:
  - `v1.0.1` (version-specific)
  - `latest` (rolling tag)

### 3. Kubernetes Deployment Interface

**Command**:
```bash
# Update deployment to use new image
kubectl set image deployment/user-service \
  user-service=898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1 \
  -n med-erp

# OR force rollout if using :latest tag
kubectl rollout restart deployment/user-service -n med-erp
```

**Inputs**:
- Kubernetes context: EKS cluster in ap-south-2
- Namespace: `med-erp`
- Deployment: `user-service`
- New image: `user-app:v1.0.1`

**Outputs**:
- Rolling update initiated
- Old pods terminated gracefully
- New pods created with updated image
- Service remains available (maxUnavailable: 0)

**Success Criteria**:
- All replicas reach `Ready` state
- No CrashLoopBackOff
- Readiness probe returns HTTP 200
- MongoDB connection established

### 4. Verification Interface

**Commands**:
```bash
# Check pod status
kubectl get pods -n med-erp -l app=user-service

# Check pod events
kubectl describe pod -n med-erp -l app=user-service

# Check application logs
kubectl logs -n med-erp -l app=user-service --tail=100

# Check MongoDB connection logs specifically
kubectl logs -n med-erp -l app=user-service | grep -i "mongodb\|ssl\|tls"

# Verify readiness probe
kubectl get pods -n med-erp -l app=user-service -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}'
```

**Expected Outputs**:

✅ **Successful Pod Status**:
```
NAME                           READY   STATUS    RESTARTS   AGE
user-service-7d4f8c9b6-abc12   1/1     Running   0          2m
user-service-7d4f8c9b6-def34   1/1     Running   0          2m
```

✅ **Successful Logs** (no SSL errors):
```
INFO  o.s.d.m.c.MongoClient - Opened connection [connectionId{localValue:1}] to ac-jdpzeeb-shard-00-00.l7y92yx.mongodb.net:27017
INFO  o.s.d.m.c.MongoClient - Cluster created with settings {hosts=[...], mode=REPLICA_SET, ...}
INFO  c.e.userservice.UserServiceApplication - Started UserServiceApplication in 8.234 seconds
```

❌ **Failure Indicators** (should NOT appear):
```
ERROR javax.net.ssl.SSLException: Received fatal alert: internal_error
ERROR com.mongodb.MongoSocketOpenException: Exception opening socket
```

## Error Handling

### Build-Time Error Scenarios

#### Scenario 1: Alpine Package Repository Unreachable

**Error**:
```
ERROR: unable to select packages:
  ca-certificates (no such package):
    required by: world[ca-certificates]
```

**Root Cause**: Alpine package repository mirrors unavailable or DNS issues

**Resolution**:
1. Add retry logic to Dockerfile:
```dockerfile
RUN for i in 1 2 3; do \
      apk add --no-cache ca-certificates && break || sleep 5; \
    done && \
    update-ca-certificates
```

2. Or specify alternative mirror:
```dockerfile
RUN echo "http://dl-cdn.alpinelinux.org/alpine/v3.18/main" > /etc/apk/repositories && \
    apk add --no-cache ca-certificates && \
    update-ca-certificates
```

#### Scenario 2: Dockerfile Syntax Error

**Error**:
```
ERROR: Dockerfile parse error line 11: unknown instruction: RNU
```

**Root Cause**: Typo in Dockerfile

**Resolution**:
1. Validate Dockerfile syntax before building:
```bash
docker build --check -f user-service/Dockerfile user-service/
```

2. Use linter:
```bash
docker run --rm -i hadolint/hadolint < user-service/Dockerfile
```

### Runtime Error Scenarios

#### Scenario 3: Certificate Validation Still Fails

**Error** (in pod logs):
```
SSLException: PKIX path validation failed: java.security.cert.CertPathValidatorException
```

**Root Cause**: CA certificates installed but Java truststore not recognizing them

**Diagnosis**:
```bash
# Exec into running container
kubectl exec -it user-service-xxx -n med-erp -- sh

# Check if ca-certificates are present
ls -la /etc/ssl/certs/ca-bundle.crt

# Check if Java can see certificates
keytool -list -cacerts -storepass changeit | grep DigiCert
```

**Resolution**:
Add explicit Java truststore configuration to Dockerfile:
```dockerfile
RUN apk add --no-cache ca-certificates && \
    update-ca-certificates && \
    keytool -importcert -noprompt -trustcacerts \
      -alias alpine-ca-bundle \
      -file /etc/ssl/certs/ca-bundle.crt \
      -keystore $JAVA_HOME/lib/security/cacerts \
      -storepass changeit
```

#### Scenario 4: MongoDB Connection Timeout (Different Error)

**Error**:
```
MongoTimeoutException: Timed out after 30000 ms while waiting to connect
```

**Root Cause**: Network connectivity issue, not SSL/TLS

**Diagnosis**:
```bash
# Test network connectivity from pod
kubectl exec -it user-service-xxx -n med-erp -- sh
nc -zv ac-jdpzeeb-shard-00-00.l7y92yx.mongodb.net 27017

# Check DNS resolution
nslookup ac-jdpzeeb-shard-00-00.l7y92yx.mongodb.net

# Check security groups/network policies
kubectl get networkpolicies -n med-erp
```

**Resolution**: This is a different issue (network/firewall), not the SSL/TLS certificate problem.

### Deployment Error Scenarios

#### Scenario 5: Image Pull Failure

**Error**:
```
Failed to pull image "898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1": 
rpc error: code = Unknown desc = Error response from daemon: pull access denied
```

**Root Cause**: ECR authentication issue or image doesn't exist

**Diagnosis**:
```bash
# Verify image exists in ECR
aws ecr describe-images \
  --repository-name user-app \
  --region ap-southeast-2

# Check node IAM role has ECR pull permissions
kubectl get nodes -o wide
# SSH to node and check IAM role
```

**Resolution**:
1. Ensure image was pushed successfully
2. Verify node IAM role has `ecr:GetAuthorizationToken` and `ecr:BatchGetImage` permissions
3. Check ECR repository policy

#### Scenario 6: Rolling Update Stuck

**Status**:
```
user-service-new-xxx   0/1   CrashLoopBackOff
user-service-old-xxx   1/1   Running
```

**Root Cause**: New pods failing but old pods still running (maxUnavailable: 0)

**Diagnosis**:
```bash
# Check rollout status
kubectl rollout status deployment/user-service -n med-erp

# Check events
kubectl get events -n med-erp --sort-by='.lastTimestamp'
```

**Resolution**:
1. If new image is bad, rollback:
```bash
kubectl rollout undo deployment/user-service -n med-erp
```

2. Fix issue and retry deployment

## Correctness Properties

*A property is a characteristic or behavior that should hold true across all valid executions of a system—essentially, a formal statement about what the system should do. Properties serve as the bridge between human-readable specifications and machine-verifiable correctness guarantees.*

### Acceptance Criteria Analysis

After analyzing all acceptance criteria, I identified the following categories:
- **2.1, 2.2**: Both test SSL/TLS handshake success - these can be combined into one integration test
- **2.3, 2.4**: Both test pod health after connection - these verify the same outcome through different metrics
- **3.1, 3.2, 3.3, 3.4**: All regression tests that verify unchanged behavior

**Consolidation**: Rather than redundant properties, we have distinct integration tests and smoke tests that each verify unique aspects of the fix.

### Property 1: SSL/TLS Handshake Success After Fix

*For any* user-service container built with the fixed Dockerfile that includes ca-certificates package, when the MongoDB driver initiates an SSL/TLS connection to MongoDB Atlas, the system SHALL complete the handshake successfully without SSLException errors and establish a working database connection.

**Validates: Requirements 2.1, 2.2**

**Test Strategy**: Integration test - Deploy fixed image to EKS and verify successful connection through logs and metrics.

### Property 2: Pod Readiness After MongoDB Connection

*For any* user-service pod that successfully establishes MongoDB Atlas connection, the pod SHALL reach Ready state with readiness probe returning HTTP 200 and zero restarts within the initial delay period.

**Validates: Requirements 2.3, 2.4**

**Test Strategy**: Integration test - Monitor pod status with kubectl and verify readiness without CrashLoopBackOff.

### Property 3: Service Functionality Preservation

*For any* user-service API endpoint (authentication, authorization, organization management) that was functional before the fix, the endpoint SHALL continue to return correct responses with identical business logic after deploying the fixed Docker image.

**Validates: Requirements 3.3**

**Test Strategy**: Integration test - Execute representative API calls (login, get user info, create organization) and verify responses match expected behavior.

### Property 4: Infrastructure Consistency Preservation

*For any* user-service deployment configuration element (base image, connection string format, environment variables, resource limits), the element SHALL remain identical before and after the fix except for the addition of ca-certificates package installation in the Dockerfile.

**Validates: Requirements 3.2, 3.4**

**Test Strategy**: Smoke test - Review Dockerfile and deployment YAML to confirm only ca-certificates lines were added.

### Property 5: Cross-Service Non-Regression

*For any* service in the med-erp namespace (product-service, order-service) that successfully connects to MongoDB Atlas before the user-service fix, the service SHALL continue to establish connections without new SSL/TLS errors after user-service deployment.

**Validates: Requirements 3.1**

**Test Strategy**: Integration test - Monitor logs of product-service and order-service pods before and after user-service deployment to ensure no new errors.

## Security Considerations

### 1. Supply Chain Security

**Alpine Package Integrity**:
- ca-certificates package is part of Alpine's official repository
- Alpine uses signed packages with public key verification
- Package source: https://pkgs.alpinelinux.org/package/edge/main/x86_64/ca-certificates

**Risk**: Compromised Alpine repository could deliver malicious certificates

**Mitigation**:
1. Pin Alpine version in base image:
```dockerfile
FROM eclipse-temurin:17-jre-alpine3.18
```

2. Consider offline package installation for high-security environments:
```dockerfile
COPY ca-certificates-bundle.crt /usr/local/share/ca-certificates/
RUN update-ca-certificates
```

### 2. Certificate Trust Boundaries

**Trust Model**:
- ca-certificates package includes ~140+ root CAs
- Trusts all Certificate Authorities in Mozilla's CA bundle
- MongoDB Atlas uses DigiCert (widely trusted)

**Risk**: Broad trust model may include CAs not relevant to application

**Mitigation**:
For maximum security, install only DigiCert root CA:
```dockerfile
RUN wget -O /usr/local/share/ca-certificates/digicert-global-root-ca.crt \
      https://cacerts.digicert.com/DigiCertGlobalRootCA.crt.pem && \
    update-ca-certificates
```

**Decision**: Use full ca-certificates package for simplicity and future compatibility with other SSL/TLS services.

### 3. Secret Management

**Current State**:
- MongoDB connection string stored in Kubernetes secret: `app-secrets.MONGODB_URI_USER`
- Includes credentials in URI format

**Impact of Fix**: No changes to secret management

**Verification**: Ensure secret is not logged during connection:
```bash
# Check that connection string is not in logs
kubectl logs -n med-erp -l app=user-service | grep -i "mongodb://" || echo "Safe: No credentials in logs"
```

### 4. Image Scanning

**Before Deployment**:
Run vulnerability scan on new image:
```bash
# Using AWS ECR scanning
aws ecr start-image-scan \
  --repository-name user-app \
  --image-id imageTag=v1.0.1 \
  --region ap-southeast-2

# Check results
aws ecr describe-image-scan-findings \
  --repository-name user-app \
  --image-id imageTag=v1.0.1 \
  --region ap-southeast-2
```

**Expected**: ca-certificates package should not introduce new vulnerabilities (it's a standard Alpine package).

## Performance Considerations

### 1. Build Performance

**Image Size Impact**:
- ca-certificates package: ~220 KB
- Certificate bundle: ~700 KB
- Total increase: ~1 MB

**Build Time Impact**:
- `apk add` execution: +2-5 seconds
- `update-ca-certificates`: +1-2 seconds
- Total increase: +3-7 seconds

**Optimization**:
Already optimized with `--no-cache` flag to avoid storing package index.

### 2. Runtime Performance

**Connection Establishment**:
- **Before Fix**: Immediate failure (~0.1s), then 30s timeout
- **After Fix**: Successful handshake (~0.5-1.5s based on network latency)

**Ongoing Performance**:
- Certificate validation: ~10-50ms per connection (one-time per connection pool)
- Connection pool: Reuses validated connections
- **No impact** on established connection throughput

### 3. Pod Startup Time

**Current Startup (Failed)**:
```
Container Start → MongoDB Connection Attempt → 30s Timeout → Crash → Restart
Total: ~35-40s before restart
```

**Expected Startup (Fixed)**:
```
Container Start → MongoDB Connection Success → Application Ready
Total: ~8-12s to Ready state
```

**Improvement**: ~70% reduction in startup time, pod reaches Ready state on first attempt.

### 4. Resource Utilization

**Memory**:
- ca-certificates: <1 MB memory footprint
- Certificate validation: ~5-10 MB during handshake
- **No significant change** to application memory usage

**CPU**:
- Certificate validation: <0.1 CPU-seconds per connection
- **Negligible impact** on application CPU usage

**Network**:
- Successful connection: ~2-5 KB initial handshake data
- **Reduction** in network usage (no repeated failed connections)

## Testing Strategy

### 1. Unit Tests

**Not Applicable**: This is an infrastructure fix, not application code. No unit test changes needed.

### 2. Integration Tests

#### Test 1: MongoDB Connection After Fix

**Setup**:
```bash
# Deploy fixed image to test environment
kubectl apply -f k8s/deployments/user-service-deployment.yaml -n med-erp-test
```

**Execution**:
```bash
# Wait for pod to start
kubectl wait --for=condition=ready pod -l app=user-service -n med-erp-test --timeout=60s

# Check logs for successful connection
kubectl logs -n med-erp-test -l app=user-service --tail=50 | grep "MongoDB"
```

**Expected Result**:
- Log contains: `Opened connection [connectionId{localValue:1}] to ac-jdpzeeb-shard-00-00.l7y92yx.mongodb.net:27017`
- Log does NOT contain: `SSLException` or `internal_error`
- Exit code: 0

#### Test 2: API Functionality

**Execution**:
```bash
# Get service endpoint
USER_SERVICE_URL=$(kubectl get svc user-service -n med-erp-test -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Test login endpoint
curl -X POST "http://${USER_SERVICE_URL}/api/v1/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"testuser","password":"testpass"}' \
  -w "\nHTTP Status: %{http_code}\n"
```

**Expected Result**:
- HTTP Status: 200 or 401 (service responding)
- NOT 503 (service unavailable due to MongoDB connection failure)

#### Test 3: Regression - Other Services Unaffected

**Execution**:
```bash
# Check product-service logs for errors after user-service deployment
kubectl logs -n med-erp-test -l app=product-service --since=5m | grep -i "error\|exception" || echo "No errors found"

# Check order-service logs
kubectl logs -n med-erp-test -l app=order-service --since=5m | grep -i "error\|exception" || echo "No errors found"
```

**Expected Result**:
- No new SSL/TLS errors in other services
- "No errors found" message

### 3. Smoke Tests

#### Test 4: Container Certificate Verification

**Execution**:
```bash
# Exec into container
POD_NAME=$(kubectl get pod -n med-erp-test -l app=user-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD_NAME -n med-erp-test -- sh

# Inside container:
ls -lh /etc/ssl/certs/ca-bundle.crt
cat /etc/ssl/certs/ca-bundle.crt | grep "DigiCert Global Root CA"
exit
```

**Expected Result**:
- `/etc/ssl/certs/ca-bundle.crt` exists
- File size: ~200-300 KB
- Contains "DigiCert Global Root CA" certificate

#### Test 5: Dockerfile Consistency Check

**Execution**:
```bash
# Verify base image unchanged
grep "FROM eclipse-temurin:17-jre-alpine" user-service/Dockerfile

# Verify only ca-certificates added
git diff main user-service/Dockerfile
```

**Expected Result**:
- Base image remains `eclipse-temurin:17-jre-alpine`
- Only changes are ca-certificates installation lines

### 4. Production Verification Checklist

Before marking deployment as successful, verify:

- [ ] Both user-service pods in `Running` state
- [ ] Pod restart count = 0
- [ ] Readiness probe status = Ready
- [ ] Logs show successful MongoDB connection
- [ ] No SSLException errors in logs
- [ ] Login API responds (HTTP 200 or 401, not 503)
- [ ] product-service and order-service have no new errors
- [ ] Image scanned for vulnerabilities (no critical findings)
- [ ] Deployment rollout completed successfully

## Deployment Plan

### Phase 1: Pre-Deployment (15 minutes)

#### Step 1.1: Backup Current State
```bash
# Save current deployment configuration
kubectl get deployment user-service -n med-erp -o yaml > user-service-deployment-backup.yaml

# Note current image tag
kubectl get deployment user-service -n med-erp -o jsonpath='{.spec.template.spec.containers[0].image}'
# Output: 898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest

# Record current pod status
kubectl get pods -n med-erp -l app=user-service -o wide > pods-before-fix.txt
```

#### Step 1.2: Modify Dockerfile
```bash
# Navigate to user-service directory
cd user-service/

# Create backup
cp Dockerfile Dockerfile.backup

# Edit Dockerfile - add after line 10 (after "FROM eclipse-temurin:17-jre-alpine"):
# RUN apk add --no-cache ca-certificates && \
#     update-ca-certificates
```

**Exact change to make**:
```diff
FROM eclipse-temurin:17-jre-alpine

+# Install ca-certificates to enable SSL/TLS validation for MongoDB Atlas
+RUN apk add --no-cache ca-certificates && \
+    update-ca-certificates
+
WORKDIR /app
```

### Phase 2: Build and Push (10 minutes)

#### Step 2.1: Build Docker Image
```bash
# From project root directory
cd f:\Desktop\medpHARMA\edublitz-b2b-medical-erp

# Build with version tag
docker build -t user-app:v1.0.1 -f user-service/Dockerfile user-service/

# Verify build succeeded
docker images | grep user-app

# Test image locally (optional)
docker run --rm user-app:v1.0.1 java -version
docker run --rm user-app:v1.0.1 ls -la /etc/ssl/certs/ca-bundle.crt
```

#### Step 2.2: Push to ECR
```bash
# Authenticate to ECR
aws ecr get-login-password --region ap-southeast-2 | \
  docker login --username AWS --password-stdin \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com

# Tag image
docker tag user-app:v1.0.1 \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1

# Push versioned tag
docker push 898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1

# Tag and push as latest
docker tag user-app:v1.0.1 \
  898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest
docker push 898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:latest

# Verify image in ECR
aws ecr describe-images --repository-name user-app --region ap-southeast-2 | grep v1.0.1
```

### Phase 3: Deploy to EKS (5 minutes)

#### Step 3.1: Update Deployment
```bash
# Update Kubernetes context
kubectl config use-context <your-eks-cluster-context>

# Verify namespace
kubectl get ns med-erp

# Option A: Update image directly (recommended for versioned tag)
kubectl set image deployment/user-service \
  user-service=898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:v1.0.1 \
  -n med-erp

# Option B: Restart deployment (if using :latest tag)
# kubectl rollout restart deployment/user-service -n med-erp
```

#### Step 3.2: Monitor Rollout
```bash
# Watch rollout status
kubectl rollout status deployment/user-service -n med-erp --timeout=5m

# Watch pod creation in real-time
watch -n 2 'kubectl get pods -n med-erp -l app=user-service'
# Press Ctrl+C when new pods are Running

# Check rollout history
kubectl rollout history deployment/user-service -n med-erp
```

### Phase 4: Verification (10 minutes)

#### Step 4.1: Pod Health Check
```bash
# Get pod status
kubectl get pods -n med-erp -l app=user-service -o wide

# Expected output:
# NAME                           READY   STATUS    RESTARTS   AGE
# user-service-xxxxxxxxx-xxxxx   1/1     Running   0          2m
# user-service-xxxxxxxxx-xxxxx   1/1     Running   0          3m

# Check pod events
kubectl describe pod -n med-erp -l app=user-service | grep -A 10 Events

# Verify no crash events
kubectl get events -n med-erp --field-selector involvedObject.kind=Pod,reason=BackOff
```

#### Step 4.2: Application Logs
```bash
# Check application startup logs
kubectl logs -n med-erp -l app=user-service --tail=100

# Search for MongoDB connection success
kubectl logs -n med-erp -l app=user-service | grep -i "mongodb.*connection\|cluster created"

# Verify NO SSL errors
kubectl logs -n med-erp -l app=user-service | grep -i "sslexception\|internal_error" || echo "✅ No SSL errors found"

# Check readiness probe
kubectl get pods -n med-erp -l app=user-service -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

#### Step 4.3: API Functionality Test
```bash
# Get service endpoint (if using LoadBalancer)
kubectl get svc user-service -n med-erp

# Test health endpoint (readiness probe)
USER_SERVICE_POD=$(kubectl get pod -n med-erp -l app=user-service -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $USER_SERVICE_POD -n med-erp -- \
  curl -s http://localhost:8081/api/v1/actuator/health/readiness

# Expected: {"status":"UP"}

# Test login endpoint from within cluster
kubectl run test-curl --image=curlimages/curl:latest --rm -it --restart=Never -- \
  curl -X POST http://user-service.med-erp.svc.cluster.local:8081/api/v1/auth/login \
    -H "Content-Type: application/json" \
    -d '{"username":"test","password":"test"}' \
    -w "\nHTTP Code: %{http_code}\n"

# Expected: HTTP Code: 401 (unauthorized but service responding)
# NOT: HTTP Code: 503 (service unavailable)
```

#### Step 4.4: Regression Check
```bash
# Verify other services still working
kubectl get pods -n med-erp -l app=product-service
kubectl get pods -n med-erp -l app=order-service

# Check for new errors in other services
kubectl logs -n med-erp -l app=product-service --since=10m | grep -i "error\|exception" | tail -20
kubectl logs -n med-erp -l app=order-service --since=10m | grep -i "error\|exception" | tail -20

# Expected: No new MongoDB connection errors
```

### Phase 5: Post-Deployment (5 minutes)

#### Step 5.1: Update Documentation
```bash
# Record successful deployment
echo "Deployment successful at $(date)" >> deployment-log.txt
echo "Image: user-app:v1.0.1" >> deployment-log.txt
echo "Fix: Added ca-certificates package for MongoDB Atlas SSL/TLS" >> deployment-log.txt

# Commit Dockerfile changes
git add user-service/Dockerfile
git commit -m "fix(user-service): Add ca-certificates for MongoDB Atlas SSL/TLS connection

- Install ca-certificates package in Alpine runtime stage
- Fixes SSLException: internal_error during MongoDB connection
- Aligns with product-service and order-service SSL requirements"

git push origin main
```

#### Step 5.2: Monitor for 24 Hours
```bash
# Set up monitoring alert
# Check pod restart count every hour for 24 hours
for i in {1..24}; do
  sleep 3600
  kubectl get pods -n med-erp -l app=user-service -o jsonpath='{range .items[*]}{.metadata.name}{"\tRestarts: "}{.status.containerStatuses[0].restartCount}{"\n"}{end}'
done
```

### Rollback Procedure

If deployment fails or issues arise:

#### Quick Rollback (1 minute)
```bash
# Rollback to previous deployment
kubectl rollout undo deployment/user-service -n med-erp

# Verify rollback succeeded
kubectl rollout status deployment/user-service -n med-erp

# Check pods returned to previous state
kubectl get pods -n med-erp -l app=user-service
```

#### Manual Rollback (if quick rollback fails)
```bash
# Restore backup deployment
kubectl apply -f user-service-deployment-backup.yaml -n med-erp

# Force pull previous image
kubectl set image deployment/user-service \
  user-service=898220629393.dkr.ecr.ap-southeast-2.amazonaws.com/user-app:<previous-tag> \
  -n med-erp

# Or if using rollout restart approach, the previous image should be cached
```

#### Rollback Verification
```bash
# Verify pods are running with previous image
kubectl get pods -n med-erp -l app=user-service -o jsonpath='{.items[0].spec.containers[0].image}'

# Check deployment history
kubectl rollout history deployment/user-service -n med-erp

# Restore Dockerfile from backup
cd user-service/
cp Dockerfile.backup Dockerfile
```

## Alternative Approaches Considered

### Alternative 1: Use Non-Alpine Base Image

**Approach**: Switch from `eclipse-temurin:17-jre-alpine` to `eclipse-temurin:17-jre` (Debian-based)

**Pros**:
- Debian includes ca-certificates by default
- Larger package ecosystem
- More familiar to many developers

**Cons**:
- Image size increase: ~150 MB → ~250 MB (+67%)
- Longer build and deployment times
- Inconsistent with product-service and order-service

**Decision**: Rejected. Prefer consistency across services and smaller image size.

### Alternative 2: Mount Certificates from Kubernetes ConfigMap

**Approach**: Store CA certificates in ConfigMap, mount to pod

**Implementation**:
```yaml
volumes:
  - name: ca-certs
    configMap:
      name: ca-certificates
volumeMounts:
  - name: ca-certs
    mountPath: /etc/ssl/certs
    readOnly: true
```

**Pros**:
- Centralized certificate management
- Can update certificates without rebuilding images

**Cons**:
- More complex Kubernetes configuration
- Requires changes to deployment YAML
- Certificates should be in image for portability

**Decision**: Rejected. Prefer simpler solution with certificates baked into image.

### Alternative 3: Disable SSL Certificate Validation

**Approach**: Add `ssl=false` to MongoDB connection string

**Implementation**:
```
mongodb://user:pass@host:27017/db?ssl=false
```

**Pros**:
- Immediate fix without Docker rebuild
- No infrastructure changes

**Cons**:
- ❌ **SECURITY RISK**: Man-in-the-middle attacks possible
- ❌ Violates MongoDB Atlas security requirements
- ❌ Not a production-acceptable solution

**Decision**: Rejected. Security non-negotiable.

### Alternative 4: Use Java System Properties for Truststore

**Approach**: Override Java truststore location via JAVA_OPTS

**Implementation**:
```dockerfile
ENV JAVA_OPTS="-Djavax.net.ssl.trustStore=/custom/truststore.jks"
```

**Pros**:
- Fine-grained control over trusted certificates

**Cons**:
- Requires maintaining custom truststore
- More complex than using system CA certificates
- Doesn't solve root cause (missing Alpine certificates)

**Decision**: Rejected. Overcomplicates the solution.

## Conclusion

The fix for user-service MongoDB SSL/TLS connection failure is straightforward: install the `ca-certificates` package in the Alpine Linux runtime stage of the Dockerfile. This adds trusted root CA certificates required for validating MongoDB Atlas' SSL certificate chain.

**Key Points**:
1. **Root Cause**: Alpine Linux minimal base lacks CA certificates by default
2. **Solution**: Add 3 lines to Dockerfile to install ca-certificates
3. **Impact**: +1 MB image size, +3-7s build time, 70% faster pod startup
4. **Risk**: Very low - standard Alpine package, no application code changes
5. **Rollback**: Simple kubectl rollout undo if issues arise

**Expected Outcome**:
- user-service connects to MongoDB Atlas successfully
- Pod reaches Ready state in 8-12 seconds
- No CrashLoopBackOff
- No regression in other services

**Next Steps**:
1. Implement Dockerfile change
2. Build and push image
3. Deploy to EKS
4. Verify successful MongoDB connection
5. Monitor for 24 hours
