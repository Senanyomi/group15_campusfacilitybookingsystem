# Facility Booking — Architecture & API Reference

## 1. MVC Architecture

The application is structured around the **Model-View-Controller (MVC)** pattern, with a Laravel (PHP) backend serving a decoupled React frontend.

### How a Request Flows Through the System

When a user performs an action — say, clicking **Book Now** in the React interface — the following sequence takes place:

1. **View (Frontend):** The React layer captures the user's input and sends an HTTP request through Axios to the appropriate backend API endpoint.
2. **Controller:** Laravel's router receives the request and hands it off to the relevant Controller (e.g., `BookingController@store`). The Controller acts as the central coordinator — it validates the incoming data (checking that dates and times are well-formed) and enforces business rules (such as preventing double-bookings).
3. **Model:** To carry out its logic, the Controller works through Eloquent Models (`Booking`, `Facility`, `User`, etc.). Each model maps to a database table and manages data persistence, retrieval, and relationship resolution (e.g., a `Booking` belongs to both a `Facility` and a `User`).
4. **Response:** Once the Model completes its operation, it returns the result to the Controller, which serialises it as a JSON response and sends it back to the React frontend to update the UI.

---

## 2. API Endpoints

All endpoints return JSON and follow standard HTTP status codes (`200 OK`, `201 Created`, `401 Unauthorized`, `409 Conflict`). Protected routes require a valid Bearer token issued by Laravel Sanctum.

---

### Authentication

#### Register a New User
- **`POST /api/auth/register`**
- **Description:** Creates a new standard user account.
- **Request Body (JSON):**
  ```json
  {
    "name": "Jane Doe",
    "email": "jane@example.com",
    "password": "password"
  }
  ```
- **Success (`201 Created`):**
  ```json
  {
    "user": { "id": 1, "name": "Jane Doe", "email": "jane@example.com", "role": "user" },
    "token": "1|abc123def456..."
  }
  ```

#### Log In
- **`POST /api/auth/login`**
- **Description:** Authenticates a user and returns an API token.
- **Request Body (JSON):**
  ```json
  {
    "email": "jane@example.com",
    "password": "password"
  }
  ```
- **Success (`200 OK`):** Returns the user object and a Bearer token.
- **Failure (`401 Unauthorized`):** `{ "message": "Invalid credentials." }`

---

### Facilities (Rooms)

#### List All Facilities
- **`GET /api/facilities`**
- **Description:** Public endpoint that returns all available rooms.
- **Optional Query Parameter:** `?building_id=1` — filters results to a specific building.
- **Success (`200 OK`):**
  ```json
  [
    {
      "id": 1,
      "name": "Conference Room A",
      "capacity": 20,
      "building": { "id": 1, "name": "Main Office" }
    }
  ]
  ```

#### Get Available Time Slots for a Room
- **`GET /api/facilities/{id}/slots?date=YYYY-MM-DD`**
- **Description:** Returns all 30-minute booking slots between 06:00 and 22:00 for a given room on a given date, each marked as available or booked.
- **Success (`200 OK`):**
  ```json
  [
    { "start": "06:00", "end": "06:30", "status": "available" },
    { "start": "06:30", "end": "07:00", "status": "booked" }
  ]
  ```

---

### Bookings

> **Authentication required** for all booking endpoints.

#### Get Bookings
- **`GET /api/bookings`**
- **Description:** Returns bookings scoped to the caller. Standard users see only their own bookings; Admins receive all bookings system-wide.
- **Success (`200 OK`):** Array of Booking objects.

#### Create a Booking
- **`POST /api/bookings`**
- **Description:** Reserves a facility for a specific time window. Validates availability at the point of creation to prevent double-bookings.
- **Request Body (JSON):**
  ```json
  {
    "facility_id": 1,
    "user_id": 2,
    "date": "2026-10-15",
    "start_time": "14:00",
    "end_time": "15:00"
  }
  ```
- **Success (`201 Created`):** Returns the newly created Booking object.
- **Conflict (`409 Conflict`):** `{ "message": "The facility is already booked for this time slot." }`

#### Cancel a Booking
- **`DELETE /api/bookings/{id}`**
- **Description:** Soft-cancels a booking by setting its status to `cancelled`. The record is retained in the database rather than deleted outright.
- **Success (`200 OK`):** Returns the updated Booking object.

---

### Admin Operations

> **Admin role required** for all endpoints in this section.

#### Dashboard Statistics
- **`GET /api/admin/stats`**
- **Description:** Returns an aggregated summary of system activity. Results are heavily cached to keep response times low.
- **Success (`200 OK`):**
  ```json
  {
    "total_bookings": 150,
    "total_facilities": 12,
    "open_complaints": 3,
    "recent_bookings": [ "..." ]
  }
  ```

#### Create Buildings & Facilities
- **`POST /api/buildings`** / **`POST /api/facilities`**
- **Description:** Creates a new building or room in the system. On success, the relevant read-caches are automatically invalidated so users see the new entry without delay.
- **Success (`201 Created`):** Returns the created resource object.
