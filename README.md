# 🚗 Uber Application – Microservices Backend

A scalable **Uber-like ride-hailing backend system** built using **Java, Spring Boot, Microservices, Apache Kafka, Redis, MySQL, and Docker**.

The application demonstrates a real-world ride-dispatch workflow where riders request rides, real-time driver locations are maintained using Redis GEO, and nearby drivers are identified and matched asynchronously using Kafka-based event-driven communication.

---

## 🏗️ Architecture

The application is divided into three independent microservices:

```text
                         ┌─────────────────┐
                         │    Rider App    │
                         └────────┬────────┘
                                  │
                                  │ REST API
                                  ▼
                     ┌────────────────────────┐
                     │      Ride-Service      │
                     │                        │
                     │ Ride Management        │
                     │ Ride Lifecycle         │
                     │ Kafka Producer/Consumer│
                     └───────────┬────────────┘
                                 │
                         ride.requested
                                 │
                                 ▼
                         ┌───────────────┐
                         │     Kafka     │
                         └───────┬───────┘
                                 │
                                 ▼
                     ┌────────────────────────┐
                     │    Matching-Service    │
                     │                        │
                     │ Driver Matching        │
                     │ Kafka Consumer         │
                     │ OpenFeign Client       │
                     └───────────┬────────────┘
                                 │
                           OpenFeign / REST
                                 │
                                 ▼
                     ┌────────────────────────┐
                     │    Location-Service   │
                     │                        │
                     │ Driver Locations       │
                     │ Nearby Driver Search   │
                     └───────────┬────────────┘
                                 │
                              GEO Query
                                 │
                                 ▼
                           ┌───────────┐
                           │   Redis   │
                           │   GEO     │
                           └───────────┘

                        Persistent Data
                              │
                              ▼
                         ┌─────────┐
                         │  MySQL  │
                         └─────────┘
```

---

## 🚀 Core Features

* 🚕 Ride request and ride lifecycle management
* 📍 Real-time driver location tracking
* 🌎 Geospatial nearby-driver search using Redis GEO
* 🔎 Nearby driver discovery within a configurable radius
* 🤝 Automated driver matching
* ⚡ Event-driven communication using Apache Kafka
* 🔄 Asynchronous Ride-Service ↔ Matching-Service communication
* 🔗 Synchronous Matching-Service → Location-Service communication using OpenFeign
* 🗄️ Persistent relational data management using MySQL
* 🐳 Docker-based containerization
* 🧩 Independent microservices for better scalability and maintainability
* ❤️ Spring Boot Actuator health monitoring

---

## 🔄 End-to-End Ride Flow

### 1. Driver Location Update

Drivers periodically send their current location to the Location-Service.

```text
Driver App
    ↓
Location-Service
    ↓
Redis GEO
    ↓
Driver latitude + longitude
```

Redis GEO allows the system to efficiently find drivers based on geographic proximity.

---

### 2. Rider Requests a Ride

The rider submits a ride request through the Ride-Service.

```text
Rider
  ↓
Ride-Service
  ↓
Create Ride
  ↓
Publish ride.requested
```

The ride request is published to Kafka using the `ride.requested` event.

---

### 3. Matching-Service Receives the Request

Matching-Service consumes the `ride.requested` event.

```text
Kafka
  ↓
RideEventConsumer
  ↓
MatchingService
```

The Matching-Service extracts the pickup coordinates from the ride request.

---

### 4. Find Nearby Drivers

Matching-Service communicates with Location-Service using OpenFeign.

```text
Matching-Service
       ↓
 OpenFeign / REST
       ↓
Location-Service
       ↓
    Redis GEO
       ↓
Nearby Drivers
```

The Location-Service performs a geospatial search and returns nearby available drivers.

The current implementation searches within approximately **5 km** and returns up to **10 nearby drivers**.

---

### 5. Driver Matching

Matching-Service evaluates the available drivers and selects a suitable driver.

