# Cloud-Native Java Microservices

A hands-on backend microservices project built with **Java, Spring Boot, Kafka, Docker, Kubernetes, and AWS**.

The project demonstrates how a microservices-based application can evolve from a local development environment to a cloud deployment using **AWS ECS/Fargate, ECR, Application Load Balancer, Amazon RDS, and ECS Service Connect**.

The project also includes **API Gateway, Keycloak, JWT-based authentication, Kubernetes orchestration, Prometheus, Grafana, and alerting** for the local/Kubernetes environment.

---

## Architecture

### Local / Kubernetes Architecture

```text
                         Client
                           |
                           | JWT
                           v
                       Keycloak
                           |
                           v
                 API Gateway :8080
                    /             \
                   /               \
                  v                 v
        Product Service :8081   Order Service :8082
                |                    |
                v                    |
           MySQL :3306               |
                                     |
                         +-----------+-----------+
                         |                       |
                         v                       v
                  Product Service             Kafka
                    (REST)                      |
                                                v
                                            Consumers

              Kubernetes
                   |
                   +---- Prometheus
                   |
                   +---- Grafana
                            |
                            v
                         Alerts
```

### AWS Deployment Architecture

The AWS deployment focuses on running the backend services using managed AWS infrastructure where appropriate, while Kafka is currently self-managed on ECS/Fargate.

```text
                         Internet
                            |
                            v
                 Application Load Balancer
                         HTTP :80
                            |
                            v
                    Order Service
                     ECS / Fargate
                         :8082
                      /          \
                     /            \
                    v              v
          Product Service         Kafka
           ECS / Fargate       ECS / Fargate
               :8081              :9092
                    \              /
                     \            /
                      v          v
                         Amazon RDS
                          MySQL
                    product_db / order_db

          ECR
           |
           +---- Product Service image
           +---- Order Service image
           +---- Kafka image

        ECS Service Connect
        service discovery between services
```

> **Note:** The AWS deployment is currently a learning environment. The application was deployed and end-to-end tested, and resources are scaled down/stopped when not in use to control costs.

---

# Components

## API Gateway

The API Gateway acts as the external entry point for the local/Kubernetes application.

Responsibilities:

* Single entry point for backend services
* Routing requests to Product and Order services
* JWT authentication
* Role-based authorization
* OAuth2 Resource Server integration
* Reactive gateway using Spring WebFlux

### Example Routes

```text
/products/**  → Product Service :8081
/orders/**    → Order Service :8082
```

---

## Authentication & Authorization

Authentication is implemented using **Keycloak** as the Identity Provider.

The flow is:

```text
Client
   |
   v
Keycloak
   |
   | JWT Access Token
   v
Client
   |
   v
API Gateway
   |
   +-- Validate JWT
   +-- Extract roles
   +-- Authorize request
   |
   v
Microservices
```

Current authorization concepts include:

* JWT-based authentication
* Role-based access control
* `USER` and `ADMIN` roles
* Mapping Keycloak `realm_access.roles` to Spring Security authorities
* Protected operations based on roles
* `401 Unauthorized` for invalid/missing authentication
* `403 Forbidden` for insufficient permissions

---

# Product Service

A Spring Boot REST service responsible for product management.

### Features

* REST APIs
* CRUD operations
* MySQL persistence
* Spring Data JPA / Hibernate
* Request validation
* Exception handling
* Spring Boot Actuator
* Prometheus metrics

### Port

```text
8081
```

---

# Order Service

A Spring Boot REST service responsible for order management.

### Features

* REST APIs
* Order creation and retrieval
* MySQL persistence
* Spring Data JPA / Hibernate
* Synchronous communication with Product Service
* Kafka event publishing
* Kafka event consumption
* Spring Boot Actuator
* Prometheus metrics

### Port

```text
8082
```

---

# Service-to-Service Communication

The project demonstrates both synchronous and asynchronous communication patterns.

## Synchronous Communication

The Order Service communicates with the Product Service using REST.

```text
Order Service
      |
      | HTTP REST
      v
Product Service
```

This is used when the Order Service requires an immediate response from the Product Service.

---

## Asynchronous Communication

Order events are published to Kafka.

```text
Order Service
      |
      | OrderCreatedEvent
      v
    Kafka
      |
      v
   Consumer
```

This demonstrates event-driven communication where the producer does not need to wait for downstream processing to complete.

---

# Kafka

Apache Kafka is used for asynchronous, event-driven communication.

The project demonstrates:

* Kafka producers
* Kafka consumers
* Topics
* Consumer groups
* Partitions
* Offsets
* Asynchronous event processing
* Order-created events

### Example Flow

