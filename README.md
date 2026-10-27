# 🏥 Patient Management System with Microservices

Enterprise-grade Patient Management System built with **Java Spring Boot 4.1** microservices, featuring **gRPC** inter-service communication, **Apache Kafka** event streaming, **JWT-secured API Gateway**, and **AWS infrastructure** (CloudFormation via CDK, ECS Fargate, RDS PostgreSQL, MSK).

---

## 📐 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        Client (REST)                             │
│                  http://localhost:4004                            │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│                  API Gateway (:4004)                              │
│         Spring Cloud Gateway (WebFlux)                           │
│         JWT Validation Filter                                    │
│         Route: /auth/** → Auth Service                           │
│         Route: /api/patients/** → Patient Service (JWT-secured)  │
└──────┬───────────────────────────────────┬───────────────────────┘
       │                                   │
       ▼                                   ▼
┌──────────────────┐            ┌──────────────────────────────────┐
│  Auth Service    │            │      Patient Service (:4000)     │
│    (:4005)       │            │  ┌─────────┐   ┌──────────────┐  │
│                  │            │  │REST API  │   │gRPC Client   │  │
│  • Login         │            │  │CRUD Ops  │   │→ Billing Svc │  │
│  • JWT Generate  │            │  └────┬─────┘   └──────┬───────┘  │
│  • JWT Validate  │            │       │                │          │
│  • Spring Sec.   │            │       ▼                ▼          │
│  • PostgreSQL    │            │  ┌─────────┐   ┌──────────────┐  │
│                  │            │  │PostgreSQL│   │Kafka Producer│  │
└──────────────────┘            │  └─────────┘   └──────┬───────┘  │
                                └───────────────────────┼──────────┘
                                                        │
                  ┌─────────────────────────────────────┤
                  │                                     │
                  ▼                                     ▼
┌──────────────────────────┐         ┌─────────────────────────────┐
│  Billing Service (:4001) │         │  Analytics Service (:4002)  │
│  gRPC Server (:9001)     │         │                             │
│                          │         │  • Kafka Consumer           │
│  • CreateBillingAccount  │         │  • Topic: "patient"         │
│  • Protobuf messages     │         │  • PatientEvent processing  │
└──────────────────────────┘         └─────────────────────────────┘
```

---

## 🧩 Microservices

### 1. API Gateway
> **Port:** `4004` &nbsp;|&nbsp; **Framework:** Spring Cloud Gateway (WebFlux)

The single entry point for all client requests. Routes traffic to downstream services and enforces authentication.

| Route Pattern | Target Service | Auth Required |
|---|---|---|
| `/auth/**` | Auth Service (`:4005`) | ❌ |
| `/api/patients/**` | Patient Service (`:4000`) | ✅ JWT |
| `/api-docs/patients` | Patient Service OpenAPI | ❌ |
| `/api-docs/auth` | Auth Service OpenAPI | ❌ |

**Key component:** [`JwtValidationGatewayFilterFactory`](file:///e:/Patient%20Management%20System%20with%20Microservices/api-gateway/src/main/java/com/pm/apigateway/filter/JwtValidationGatewayFilterFactory.java) — validates Bearer tokens against the Auth Service before forwarding requests.

---

### 2. Auth Service
> **Port:** `4005` &nbsp;|&nbsp; **Database:** PostgreSQL

Handles user authentication and JWT token lifecycle.

| Endpoint | Method | Description |
|---|---|---|
| `/login` | `POST` | Authenticate user, return JWT |
| `/validate` | `GET` | Validate a Bearer token |

**Stack:** Spring Security, JJWT (`0.12.6`), Spring Data JPA, PostgreSQL, OpenAPI/Swagger

---

### 3. Patient Service
> **Port:** `4000` &nbsp;|&nbsp; **Database:** PostgreSQL

The core domain service. Manages patient records and orchestrates cross-service workflows.

| Endpoint | Method | Description |
|---|---|---|
| `/patients` | `GET` | List all patients |
| `/patients` | `POST` | Create a new patient |
| `/patients/{id}` | `PUT` | Update a patient |
| `/patients/{id}` | `DELETE` | Delete a patient |

**On patient creation:**
1. Persists the patient in PostgreSQL
2. Calls **Billing Service** via gRPC to create a billing account
3. Publishes a `PATIENT_CREATED` event to Kafka (Protobuf-serialized)

**Patient Model fields:** `id` (UUID), `name`, `email` (unique), `address`, `dateOfBirth`, `registeredDate`

---

### 4. Billing Service
> **HTTP Port:** `4001` &nbsp;|&nbsp; **gRPC Port:** `9001`

Receives gRPC calls from the Patient Service to create billing accounts.

**Protobuf contract** ([`billing_service.proto`](file:///e:/Patient%20Management%20System%20with%20Microservices/patient-service/src/main/proto/billing_service.proto)):
```protobuf
service BillingService {
  rpc CreateBillingAccount (BillingRequest) returns (BillingResponse);
}
```

---

### 5. Analytics Service
> **Port:** `4002`

Consumes Kafka events from the `patient` topic and processes patient lifecycle analytics.

**Kafka Consumer config:**
- **Topic:** `patient`
- **Group ID:** `analytics-service`
- **Deserialization:** Protobuf (`PatientEvent`)

---

## 🛠️ Tech Stack

| Category | Technology | Version |
|---|---|---|
| **Language** | Java | 25 |
| **Framework** | Spring Boot | 4.1.x |
| **API Gateway** | Spring Cloud Gateway (WebFlux) | 2025.1.2 |
| **Database** | PostgreSQL | - |
| **ORM** | Spring Data JPA / Hibernate | - |
| **Authentication** | Spring Security + JJWT | 0.12.6 |
| **Inter-service (sync)** | gRPC + Protobuf | 1.81.1 / 4.29.1 |
| **Inter-service (async)** | Apache Kafka | - |
| **API Docs** | SpringDoc OpenAPI (Swagger) | 2.6.0 / 3.1.0 |
| **Infrastructure** | AWS CDK (Java) | 2.263.0 |
| **Cloud Services** | ECS Fargate, RDS, MSK, ALB | - |
| **Containerization** | Docker (multi-stage) | Maven 3.9.16 + Eclipse Temurin 25 |
| **Build Tool** | Maven | - |
| **Testing** | JUnit Jupiter 6, REST Assured 6 | - |
| **Local Cloud** | LocalStack | - |

---

## 🏗️ Infrastructure (AWS CDK)

The [`infrastructure/`](file:///e:/Patient%20Management%20System%20with%20Microservices/infrastructure) module uses **AWS CDK for Java** to define the entire cloud stack, deployable via **LocalStack** for local development.

**Resources provisioned:**

| Resource | Details |
|---|---|
| **VPC** | `PatientManagementVPC` — 2 Availability Zones |
| **RDS** | 2× PostgreSQL 18.3 instances (Auth DB + Patient DB), `db.t3.micro`, encrypted, auto-generated credentials |
| **MSK** | Kafka 4.0.0 cluster, 1 broker node (`kafka.m5.xlarge`) |
| **ECS Cluster** | Fargate cluster with Cloud Map service discovery (`patient-management.local`) |
| **Fargate Services** | 5 services (API Gateway, Auth, Patient, Billing, Analytics) — 256 CPU / 512 MiB each |
| **ALB** | Application Load Balancer fronting the API Gateway |
| **CloudWatch** | Log groups per service (`/ecs/<service-name>`), 1-day retention |
| **Health Checks** | Route 53 TCP health checks on both RDS instances |

**Deploy locally with LocalStack:**
```bash
cd infrastructure
mvn compile exec:java   # synthesize CDK template
./localstack-deploy.sh  # deploy to LocalStack
```

---

## 🚀 Getting Started

### Prerequisites
- Java 25+
- Maven 3.9+
- Docker
- LocalStack (for infrastructure testing)

### Build all services
```bash
# Build each service
cd patient-service && mvn clean package && cd ..
cd auth-service && mvn clean package && cd ..
cd billing-service && mvn clean package && cd ..
cd analytics-service && mvn clean package && cd ..
cd api-gateway && mvn clean package && cd ..
```

### Docker build
```bash
docker build -t patient-service ./patient-service
docker build -t auth-service ./auth-service
docker build -t billing-service ./billing-service
docker build -t analytics-service ./analytics-service
docker build -t api-gateway ./api-gateway
```

### Service Ports

| Service | HTTP Port | gRPC Port |
|---|---|---|
| API Gateway | `4004` | — |
| Patient Service | `4000` | — |
| Auth Service | `4005` | — |
| Billing Service | `4001` | `9001` |
| Analytics Service | `4002` | — |

---

## 📡 API Usage Examples

### 1. Login (get JWT token)
```http
POST http://localhost:4004/auth/login
Content-Type: application/json

{
  "email": "testuser@test.com",
  "password": "password123"
}
```

### 2. Create Patient (with JWT)
```http
POST http://localhost:4004/api/patients
Content-Type: application/json
Authorization: Bearer <token>

{
  "name": "John Doe",
  "email": "john@example.com",
  "address": "New York, USA",
  "dateOfBirth": "1990-05-15",
  "registeredDate": "2026-01-01"
}
```

### 3. List Patients
```http
GET http://localhost:4004/api/patients
Authorization: Bearer <token>
```

### 4. gRPC — Create Billing Account
```
Service: billing.BillingService/CreateBillingAccount
Host: localhost:9001

{
  "patientId": "<uuid>",
  "name": "John Doe",
  "email": "john@example.com"
}
```

---

## 📂 Project Structure

```
├── api-gateway/              # Spring Cloud Gateway — routing & JWT validation
├── auth-service/             # Authentication — login, JWT generation & validation
├── patient-service/          # Core domain — CRUD, gRPC client, Kafka producer
├── billing-service/          # Billing — gRPC server for billing account creation
├── analytics-service/        # Analytics — Kafka consumer for patient events
├── infrastructure/           # AWS CDK stack (VPC, RDS, MSK, ECS, ALB)
├── integration-tests/        # End-to-end tests (REST Assured + JUnit Jupiter)
├── api-requests/             # HTTP request files for manual API testing
│   ├── auth-service/         #   Login & validate requests
│   └── patient-service/      #   CRUD patient requests
└── grpc-requests/            # gRPC request files for manual testing
    └── billing-service/      #   CreateBillingAccount request
```

---

## 🔄 Communication Patterns

| Pattern | From | To | Protocol | Purpose |
|---|---|---|---|---|
| **Synchronous** | API Gateway | Auth Service | HTTP/REST | JWT validation |
| **Synchronous** | Patient Service | Billing Service | gRPC (Protobuf) | Create billing account |
| **Asynchronous** | Patient Service | Analytics Service | Kafka (Protobuf) | Patient created events |

---

## 📜 License

This project is licensed under the [MIT License](file:///e:/Patient%20Management%20System%20with%20Microservices/LICENSE).

Copyright © 2026 Tenka04