The current matching logic considers:

```text
70% → Distance
30% → Driver Rating
```

This provides a simple weighted matching strategy instead of selecting a driver solely based on distance.

---

### 6. Driver Assignment

Once a driver is selected, Matching-Service publishes a:

```text
ride.matched
```

event to Kafka.

```text
Matching-Service
       ↓
ride.matched
       ↓
Kafka
       ↓
Ride-Service
```

Ride-Service consumes the event and updates the ride with the selected driver.

---

## 📡 Event-Driven Communication

Apache Kafka is used to decouple the microservices.

### `ride.requested`

Published by **Ride-Service** when a rider requests a ride.

```text
Ride-Service
     ↓
Kafka
     ↓
Matching-Service
```

### `ride.matched`

Published by **Matching-Service** after selecting a driver.

```text
Matching-Service
     ↓
Kafka
     ↓
Ride-Service
```

This allows the ride and matching workflows to operate independently.

---

## 🧩 Microservices

### 🚕 Ride-Service

Responsible for:

* Creating ride requests
* Managing ride information
* Maintaining ride status
* Publishing `ride.requested`
* Consuming `ride.matched`
* Updating driver assignment

---

### 🤝 Matching-Service

Responsible for:

* Consuming ride requests
* Finding nearby drivers
* Communicating with Location-Service
* Applying driver matching logic
* Selecting a suitable driver
* Publishing `ride.matched`

---

### 📍 Location-Service

Responsible for:

* Receiving driver location updates
* Maintaining real-time driver coordinates
* Performing geospatial searches
* Finding nearby drivers using Redis GEO

---

## 🛠️ Technology Stack

| Technology                 | Usage                                   |
| -------------------------- | --------------------------------------- |
| **Java 17**                | Application development                 |
| **Spring Boot**            | Microservice framework                  |
| **Spring Web / REST**      | REST API development                    |
| **Spring Data JPA**        | Database persistence                    |
| **Hibernate**              | ORM implementation                      |
| **MySQL**                  | Persistent relational data              |
| **Redis**                  | Real-time driver location storage       |
| **Redis GEO**              | Geospatial nearby-driver search         |
| **Apache Kafka**           | Asynchronous event-driven communication |
| **Spring Kafka**           | Kafka producer/consumer integration     |
| **Spring Cloud OpenFeign** | Inter-service REST communication        |
| **ZooKeeper**              | Kafka coordination in the current setup |
| **Lombok**                 | Reducing Java boilerplate               |
| **Spring Boot Actuator**   | Health and monitoring endpoints         |
| **Maven**                  | Dependency management and build         |
| **Docker**                 | Containerization                        |

---

## 🔀 Communication Pattern

The project demonstrates both major microservice communication patterns.

### Asynchronous Communication

Used between Ride-Service and Matching-Service:

```text
Ride-Service
     │
     │ Kafka
     ▼
Matching-Service
```

Events:

```text
ride.requested
ride.matched
```

### Synchronous Communication

Used between Matching-Service and Location-Service:

```text
Matching-Service
       │
       │ OpenFeign / REST
       ▼
Location-Service
```

This combination demonstrates how synchronous and asynchronous communication can coexist within a microservices architecture.

---

## ⚡ Why Redis?

Driver locations are highly dynamic and can change frequently.

Using MySQL as the primary real-time location store would create unnecessary database load for frequent location updates and proximity searches.

Redis provides:

* Fast reads and writes
* In-memory performance
* GEO support
* Efficient proximity searches
* Low-latency access to frequently changing location data

Therefore:

```text
Real-time Driver Location → Redis

Persistent Business Data → MySQL
```

---

## 📨 Why Kafka?

Kafka provides asynchronous communication between Ride-Service and Matching-Service.

Instead of tightly coupling the services:

```text
Ride-Service → Matching-Service
```

the application uses:

```text
Ride-Service
     ↓
Kafka
     ↓
Matching-Service
```

Benefits include:

* Loose coupling
* Asynchronous processing
* Better scalability
* Independent service availability
* Event-driven architecture
* Ability to add additional consumers later

---

## 🐳 Docker

Docker is used to containerize the application and infrastructure components.

A typical local environment can run:

```text
┌─────────────────────────────┐
│          Docker             │
│                             │
│  ┌───────────────────────┐  │
│  │ Ride-Service          │  │
│  ├───────────────────────┤  │
│  │ Matching-Service       │  │
│  ├───────────────────────┤  │
│  │ Location-Service       │  │
│  ├───────────────────────┤  │
│  │ Kafka                  │  │
│  ├───────────────────────┤  │
│  │ ZooKeeper              │  │
│  ├───────────────────────┤  │
│  │ Redis                  │  │
│  ├───────────────────────┤  │
│  │ MySQL                  │  │
│  └───────────────────────┘  │
└─────────────────────────────┘
```

This makes the complete development environment reproducible and easier to run.

---

## 📂 Project Structure

```text
uber-application/
│
├── ride-service/
│   ├── controller/
│   ├── service/
│   ├── dto/
│   ├── entity/
│   ├── event/
│   ├── repository/
│   └── configuration/
│
├── matching-service/
│   ├── controller/
│   ├── service/
│   ├── client/
│   ├── dto/
│   ├── event/
│   └── configuration/
│
├── location-service/
│   ├── controller/
│   ├── service/
│   ├── dto/
│   ├── configuration/
│   └── repository/
│
├── docker-compose.yml
└── README.md
```

---

## 🎯 Project Highlights

* Designed a **3-microservice ride-hailing backend**
* Implemented **event-driven architecture using Kafka**
* Implemented **real-time geospatial driver tracking using Redis GEO**
* Implemented **nearby-driver discovery within a configurable radius**
* Implemented **weighted driver matching based on proximity and rating**
* Implemented **synchronous microservice communication using OpenFeign**
* Used **Spring Data JPA and MySQL** for persistent application data
* Containerized services and infrastructure using **Docker**
* Added Spring Boot Actuator for service health monitoring

---

## 🔮 Possible Future Enhancements

The current project focuses on the core ride-dispatch workflow. A production-scale implementation could be extended with:

* 👤 User/Authentication Service
* 🚘 Driver Management Service
* 💳 Payment Service
* 💰 Dynamic Pricing / Surge Pricing
* 🗺️ Route and Distance Service
* 🔔 Notification Service
* ⭐ Rating and Review Service
* 🛡️ Fraud Detection
* 📊 Trip Analytics
* 🔐 JWT/OAuth2 authentication
* ☸️ Kubernetes deployment
* 🔍 Distributed tracing with OpenTelemetry
* 📈 Prometheus and Grafana monitoring

---

## 🧠 Key Architecture Flow

```text
Rider
  │
  ▼
Ride-Service
  │
  │ ride.requested
  ▼
Kafka
  │
  ▼
Matching-Service
  │
  │ OpenFeign
  ▼
Location-Service
  │
  ▼
Redis GEO
  │
  ▼
Nearby Drivers
  │
  ▼
Matching Algorithm
  │
  │ ride.matched
  ▼
Kafka
  │
  ▼
Ride-Service
  │
  ▼
Driver Assigned
```

---

## 📌 Summary

This project demonstrates the design and implementation of a **real-world-inspired ride-hailing backend using Spring Boot microservices**.

The system combines **Kafka for asynchronous event communication, Redis GEO for real-time driver location and proximity searches, OpenFeign for synchronous service communication, and MySQL for persistent data storage**, while Docker provides a consistent containerized environment.

The architecture is designed to demonstrate key microservices concepts including **service decomposition, event-driven communication, synchronous inter-service communication, caching/in-memory data stores, geospatial queries, persistence, and containerization**.