```text
POST /orders
      |
      v
Order Service
      |
      | Publish OrderCreatedEvent
      v
Kafka
      |
      | Consume event
      v
Kafka Consumer
```

### AWS Kafka Deployment

For the current AWS learning environment, Kafka is deployed as **self-managed Apache Kafka on ECS/Fargate**.

AWS MSK is not currently used.

---

# Docker

The application and supporting infrastructure can be run using Docker.

Docker is used for:

* Product Service
* Order Service
* API Gateway
* Kafka
* MySQL
* Keycloak

Docker Compose is provided for local development and testing.

The application images used for AWS ECS/Fargate are built for the target **Linux AMD64** architecture.

---

# Kubernetes

Kubernetes is used as the local container orchestration environment.

The project includes Kubernetes configurations for:

* Product Service
* Order Service
* Kafka
* MySQL
* Monitoring

The Kubernetes setup demonstrates:

* Deployments
* Services
* NodePort
* Configuration
* Service discovery
* Container orchestration

---

# AWS Deployment

The project has been deployed and tested on AWS using the following services:

| AWS Service                   | Purpose                                                  |
| ----------------------------- | -------------------------------------------------------- |
| **Amazon ECS / Fargate**      | Runs Product Service, Order Service and Kafka containers |
| **Amazon ECR**                | Stores Docker images                                     |
| **Application Load Balancer** | Public HTTP entry point for Order Service                |
| **Amazon RDS for MySQL**      | Managed relational database                              |
| **ECS Service Connect**       | Service discovery and service-to-service communication   |
| **Amazon CloudWatch Logs**    | ECS container/application logs                           |
| **Default VPC**               | Network environment for the learning deployment          |

---

## AWS Request Flow

The current AWS deployment was tested using the following flow:

```text
Client
  |
  | HTTP :80
  v
Application Load Balancer
  |
  | HTTP :8082
  v
Order Service
  |
  +----------------------+
  |                      |
  | REST                 | Kafka Event
  v                      v
Product Service         Kafka
  |                      |
  +----------+-----------+
             |
             v
        Amazon RDS
          MySQL
```

The Order Service was successfully tested through the ALB for:

```text
GET  /orders
POST /orders
```

The application was also tested end-to-end with:

```text
Order Service
      |
      +---- REST ----> Product Service
      |
      +---- Kafka ---> Kafka Consumer
      |
      +---- MySQL ---> Amazon RDS
```

---

# AWS Networking

The deployment uses the AWS VPC networking model with ECS tasks, RDS, and the Application Load Balancer inside the same VPC.

The current security-group design includes:

```text
Internet
   |
   | TCP :80
   v
ALB Security Group
   |
   | TCP :8082
   v
Order Service Security Group
   |
   +---- Product Service :8081
   |
   +---- Kafka :9092
   |
   +---- RDS MySQL :3306
```

Amazon RDS is configured without public access.

ECS Service Connect provides service discovery for application services such as:

```text
product-service:8081
kafka:9092
```

---

# AWS Container Registry

Docker images are stored in Amazon ECR before being deployed to ECS/Fargate.

Example flow:

```text
Source Code
    |
    v
Docker Build
    |
    v
AMD64 Docker Image
    |
    v
Amazon ECR
    |
    v
ECS / Fargate
```

---

# AWS Configuration

The application supports environment-based configuration.

For example:

```properties
spring.datasource.url=${SPRING_DATASOURCE_URL:jdbc:mysql://localhost:3306/order_db}
spring.datasource.username=${SPRING_DATASOURCE_USERNAME:appuser}
spring.datasource.password=${SPRING_DATASOURCE_PASSWORD:app_password}

spring.kafka.bootstrap-servers=${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}
```

This allows the same application to use local infrastructure during development and AWS resources during deployment.

---

# Monitoring

The local/Kubernetes environment includes:

* Spring Boot Actuator
* Micrometer
* Prometheus
* Grafana
* ServiceMonitor
* HTTP metrics
* JVM metrics

Example metrics include:

* HTTP request count
* HTTP error rate
* Average response time
* JVM heap usage
* CPU-related metrics

The AWS deployment currently uses **CloudWatch Logs** for ECS container/application logs.

---

# Alerting

The Kubernetes monitoring environment includes Grafana alerting.

An example alert monitors the Product Service HTTP error rate.

Example condition:

```text
HTTP error rate > 5%
for 2 minutes
```

The alert lifecycle was tested through:

```text
Normal
   ↓
Pending
   ↓
Firing
   ↓
Normal
```

---

# Security Flow

For the local/Kubernetes application:

```text
Client
  |
  | Login
  v
Keycloak
  |
  | JWT
  v
Client
  |
  v
API Gateway
  |
  +-- Validate JWT
  |
  +-- Extract roles
  |
  +-- Authorize request
  |
  v
Microservices
```

---

# Project Structure

```text
cloud-native-java-microservices/
│
├── product-service/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── order-service/
│   ├── src/
│   ├── Dockerfile
│   └── pom.xml
│
├── api-gateway/
│   ├── src/
│   └── pom.xml
│
├── kubernetes/
│   ├── product-service/
│   ├── order-service/
│   ├── kafka/
│   ├── mysql/
│   └── monitoring/
│
├── docker-compose.yml
├── README.md
└── .gitignore
```

AWS-specific task-definition JSON files and local MySQL setup files are intentionally excluded from the repository because they contain environment-specific configuration.

---

# Local Monitoring Access

When the Kubernetes environment is running, Grafana can be accessed using port forwarding:

```bash
kubectl port-forward service/grafana 3000:3000
```

Prometheus:

```bash
kubectl port-forward service/prometheus 9090:9090
```

Then open:

```text
Grafana:
http://localhost:3000

Prometheus:
http://localhost:9090
```

---

# Kubernetes Monitoring Flow

```text
Application
     |
     v
Actuator / Micrometer
     |
     v
Prometheus
     |
     v
Grafana
     |
     v
Alerts
```

---

# Technology Stack

| Technology                      | Purpose                            |
| ------------------------------- | ---------------------------------- |
| **Java 25**                     | Application development            |
| **Spring Boot**                 | Microservices framework            |
| **Spring Cloud Gateway**        | API Gateway                        |
| **Spring Security**             | Authentication and authorization   |
| **Keycloak**                    | Identity and access management     |
| **Spring Data JPA / Hibernate** | Persistence                        |
| **MySQL**                       | Relational database                |
| **Apache Kafka**                | Event-driven communication         |
| **Docker**                      | Containerization                   |
| **Docker Compose**              | Local development                  |
| **Kubernetes**                  | Container orchestration            |
| **AWS ECS / Fargate**           | Cloud container deployment         |
| **Amazon ECR**                  | Container image registry           |
| **Application Load Balancer**   | HTTP traffic routing               |
| **Amazon RDS**                  | Managed MySQL database             |
| **ECS Service Connect**         | Service discovery                  |
| **CloudWatch Logs**             | AWS container/application logs     |
| **Prometheus**                  | Metrics collection                 |
| **Grafana**                     | Metrics visualization and alerting |
| **Maven**                       | Build and dependency management    |
| **Git / GitHub**                | Source control                     |

---

# Project Goals

This project is being developed as a practical learning project covering:

* Java backend development
* Spring Boot microservices
* REST API design
* Synchronous service communication
* Event-driven architecture
* Apache Kafka
* API Gateway
* JWT authentication
* Keycloak
* Role-based authorization
* Docker
* Kubernetes
* Service discovery
* Prometheus
* Grafana
* Alerting
* AWS ECS/Fargate
* Amazon ECR
* Application Load Balancer
* Amazon RDS
* ECS Service Connect
* CloudWatch Logs
* Cloud-native deployment concepts

The project is intentionally being evolved incrementally, with additional AWS architecture, security, observability, automation, and scalability improvements planned.

---

# Planned AWS Improvements

The following areas are planned for future iterations:

* Move ECS tasks to private subnets / remove public IP exposure
* AWS Secrets Manager or Systems Manager Parameter Store
* ECS Service Auto Scaling
* HTTPS using AWS Certificate Manager
* GitHub Actions → ECR → ECS CI/CD
* More comprehensive CloudWatch monitoring
* AWS-native Kafka evaluation, including Amazon MSK
* Infrastructure as Code using Terraform or AWS CDK
* Improved production-style security and networking

These features are **planned and are not currently represented as completed implementations**.

---

# Current AWS Learning Outcome

The AWS phase of this project has provided hands-on experience with:

```text
Docker
   ↓
Amazon ECR
   ↓
Amazon ECS / Fargate
   ↓
ECS Service Connect
   ↓
Application Load Balancer
   ↓
Amazon RDS
   ↓
Apache Kafka
   ↓
CloudWatch Logs
```

The deployment has also provided practical experience with:

* VPC networking
* Security groups
* ECS task definitions
* ECS services
* Fargate networking
* Container architecture compatibility
* ECR image deployment
* ALB target groups and health checks
* RDS connectivity
* Service discovery
* Kafka producer/consumer communication
* Troubleshooting distributed application connectivity

---

# Future Direction

The project will continue evolving toward a more production-oriented cloud-native architecture while keeping the implementation grounded in practical backend engineering concepts.

More components and improvements will be added as the project evolves.
