# Fitness Tracker API

A RESTful backend for a fitness tracking application, built with Spring Boot. Users can register, log in, track workouts, and receive recommendations tied to their activities — built with a layered architecture, JWT-based stateless authentication, and a MySQL database.

**Live demo:** https://fitness-tracker-api-m1af.onrender.com
**API docs (Swagger):** https://fitness-tracker-api-m1af.onrender.com/swagger-ui.html

> Note: hosted on a free tier — the app spins down after inactivity, so the first request after idle time may take ~50 seconds to respond.

---

## Tech Stack

- **Language:** Java 17
- **Framework:** Spring Boot 4.0
- **Security:** Spring Security + JWT (jjwt)
- **Database:** MySQL (via Spring Data JPA / Hibernate)
- **API Docs:** springdoc-openapi (Swagger UI)
- **Build Tool:** Maven
- **Containerization:** Docker (multi-stage build)
- **Hosting:** Render (app) + Aiven (MySQL)

---

## Features

- **Authentication** — user registration and login with BCrypt password hashing and JWT issuance
- **Activity tracking** — log workouts with type, duration, calories, and flexible metadata (JSON column) for activity-specific fields like distance or heart rate
- **Recommendations** — attach structured feedback (improvements, suggestions, safety notes) to a logged activity
- **Role-based access** — JWT carries user role; certain routes are restricted accordingly
- **Stateless security** — no server-side sessions; every request is authenticated via a Bearer token
- **API documentation** — interactive Swagger UI generated from the codebase

---

## Architecture

```
Controller → Service → Repository → Database
```

- **Controller layer** — handles HTTP requests/responses (`AuthController`, `ActivityController`, `RecommendationController`)
- **Service layer** — business logic (`UserService`, `ActivityService`, `RecommendationService`)
- **Repository layer** — Spring Data JPA interfaces for persistence
- **Security layer** — `JwtUtils` (token generation/validation), `JwtAuthenticationFilter` (per-request auth), `SecurityConfig` (route access rules)

---

## API Endpoints

| Method | Endpoint | Description | Auth required |
|--------|----------|-------------|----------------|
| POST | `/api/auth/register` | Register a new user | No |
| POST | `/api/auth/login` | Log in and receive a JWT | No |
| GET | `/api/activities` | Get the logged-in user's activities | Yes |
| POST | `/api/activities` | Log a new activity | Yes |
| POST | `/api/recommendation/generate` | Attach a recommendation to an activity | Yes |
| GET | `/api/recommendation/user/{userId}` | Get recommendations for a user | Yes |
| GET | `/api/recommendation/activity/{activityId}` | Get recommendations for an activity | Yes |

Full request/response schemas are available in Swagger UI at `/swagger-ui.html`.

---

## Running Locally

### Prerequisites
- Java 17+
- Maven
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

4. The app will be available at `http://localhost:8080`, with Swagger UI at `http://localhost:8080/swagger-ui.html`.

### Running with Docker

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

This project uses a multi-stage Dockerfile (Maven build stage → lightweight JRE runtime stage) and is deployed on **Render** (Docker-based web service), connected to a **MySQL** database hosted on **Aiven**. Environment variables (`DB_URL`, `DB_USER`, `DB_PWD`, `JWT_SECRET`) are configured directly in the hosting environment rather than committed to source control.

---

## License

This project is for educational and portfolio purposes.
