# Fitness Tracker API

A RESTful backend for a fitness tracking application, built with Spring Boot. Users can register, log in, track workouts, and receive recommendations tied to their activities.

Built as a **monolith** (a single deployable service) using a clean, layered architecture with JWT-based stateless authentication and a MySQL database.

🔗 **Live demo:** https://fitness-tracker-api-m1af.onrender.com
📘 **API docs (Swagger UI):** https://fitness-tracker-api-m1af.onrender.com/swagger-ui.html

> This is hosted on a free tier, so the app spins down after ~15 minutes of inactivity. The first request after that may take up to 50 seconds to respond while it wakes up — this is a hosting limitation, not a bug.

---

## Table of Contents

- [Tech Stack](#tech-stack)
- [Features](#features)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [How a Request Flows Through the App](#how-a-request-flows-through-the-app)
- [API Endpoints](#api-endpoints)
- [Running Locally](#running-locally)
- [Running with Docker](#running-with-docker)
- [Deployment](#deployment)
- [License](#license)

---

## Tech Stack

| Category | Technology |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 4.0 |
| Security | Spring Security, JWT (jjwt) |
| Database | MySQL |
| ORM | Spring Data JPA / Hibernate |
| API Documentation | springdoc-openapi (Swagger UI) |
| Build Tool | Maven |
| Containerization | Docker (multi-stage build) |
| Hosting | Render (application) + Aiven (MySQL database) |

---

## Features

- **Authentication** — user registration and login, passwords hashed with BCrypt, JWT issued on successful login
- **Activity tracking** — log workouts with type, duration, calories, and flexible metadata (a JSON column) for activity-specific fields like distance or heart rate
- **Recommendations** — attach structured feedback (improvements, suggestions, safety notes) to a specific logged activity
- **Role-based access control** — the JWT carries the user's role, and certain routes are restricted accordingly
- **Stateless security** — no server-side sessions; every request is authenticated independently via a Bearer token
- **Interactive API documentation** — Swagger UI generated automatically from the codebase

---

## Architecture

The application follows a standard **layered architecture**, which keeps each part of the codebase focused on a single responsibility:

```
┌─────────────────────────────────────────────────────────┐
│                        Client                            │
│              (browser, Postman, mobile app)               │
└───────────────────────────┬───────────────────────────────┘
                             │ HTTP request (+ JWT if protected)
                             ▼
┌─────────────────────────────────────────────────────────┐
│                 JwtAuthenticationFilter                  │
│   Reads the Bearer token, validates it, and sets the      │
│   authenticated user in Spring Security's context         │
└───────────────────────────┬───────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────┐
│                       SecurityConfig                      │
│     Decides which routes are public vs. protected vs.     │
│                    admin-only                              │
└───────────────────────────┬───────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────┐
│                        Controller                          │
│   Receives the HTTP request, converts JSON → DTO           │
│   (AuthController, ActivityController,                     │
│    RecommendationController)                                │
└───────────────────────────┬───────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────┐
│                         Service                             │
│         All business logic lives here                       │
│   (UserService, ActivityService, RecommendationService)     │
└───────────────────────────┬───────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────┐
│                       Repository                            │
│    Spring Data JPA interfaces — save/find/delete with       │
│                     no manual SQL                            │
└───────────────────────────┬───────────────────────────────┘
                             ▼
┌─────────────────────────────────────────────────────────┐
│                      MySQL Database                         │
│         Users · Activities · Recommendations                │
└─────────────────────────────────────────────────────────┘
```

**Why this structure?** Each layer only knows about the layer directly below it. Controllers never talk to the database directly, and services never deal with HTTP details. This makes the codebase easier to test, easier to change, and is the standard pattern expected in any professional Spring Boot codebase.

---

## Project Structure

```
fitness-monolith/
│
├── Dockerfile                          # Multi-stage build: compiles the jar, then runs it on a slim JRE image
├── pom.xml                             # Maven project config and dependencies
├── mvnw / mvnw.cmd                     # Maven wrapper (build without installing Maven locally)
│
└── src/
    ├── main/
    │   ├── java/com/project/fitness/
    │   │   ├── FitnessMonolithApplication.java     # Application entry point
    │   │   │
    │   │   ├── config/
    │   │   │   └── OpenApiConfig.java               # Swagger / OpenAPI documentation setup
    │   │   │
    │   │   ├── controller/                          # HTTP layer — handles requests & responses
    │   │   │   ├── AuthController.java               #   /api/auth/**   (register, login)
    │   │   │   ├── ActivityController.java            #   /api/activities/**
    │   │   │   └── RecommendationController.java      #   /api/recommendation/**
    │   │   │
    │   │   ├── dto/                                  # Data Transfer Objects — request/response shapes
    │   │   │   ├── RegisterRequest.java
    │   │   │   ├── LoginRequest.java
    │   │   │   ├── LoginResponse.java
    │   │   │   ├── UserResponse.java
    │   │   │   ├── ActivityRequest.java
    │   │   │   ├── ActivityResponse.java
    │   │   │   └── RecommendationRequest.java
    │   │   │
    │   │   ├── exceptions/
    │   │   │   └── GlobalExceptionHandler.java        # Turns exceptions into clean JSON error responses
    │   │   │
    │   │   ├── model/                                 # JPA entities — map directly to database tables
    │   │   │   ├── User.java
    │   │   │   ├── UserRole.java
    │   │   │   ├── Activity.java
    │   │   │   ├── ActivityType.java
    │   │   │   └── Recommendation.java
    │   │   │
    │   │   ├── repository/                            # Spring Data JPA interfaces (no SQL written by hand)
    │   │   │   ├── UserRepository.java
    │   │   │   ├── ActivityRepository.java
    │   │   │   └── RecommendationRepository.java
    │   │   │
    │   │   ├── security/                              # Everything related to authentication & authorization
    │   │   │   ├── JwtUtils.java                       #   Generates & validates JWTs
    │   │   │   ├── JwtAuthenticationFilter.java         #   Runs on every request, checks the token
    │   │   │   ├── CustomUserDetailsService.java        #   Loads user data for Spring Security
    │   │   │   └── SecurityConfig.java                  #   Route access rules (public / protected / admin)
    │   │   │
    │   │   └── service/                                # Business logic layer
    │   │       ├── UserService.java
    │   │       ├── ActivityService.java
    │   │       └── RecommendationService.java
    │   │
    │   └── resources/
    │       └── application.properties                  # DB connection, JWT config, logging levels
    │
    └── test/
        └── java/com/project/fitness/
            └── FitnessMonolithApplicationTests.java     # Base Spring context test
```

---

## How a Request Flows Through the App

Example: **logging a new workout** (`POST /api/activities`)

1. **Client sends the request** with a JWT in the `Authorization: Bearer <token>` header and a JSON body (activity type, duration, calories).
2. **`JwtAuthenticationFilter`** intercepts it before it reaches any controller — it validates the token and tells Spring Security who's making the request.
3. **`SecurityConfig`** checks whether this route requires authentication (it does) — since the token is valid, the request proceeds.
4. **`ActivityController`** receives the request and converts the JSON body into an `ActivityRequest` DTO.
5. **`ActivityService`** contains the actual logic — it looks up the authenticated user, builds an `Activity` entity, and saves it.
6. **`ActivityRepository`** (Spring Data JPA) translates that save into a real SQL `INSERT` against MySQL.
7. **The response flows back up** — the saved entity is converted into an `ActivityResponse` DTO (so internal database fields are never leaked to the client) and returned as JSON.

If anything fails along the way — an invalid token, bad input, or a missing resource — `GlobalExceptionHandler` catches it and returns a clean, structured error response instead of a raw stack trace.

---

## API Endpoints

| Method | Endpoint | Description | Auth required |
|---|---|---|---|
| POST | `/api/auth/register` | Register a new user | No |
| POST | `/api/auth/login` | Log in and receive a JWT | No |
| GET | `/api/activities` | Get the logged-in user's activities | Yes |
| POST | `/api/activities` | Log a new activity | Yes |
| POST | `/api/recommendation/generate` | Attach a recommendation to an activity | Yes |
| GET | `/api/recommendation/user/{userId}` | Get recommendations for a user | Yes |
| GET | `/api/recommendation/activity/{activityId}` | Get recommendations for an activity | Yes |

Full request/response schemas, and the ability to try each endpoint live, are available in Swagger UI at `/swagger-ui.html`.

---

## Running Locally

### Prerequisites
- Java 17+
- Maven (or use the included `mvnw` wrapper — no separate install needed)
- A MySQL database (local or hosted)

### Setup

1. Clone the repo:
   ```bash
   git clone https://github.com/shridharsinghh/Fitness-Tracker.git
   cd Fitness-Tracker
   ```

2. Set the required environment variables:
   ```
   DB_URL=jdbc:mysql://<host>:<port>/<database>
   DB_USER=<your-db-username>
   DB_PWD=<your-db-password>
   JWT_SECRET=<any-random-32+-character-string>
   ```

3. Run it:
   ```bash
   ./mvnw spring-boot:run
   ```

4. The app is now available at `http://localhost:8080`, with Swagger UI at `http://localhost:8080/swagger-ui.html`.

---

## Running with Docker

The `Dockerfile` uses a **multi-stage build**: the first stage compiles the project with Maven, and the second stage runs the resulting jar on a lightweight JRE image — so the final image doesn't carry the full JDK or build tools.

```bash
docker build -t fitness-tracker .

docker run -p 8080:8080 \
  -e DB_URL=jdbc:mysql://<host>:<port>/<database> \
  -e DB_USER=<your-db-username> \
  -e DB_PWD=<your-db-password> \
  -e JWT_SECRET=<your-secret> \
  fitness-tracker
```

---

## Deployment

This project is deployed on **Render** as a Docker-based web service, connected to a **MySQL** database hosted on **Aiven**. Environment variables (`DB_URL`, `DB_USER`, `DB_PWD`, `JWT_SECRET`) are configured directly in the hosting environment — never committed to source control.

```
GitHub (source) → Render (build + host) → Aiven (MySQL database)
```

Every push to `main` automatically triggers a new build and deploy on Render.

---

## License

This project is for educational and portfolio purposes.
