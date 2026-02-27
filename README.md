# 🏥 Patient Management System — Microservices Architecture

A **production-ready Patient Management System** built with **Java Spring Boot** using a full **microservices architecture**. The system covers patient registration, billing, analytics, notifications, and authentication — all communicating via REST, gRPC, and Apache Kafka — and is deployable to **AWS** using **LocalStack** and **AWS CDK** with Docker and ECS Fargate.


## 📋 Table of Contents

- [Architecture Overview](#architecture-overview)
- [Microservices](#microservices)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Getting Started](#getting-started)
  - [1. Clone the Repository](#1-clone-the-repository)
  - [2. Run with Docker](#2-run-with-docker)
- [Service Details & Environment Variables](#service-details--environment-variables)
  - [Patient Service](#patient-service)
  - [Billing Service (gRPC)](#billing-service-grpc)
  - [Analytics Service](#analytics-service)
  - [Auth Service](#auth-service)
  - [API Gateway](#api-gateway)
- [gRPC Setup](#grpc-setup)
- [Kafka Setup](#kafka-setup)
- [Authentication & Security](#authentication--security)
- [API Endpoints](#api-endpoints)
- [Infrastructure & AWS Deployment](#infrastructure--aws-deployment)
- [Integration Tests](#integration-tests)
- [Contributing](#contributing)

---

## 🏛️ Architecture Overview

This system is built around independent, loosely coupled microservices. Each service has its own **PostgreSQL database**, communicates synchronously via **REST** or **gRPC**, and asynchronously via **Apache Kafka** for event-driven workflows.

```
                          ┌───────────────┐
                          │  Client/User  │
                          └──────┬────────┘
                                 │ HTTP (Bearer Token)
                          ┌──────▼────────┐
                          │  API Gateway  │  ← JWT validation, routing
                          └──────┬────────┘
                 ┌───────────────┼───────────────┐
                 │               │               │
        ┌────────▼──────┐ ┌──────▼──────┐ ┌────▼──────────┐
        │ Patient Svc   │ │  Auth Svc   │ │ Analytics Svc │
        │ (REST + gRPC  │ │  (JWT/Auth) │ │  (Kafka Con.) │
        │  + Kafka Pub) │ └─────────────┘ └───────────────┘
        └────────┬───────┘
                 │ gRPC
        ┌────────▼───────┐
        │  Billing Svc   │  ← gRPC Server (port 9005)
        └────────────────┘
                 │ Kafka (async events)
        ┌────────▼───────────┐
        │ Notification Svc   │  ← Event consumer
        └────────────────────┘
```

**Communication Patterns:**
- **REST** — Client ↔ API Gateway ↔ Patient Service
- **gRPC** — Patient Service → Billing Service (synchronous inter-service calls)
- **Kafka** — Patient Service publishes events → Notification & Analytics Services consume them (async, event-driven)

---

## 🧩 Microservices

| Service | Responsibility | Communication |
|---------|---------------|---------------|
| **patient-service** | Core CRUD for patient records; publishes events | REST (inbound), gRPC (outbound), Kafka (producer) |
| **billing-service** | Handles billing accounts when a patient is created | gRPC Server (port 9005) |
| **analytics-service** | Consumes events to aggregate analytics data | Kafka Consumer |
| **auth-service** | User authentication; issues JWT tokens | REST |
| **api-gateway** | Single entry point; validates JWT, routes requests | REST (Spring Cloud Gateway) |
| **infrastructure** | AWS CDK code for cloud deployment (ECS, RDS, MSK) | AWS / LocalStack |
| **integration-tests** | End-to-end API tests | REST (test runner) |

---

## 🛠️ Tech Stack

| Category | Technology |
|----------|-----------|
| Language | Java 17+ |
| Framework | Spring Boot |
| ORM | Spring Data JPA / Hibernate |
| Database | PostgreSQL (per service) |
| Build Tool | Maven |
| Inter-service (sync) | gRPC + Protocol Buffers |
| Messaging (async) | Apache Kafka |
| API Gateway | Spring Cloud Gateway |
| Security | Spring Security + JWT (jjwt 0.12.6) |
| API Docs | SpringDoc OpenAPI (Swagger UI) |
| Containerization | Docker + Docker Compose |
| Cloud / IaC | AWS CDK + LocalStack (ECS Fargate, RDS, MSK) |
| Testing | JUnit, Spring Boot Test, Integration Tests |
| IDE (recommended) | IntelliJ IDEA Ultimate |

---

## 📁 Project Structure

```
patient-management/
├── patient-service/            # Patient CRUD + Kafka producer + gRPC client
├── billing-service/            # gRPC server for billing account creation
├── analytics-service/          # Kafka consumer for analytics
├── auth-service/               # JWT-based authentication
├── api-gateway/                # Spring Cloud Gateway — routing + JWT validation
├── infrastructure/             # AWS CDK (ECS, RDS, MSK on LocalStack)
├── integration-tests/          # End-to-end integration test suite
├── api-requests/               # Sample HTTP request files (REST client)
├── grpc-requests/
│   └── billing-service/        # Sample gRPC request files
└── .gitignore
```

Each service is an independent Maven project with its own:
- `pom.xml` — dependencies
- `Dockerfile` — container image
- `src/main/resources/application.properties` — config
- `src/main/java/...` — Controller → Service → Repository layers

---

## ✅ Prerequisites

Make sure you have the following installed:

| Tool | Version | Purpose |
|------|---------|---------|
| [Java JDK](https://adoptium.net/) | 17+ | Run services |
| [Maven](https://maven.apache.org/) | 3.8+ | Build |
| [Docker Desktop](https://www.docker.com/get-started) | Latest | Containers & Compose |
| [IntelliJ IDEA Ultimate](https://www.jetbrains.com/idea/) | Latest | Recommended IDE |


## 🚀 Getting Started

### 1. Clone the Repository

```bash
git clone https://github.com/VikasKumar7777/patient-management.git
cd patient-management/patient-management
```

### 2. Run with Docker

The entire stack (all services + databases + Kafka) can be started with Docker Compose from the project root:

```bash
docker-compose up --build
```

To run in detached (background) mode:

```bash
docker-compose up --build -d
```

To stop all services:

```bash
docker-compose down
```

> Each service has its own `Dockerfile`. Docker Compose orchestrates all services, databases, and the Kafka broker together.

---

## ⚙️ Service Details & Environment Variables

### Patient Service

The core service. Handles patient CRUD via REST, calls the Billing Service via gRPC when a new patient is created, and publishes patient events to Kafka.

**Environment Variables:**

```env
BILLING_SERVICE_ADDRESS=billing-service
BILLING_SERVICE_GRPC_PORT=9005
JAVA_TOOL_OPTIONS=-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
SPRING_SQL_INIT_MODE=always
```

**`application.properties` (Kafka deserializers):**

```properties
spring.kafka.consumer.key-deserializer=org.apache.kafka.common.serialization.StringDeserializer
spring.kafka.consumer.value-deserializer=org.apache.kafka.common.serialization.ByteArrayDeserializer
```

---

### Billing Service (gRPC)

Runs a **gRPC server on port 9005**. When the Patient Service creates a new patient, it makes a gRPC call to the Billing Service to create the corresponding billing account. Uses Protocol Buffers (`.proto`) for message contracts.

**Maven Dependencies (add to `<dependencies>`):**

```xml
<!-- gRPC -->
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.69.0</version>
</dependency>
<dependency>
    <groupId>org.apache.tomcat</groupId>
    <artifactId>annotations-api</artifactId>
    <version>6.0.53</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.29.1</version>
</dependency>
```

**Maven Build Section (replace `<build>`):**

```xml
<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.0</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:3.25.5:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:1.68.1:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

> The same gRPC dependencies and build config apply to **patient-service** as well (it acts as the gRPC client).

---

### Analytics Service

Consumes patient events from Kafka to track and aggregate analytics data. Uses Protobuf to deserialize Kafka messages.

**Environment Variables:**

```env
SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092
```

**Maven Dependencies (add in addition to existing):**

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
    <version>3.3.0</version>
</dependency>
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>4.29.1</version>
</dependency>
```

---

### Auth Service

Handles user authentication. Issues signed **JWT Bearer tokens** that the API Gateway uses to authorize all incoming requests. Backed by its own **PostgreSQL database**.

**Environment Variables:**

```env
SPRING_DATASOURCE_PASSWORD=password
SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db
SPRING_DATASOURCE_USERNAME=admin_user
SPRING_JPA_HIBERNATE_DDL_AUTO=update
SPRING_SQL_INIT_MODE=always
```

**Auth Service DB Environment Variables:**

```env
POSTGRES_DB=db
POSTGRES_PASSWORD=password
POSTGRES_USER=admin_user
```

**Default Seeded User (`data.sql`):**

```sql
CREATE TABLE IF NOT EXISTS "users" (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    role VARCHAR(50) NOT NULL
);

INSERT INTO "users" (id, email, password, role)
SELECT '223e4567-e89b-12d3-a456-426614174006',
       'testuser@test.com',
       '$2b$12$7hoRZfJrRKD2nIm2vHLs7OBETy.LWenXXMLKf99W8M4PUwO6KB7fu',
       'ADMIN'
WHERE NOT EXISTS (
    SELECT 1 FROM "users"
    WHERE id = '223e4567-e89b-12d3-a456-426614174006'
       OR email = 'testuser@test.com'
);
```

> Default credentials: **Email:** `testuser@test.com` | **Password:** `password`

**Key Maven Dependencies:**

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.6</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.6</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.6.0</version>
</dependency>
```

---

### API Gateway

The **single entry point** for all client requests. Uses **Spring Cloud Gateway** to route requests to the appropriate downstream service. Validates JWT tokens before forwarding — unauthorized requests are rejected here, never reaching internal services.

**How it works:**
1. Client sends request with `Authorization: Bearer <token>`
2. API Gateway validates the JWT
3. If valid → request forwarded to the target microservice
4. If invalid → `401 Unauthorized` returned immediately

---

## 🔌 gRPC Setup

The **Patient Service** is the gRPC **client** and the **Billing Service** is the gRPC **server**.

When a new patient is created via REST, the Patient Service makes a synchronous gRPC call to the Billing Service to create the corresponding billing account:

- **Host:** `billing-service` (Docker internal hostname)
- **Port:** `9005`

Protocol Buffer (`.proto`) files define the message contracts and are compiled automatically during `mvn compile` via the `protobuf-maven-plugin`. Refer to the [Billing Service](#billing-service-grpc) section for the full Maven configuration. The same gRPC dependencies apply to both services.

Sample gRPC request files are available in `grpc-requests/billing-service/`.

---

## 📨 Kafka Setup

Apache Kafka enables **asynchronous, event-driven communication** between services. When a patient record is created or modified, the **Patient Service** publishes a Kafka event (serialized with **Protobuf**). The **Analytics Service** (and optionally a Notification Service) consumes these events independently.

**Kafka Broker Container Environment Variables** _(for IntelliJ run config)_:

```env
KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
KAFKA_CFG_CONTROLLER_LISTENER_NAMES=CONTROLLER
KAFKA_CFG_CONTROLLER_QUORUM_VOTERS=0@kafka:9093
KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT
KAFKA_CFG_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
KAFKA_CFG_NODE_ID=0
KAFKA_CFG_PROCESS_ROLES=controller,broker
```

---

## 🔐 Authentication & Security

This system uses **JWT (JSON Web Tokens)** for stateless, token-based authentication:

1. **Login** — POST credentials to `/auth/login` → receive a signed JWT
2. **Authorized requests** — Include the token as `Authorization: Bearer <your-token>`
3. **Gateway validation** — API Gateway checks the JWT signature on every request; valid tokens pass through, invalid tokens return `401`
4. **Role-based access** — Users carry roles (e.g., `ADMIN`) encoded in the JWT payload

Passwords are stored hashed with **BCrypt**. The Auth Service uses **Spring Security** + **jjwt** for token lifecycle management.

---

## 📡 API Endpoints

All requests go through the **API Gateway** on port `8080`. Include your JWT in the `Authorization` header for all protected endpoints.

**Base URL:** `http://localhost:8080`

### Auth

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|:---:|
| `POST` | `/auth/login` | Login and receive a JWT | ❌ |
| `GET` | `/auth/validate` | Validate an existing JWT | ✅ |

**Login Example:**

```http
POST /auth/login
Content-Type: application/json

{
  "email": "testuser@test.com",
  "password": "password"
}
```

**Response:**

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9..."
}
```

### Patients

| Method | Endpoint | Description | Auth Required |
|--------|----------|-------------|:---:|
| `GET` | `/api/patients` | Get all patients | ✅ |
| `GET` | `/api/patients/{id}` | Get patient by ID | ✅ |
| `POST` | `/api/patients` | Create a new patient | ✅ |
| `PUT` | `/api/patients/{id}` | Update patient details | ✅ |
| `DELETE` | `/api/patients/{id}` | Delete a patient | ✅ |

**Create Patient Example:**

```http
POST /api/patients
Authorization: Bearer <token>
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "address": "123 Main Street",
  "dateOfBirth": "1990-01-15"
}
```

> 📁 Full sample HTTP request files are in `api-requests/` and gRPC samples in `grpc-requests/billing-service/`.

---

## ☁️ Infrastructure & AWS Deployment

The `infrastructure/` folder contains **AWS CDK** code to deploy the entire system to AWS — or locally using **LocalStack** (no real AWS costs needed).

**AWS Services used:**

| Service | Role |
|---------|------|
| **ECS Fargate** | Runs each microservice as a Docker container |
| **RDS (PostgreSQL)** | Managed database — one per service |
| **MSK (Managed Kafka)** | Managed Apache Kafka cluster |
| **AWS API Gateway** | Cloud-level API routing and management |

**Deploy locally with LocalStack:**

```bash
# Make sure LocalStack is running
cd infrastructure
cdklocal deploy
```

> Requires [LocalStack](https://localstack.cloud/) and [aws-cdk-local](https://github.com/localstack/aws-cdk-local) installed globally.

LocalStack emulates all AWS services on your local machine, giving you a production-equivalent environment without cloud costs. This is the recommended approach for development and learning.

---

## 🧪 Integration Tests

The `integration-tests/` module contains **end-to-end tests** that validate the full request lifecycle — from the API Gateway through to downstream services.

```bash
cd integration-tests
mvn test
```

These tests cover scenarios such as creating a patient and verifying the billing and analytics side effects.

---

## 🤝 Contributing

Contributions are welcome!

1. Fork this repository
2. Create your feature branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m 'feat: describe your change'`
4. Push to the branch: `git push origin feature/your-feature`
5. Open a Pull Request

Please make sure new code includes appropriate tests and follows existing Spring Boot / Java conventions.

---
## 📄 License

This project is open source and available for educational and personal use.

---

> ⚕️ *A production-grade microservices project demonstrating real-world patterns: REST APIs, gRPC inter-service communication, Kafka event streaming, JWT authentication, Docker containerization, and AWS deployment with Infrastructure as Code.*
