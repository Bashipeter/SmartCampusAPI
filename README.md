# Smart Campus — Sensor & Room Management API

A JAX-RS RESTful API built with Jersey 2.32 and Apache Tomcat 9 for managing campus rooms and sensors.

---

## How to Build and Run

### Prerequisites
- Java 8+
- Apache Maven
- Apache Tomcat 9.x
- NetBeans IDE

### Steps

1. **Clone the repository**
```bash
   git clone https://github.com/Bashipeter/SmartCampusAPI.git
   cd SmartCampusAPI
```

2. **Build the project**
```bash
   mvn clean package
```

3. **Deploy to Tomcat**
   - In NetBeans: right-click project → Run
   - Base URL: `http://localhost:8080/SmartCampusAPI/api/v1`

---

## API Overview

| Method | Path | Description |
|--------|------|-------------|
| GET | /api/v1 | Discovery — API metadata |
| GET | /api/v1/rooms | List all rooms |
| POST | /api/v1/rooms | Create a room |
| GET | /api/v1/rooms/{id} | Get room by ID |
| DELETE | /api/v1/rooms/{id} | Delete room (blocked if sensors exist) |
| GET | /api/v1/sensors | List all sensors (supports ?type= filter) |
| POST | /api/v1/sensors | Register a sensor (validates roomId) |
| GET | /api/v1/sensors/{id} | Get sensor by ID |
| GET | /api/v1/sensors/{id}/readings | Get reading history |
| POST | /api/v1/sensors/{id}/readings | Add a new reading |

---

## Sample curl Commands

### 1. Discovery endpoint
```bash
curl -X GET http://localhost:8080/SmartCampusAPI/api/v1
```

### 2. Create a room
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/rooms \
  -H "Content-Type: application/json" \
  -d '{"id":"HALL-01","name":"Main Hall","capacity":200}'
```

### 3. Register a sensor
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/sensors \
  -H "Content-Type: application/json" \
  -d '{"id":"CO2-099","type":"CO2","status":"ACTIVE","currentValue":400.0,"roomId":"LIB-301"}'
```

### 4. Filter sensors by type
```bash
curl -X GET "http://localhost:8080/SmartCampusAPI/api/v1/sensors?type=CO2"
```

### 5. Post a reading
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/sensors/TEMP-001/readings \
  -H "Content-Type: application/json" \
  -d '{"value":24.7}'
```

### 6. Trigger 409 — delete room with sensors
```bash
curl -X DELETE http://localhost:8080/SmartCampusAPI/api/v1/rooms/LIB-301
```

### 7. Trigger 422 — sensor with non-existent roomId
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/sensors \
  -H "Content-Type: application/json" \
  -d '{"id":"TEMP-999","type":"Temperature","status":"ACTIVE","currentValue":0.0,"roomId":"FAKE-999"}'
```

### 8. Trigger 403 — post reading to MAINTENANCE sensor
```bash
curl -X POST http://localhost:8080/SmartCampusAPI/api/v1/sensors/OCC-001/readings \
  -H "Content-Type: application/json" \
  -d '{"value":15.0}'
```

---

## Report — Question Answers

### Part 1.1 — JAX-RS Resource Lifecycle

By default, JAX-RS creates a new instance of each resource class for every incoming HTTP request (request-scoped). This means instance variables are NOT shared between requests. To maintain persistent in-memory state, this application uses a dedicated DataStore class with static ConcurrentHashMap fields. Static fields belong to the class itself, not to any instance, so they survive regardless of how many resource instances are created and destroyed. ConcurrentHashMap is used to prevent race conditions since multiple requests may attempt to read and write simultaneously.

### Part 1.2 — HATEOAS

HATEOAS (Hypermedia As The Engine Of Application State) means API responses include links to related resources, allowing clients to navigate the API dynamically. The benefit is that a client only needs to know one entry point (GET /api/v1) and can discover everything else from the response. This makes the API self-documenting and resilient to change — if a resource path changes, clients following links are unaffected.

### Part 2.1 — ID-only vs Full Objects

Returning only IDs minimises payload size but forces the client to make N additional requests to fetch details, creating an N+1 problem. Returning full objects means one request delivers everything needed, reducing round-trips. For campus management dashboards, returning full objects is the better choice.

### Part 2.2 — Idempotency of DELETE

Yes, DELETE is idempotent. If a room is deleted on the first call (204), a second identical DELETE returns 404. The server state is identical after both calls — the room does not exist. There are no side effects from repeated calls, conforming to the HTTP specification.

### Part 3.1 — @Consumes and Content-Type Mismatches

@Consumes(MediaType.APPLICATION_JSON) tells JAX-RS to only accept requests with Content-Type: application/json. If a client sends text/plain or application/xml, JAX-RS automatically returns HTTP 415 Unsupported Media Type before the method body is ever reached.

### Part 3.2 — @QueryParam vs Path-based Filtering

Query parameters (GET /sensors?type=CO2) are superior because they are optional by nature, composable (combine multiple filters), and do not imply a false resource hierarchy. Path segments identify resources; query strings filter them. Mixing these concerns violates clean resource hierarchy design.

### Part 4.1 — Sub-Resource Locator Pattern

The Sub-Resource Locator pattern delegates handling of nested paths to a dedicated class. Instead of defining every path in one massive controller, a locator method returns a SensorReadingResource instance. Benefits: each class has a single responsibility, the codebase stays maintainable, new sub-resources can be added without touching existing classes, and unit testing is simpler.

### Part 5.2 — Why 422 over 404

HTTP 404 means the URL was not found. HTTP 422 means the server understood the request but the payload contains a semantic error. When a sensor is POSTed with a non-existent roomId, the URL /api/v1/sensors is valid — the problem is inside the JSON body. Returning 404 would mislead the client into thinking the endpoint itself is missing.

### Part 5.4 — Security Risks of Exposing Stack Traces

Exposing stack traces reveals: internal class names and package structure, library names and versions (enabling CVE lookups), server file paths (aiding directory traversal), and business logic flows. The Global Exception Mapper ensures no stack trace reaches the client — all details are logged server-side only.

### Part 5.5 — Filters vs Manual Logging

Using filters is superior because: it follows the DRY principle (one place to maintain), separates concerns (resource methods focus on business logic), guarantees coverage (filters intercept every request automatically), and scales effortlessly as the API grows with zero additional work per new endpoint.
