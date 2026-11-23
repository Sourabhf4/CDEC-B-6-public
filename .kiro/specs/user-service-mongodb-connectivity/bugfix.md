# Bugfix Requirements Document

## Introduction

The user-service pod in the EKS cluster (namespace: med-erp, region: ap-south-2) is experiencing CrashLoopBackOff due to MongoDB connectivity failures. The pod's readiness probe continuously fails with HTTP 503 and "context deadline exceeded" errors, while product-service and order-service successfully connect to MongoDB Atlas using the same cluster configuration. Root cause analysis reveals that the Kubernetes secret key `MONGODB_URI_USER` contains a localhost connection string (`mongodb://127.0.0.1:27017/users_db`) instead of a valid MongoDB Atlas URI, preventing the user-service from establishing database connectivity in the containerized environment.

## Bug Analysis

### Current Behavior (Defect)

1.1 WHEN the user-service pod starts in the Kubernetes cluster THEN the system attempts to connect to `mongodb://127.0.0.1:27017/users_db` (localhost)

1.2 WHEN the MongoDB connection is attempted using the localhost URI THEN the system fails with connection timeout errors because localhost does not resolve to a MongoDB instance within the pod

1.3 WHEN the readiness probe polls `/api/v1/actuator/health/readiness` THEN the system returns HTTP 503 because the MongoDB health check fails

1.4 WHEN the readiness probe fails repeatedly THEN Kubernetes keeps the pod in an unready state and eventually triggers CrashLoopBackOff

1.5 WHEN the secret key `MONGODB_URI_USER` is read from the `app-secrets` secret THEN the system retrieves the base64-encoded localhost URI `mongodb://127.0.0.1:27017/users_db`

### Expected Behavior (Correct)

2.1 WHEN the user-service pod starts in the Kubernetes cluster THEN the system SHALL attempt to connect to a valid MongoDB Atlas URI in the format `mongodb+srv://username:password@cluster.mongodb.net/users_db`

2.2 WHEN the MongoDB connection is attempted using the Atlas URI THEN the system SHALL successfully establish connectivity to the MongoDB Atlas cluster

2.3 WHEN the readiness probe polls `/api/v1/actuator/health/readiness` THEN the system SHALL return HTTP 200 indicating healthy MongoDB connectivity

2.4 WHEN the readiness probe succeeds THEN Kubernetes SHALL mark the pod as ready and route traffic to it

2.5 WHEN the secret key `MONGODB_URI_USER` is read from the `app-secrets` secret THEN the system SHALL retrieve a base64-encoded MongoDB Atlas URI that matches the working pattern used by `MONGODB_URI_PRODUCT` and `MONGODB_URI_ORDER`

### Unchanged Behavior (Regression Prevention)

3.1 WHEN product-service reads `MONGODB_URI_PRODUCT` from the `app-secrets` secret THEN the system SHALL CONTINUE TO connect successfully to MongoDB Atlas

3.2 WHEN order-service reads `MONGODB_URI_ORDER` from the `app-secrets` secret THEN the system SHALL CONTINUE TO connect successfully to MongoDB Atlas

3.3 WHEN any service's readiness probe succeeds with existing MongoDB connectivity THEN Kubernetes SHALL CONTINUE TO mark those pods as ready and route traffic normally

3.4 WHEN the JWT_SECRET is read from the `app-secrets` secret THEN the system SHALL CONTINUE TO retrieve the correct JWT secret value for all services

3.5 WHEN the user-service deployment references the secret key `MONGODB_URI_USER` via `secretKeyRef` THEN the system SHALL CONTINUE TO use the same Kubernetes secret mounting mechanism without changes to the deployment YAML structure
