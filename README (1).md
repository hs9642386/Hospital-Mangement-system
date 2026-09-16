# Meridian HMS — Hospital Management System

A full-stack hospital management system built with **React (Vite) + Java Spring Boot + MySQL**, exposed over a **REST API**, with JWT authentication.

Modules included:
- **Authentication** — JWT login, role field (ADMIN / DOCTOR / RECEPTIONIST)
- **Patients** — registration and records
- **Doctors / Faculty allotment** — assign doctors to departments, track status (available / on leave / in surgery / off duty)
- **Bed management** — ward/bed inventory, availability tracking
- **Admissions** — admit a patient (allots a free bed + doctor), discharge (frees the bed), reassign doctor mid-stay
- **Appointments** — OPD scheduling per doctor
- **Billing** — itemized bills, automatic room-charge calculation from admission duration × bed rate, payment status
- **Dashboard** — live stats (patients, beds, admissions, revenue)

---

## Project structure

```
hospital-management-system/
├── backend/            Spring Boot 3 REST API (Java 17, Maven)
│   └── src/main/java/com/hms/
│       ├── entity/      JPA entities
│       ├── repository/  Spring Data repositories
│       ├── service/     Business logic (bed allotment, billing calc, etc.)
│       ├── controller/  REST controllers
│       ├── security/    JWT filter + util
│       ├── config/      Security config, CORS, data seeder
│       ├── dto/         Request/response payloads
│       └── exception/   Global exception handling
└── frontend/           React 19 + Vite SPA
    └── src/
        ├── api/          Axios client + API service functions
        ├── context/       Auth context (JWT session)
        ├── components/    Layout, route guard, shared UI
        ├── pages/         Dashboard, Patients, Doctors, Beds, Admissions, Appointments, Billing, Login
        └── styles/        Design system CSS
```

---

## Prerequisites

- **Java 17+** and **Maven 3.9+**
- **Node.js 18+** and npm
- **MySQL 8+** running locally (or reachable)
- Git

---

## 1. Database setup

Create the database (the app will also auto-create it on first connect if your MySQL user has privileges, via `createDatabaseIfNotExist=true`):

```sql
CREATE DATABASE IF NOT EXISTS hms_db;
```

---

## 2. Backend — Spring Boot

```bash
cd backend
```

Edit `src/main/resources/application.properties` and set your MySQL credentials:

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/hms_db?createDatabaseIfNotExist=true&useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=YOUR_MYSQL_PASSWORD
```

Also change `jwt.secret` to a random 32+ character string before any real deployment.

Run it:

```bash
mvn spring-boot:run
```

On first startup the app automatically seeds:
- **Default admin login:** `admin` / `admin123` (change this immediately after first login — there's no UI for it yet, use `PUT`/direct DB update, or add a "change password" endpoint)
- Sample departments (Cardiology, Orthopedics, General Medicine, Pediatrics)
- Sample beds across General Ward / Semi-Private / Private / ICU

The API will be live at **http://localhost:8080/api**. Spring's `ddl-auto=update` creates/updates all tables automatically — no manual schema scripts needed.

### Quick API sanity check

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"admin123"}'
```

You should get back a JWT token.

---

## 3. Frontend — React

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open **http://localhost:5173**. Log in with `admin` / `admin123`.

If your backend runs anywhere other than `http://localhost:8080/api`, set it via an env file:

```bash
# frontend/.env.local
VITE_API_BASE_URL=http://your-backend-host:8080/api
```

### Production build

```bash
npm run build
```

Outputs static files to `frontend/dist/` — serve with any static host (Nginx, Vercel, Netlify, S3, etc.), pointed at your deployed backend URL via `VITE_API_BASE_URL`.

---

## 4. How the core workflows work

**Faculty (doctor) allotment**
`Doctor` has a `department` foreign key. The Doctors page lets you assign/reassign a doctor to a department inline (`PATCH /api/doctors/{id}/allot-department/{departmentId}`), and set their live status (available, on leave, in surgery, off duty).

**Bed allotment**
`Admission` links a `Patient`, a `Bed`, and (optionally) a `Doctor`. Admitting a patient (`POST /api/admissions/admit`) checks the bed is `AVAILABLE`, then flips it to `OCCUPIED`. Discharging (`PATCH /api/admissions/{id}/discharge`) flips the bed back to `AVAILABLE` and stamps the discharge time. A bed can never be double-booked — the service layer enforces this.

**Billing**
`POST /api/bills/generate` takes a patient, optionally an admission, and a list of manual line items (consultation, medicine, lab test, surgery, other). If linked to an admission, it automatically computes a room-charge line item as `days stayed × bed's daily rate`. Bills can then be marked `PAID` / `PARTIALLY_PAID` / `UNPAID`.

---

## 5. Pushing to GitHub

This project is already a git repository with an initial commit. To push it to your own GitHub:

```bash
cd hospital-management-system
git remote add origin https://github.com/<your-username>/<your-repo>.git
git branch -M main
git push -u origin main
```

---

## 6. Extra backend modules (API-only, no UI yet)

Two modules are already live on the backend but don't have frontend pages yet — call them directly or extend the React app to cover them:

**Medicine inventory** (`/api/medicines`)
- `GET /api/medicines` — list all
- `GET /api/medicines/low-stock` — items at or below reorder level
- `GET /api/medicines/expiring?days=30` — items expiring soon
- `POST /api/medicines` — add a medicine
- `PATCH /api/medicines/{id}/dispense` — deduct stock (auto-fires a low-stock notification if it crosses the threshold)
- `PATCH /api/medicines/{id}/restock` — add stock

**Communication / notifications** (`/api/notifications`)
- Persisted notifications (appointment reminders, bed alerts, billing alerts, emergency alerts, low-stock alerts) that any real channel (email/SMS/in-app) can later be wired to send
- `GET /api/notifications/unread`, `PATCH /api/notifications/{id}/read`

## 7. Roadmap / natural next additions

- **Emergency handling** — a priority flag on `Admission`/`Appointment` for fast-tracking ICU/emergency beds
- **Richer patient records** — visit history, diagnoses, prescriptions, uploaded documents
- **Wire notifications to a real channel** — SMTP or a provider like Twilio for the `Notification` records already being created
- **Frontend pages for Medicine inventory and Notifications** — the API is ready; add `MedicineInventory.jsx` / a notification bell to the sidebar
- **Role-based UI restrictions** — currently all authenticated users see the same UI; the JWT already carries a role claim, so route/component-level guards by role are a small addition
- **Refresh tokens** and a proper "first login must change password" flow

---

## Tech stack summary

| Layer | Tech |
|---|---|
| Frontend | React 19, Vite, React Router, Axios |
| Backend | Spring Boot 3, Spring Security (JWT), Spring Data JPA |
| Database | MySQL 8 |
| Auth | JSON Web Tokens (jjwt) |
| Build tools | Maven (backend), npm/Vite (frontend) |
