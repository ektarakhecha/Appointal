# Appointal – Hospital Appointment Management System

## Introduction
Appointal is a small web application for managing doctors, patients and appointments in a hospital. The backend is built with Java and Spring Boot, the data is stored in MongoDB, and the frontend uses plain HTML, CSS and JavaScript.

## Objective
To demonstrate:
1. Development of a backend application using MongoDB
2. CRUD operations
3. Queries and indexes for good performance
4. Replication and backup strategies
5. Deployment and testing

## Features
- Dashboard with total doctors, patients and appointments
- Doctors: add, view, update, delete, search by name, filter by specialization
- Patients: add, view, update, delete, search by name
- Appointments: book, view, update, cancel, delete; filter by status and date; search by doctor/patient
- Duplicate booking prevention (same doctor + date + time)

## Technology Stack
| Layer | Technology |
|---|---|
| Frontend | HTML, CSS, basic JavaScript (fetch) |
| Backend | Java 17+, Spring Boot 3 |
| Database | MongoDB (Atlas) |
| Build tool | Maven |

## Project Structure
```
appointal/
├── pom.xml
├── .env.example
├── README.md, BACKUP_AND_RECOVERY.md, REPLICATION.md
└── src/main/
    ├── java/com/appointal/
    │   ├── AppointalApplication.java
    │   ├── controller/   (REST endpoints)
    │   ├── model/        (Doctor, Patient, Appointment)
    │   ├── repository/   (database queries)
    │   └── service/      (business logic)
    └── resources/
        ├── application.properties
        └── static/       (HTML, CSS, JS pages)
```
Flow of a request: **Browser (fetch) → Controller → Service → Repository → MongoDB**

## MongoDB Collections
| Collection | Fields |
|---|---|
| doctors | id, name, specialization, phone, email, availableDays, consultationFee |
| patients | id, name, age, gender, phone, email |
| appointments | id, patientId, doctorId, patientName, doctorName, appointmentDate, appointmentTime, reason, status |

Database name: `appointal`. Appointments store the patient and doctor name as well as their ids, so the appointment list can be shown without extra lookups.

## CRUD Operations
| Operation | Doctors | Patients | Appointments |
|---|---|---|---|
| Create | POST /api/doctors | POST /api/patients | POST /api/appointments |
| Read all | GET /api/doctors | GET /api/patients | GET /api/appointments |
| Read one | GET /api/doctors/{id} | GET /api/patients/{id} | GET /api/appointments/{id} |
| Update | PUT /api/doctors/{id} | PUT /api/patients/{id} | PUT /api/appointments/{id} |
| Delete | DELETE /api/doctors/{id} | DELETE /api/patients/{id} | DELETE /api/appointments/{id} |

## Queries
Spring Data MongoDB creates the query from the method name.

| Repository method | What it does | Used by |
|---|---|---|
| `findByNameContainingIgnoreCase(name)` (Doctor) | Finds doctors whose name contains the text (like SQL LIKE) | GET /api/doctors?name=ravi |
| `findBySpecializationIgnoreCase(spec)` | Finds doctors of one specialization | GET /api/doctors?specialization=Cardiology |
| `findByNameContainingIgnoreCase(name)` (Patient) | Finds patients by part of the name | GET /api/patients?name=asha |
| `findByStatus(status)` | Finds appointments with a given status | GET /api/appointments?status=Scheduled |
| `findByAppointmentDate(date)` | Finds appointments on a date | GET /api/appointments?date=2026-11-10 |
| `findByDoctorId(id)` | Finds all appointments of a doctor | GET /api/appointments?doctorId=... |
| `findByDoctorId...StatusNot(...)` | Finds non-cancelled appointments in the same slot (duplicate check) | Booking and updating |

## Indexes
**What is an index?** A sorted "table of contents" for a collection, kept separately from the data.

**Why use one?** Without an index MongoDB must read every document to find a match (a *collection scan*). With an index it jumps almost directly to the matching documents.

**How does it improve performance?** Searching 100,000 appointments by status would normally check all 100,000. With an index on `status`, MongoDB only reads the matching entries. The cost is a little extra storage and slightly slower writes, so we only index fields we search often.

| Collection | Index | Reason |
|---|---|---|
| doctors | email, specialization | Specialization filter and email lookup |
| patients | email | Email lookup |
| appointments | appointmentDate, status | Date and status filters |
| appointments | **compound** doctorId + appointmentDate + appointmentTime | Fast duplicate-slot check |

Indexes are declared with `@Indexed` and `@CompoundIndex` in the model classes. `spring.data.mongodb.auto-index-creation=true` makes Spring create them at startup.

**How duplicates are prevented:** before saving, `AppointmentService` searches the compound index for a non-cancelled appointment with the same doctor, date and time. If one exists, the API returns HTTP 409 (Conflict). The index is deliberately *not* set to unique, because a cancelled slot must be bookable again. (A partial unique index could be added later; see Future Scope.)

## Replication
See [REPLICATION.md](REPLICATION.md). MongoDB Atlas clusters are replica sets by default.

## Backup and Recovery
See [BACKUP_AND_RECOVERY.md](BACKUP_AND_RECOVERY.md) (`mongodump` and `mongorestore`).

## Installation
Install:
1. **Java JDK 17 or newer**. Check with `java -version`
2. **Maven**. Check with `mvn -version`
3. **VS Code** with the "Extension Pack for Java" (optional but helpful)
4. A free **MongoDB Atlas** account
5. **Postman** (for API testing)

### MongoDB Atlas setup
1. Create a free cluster in Atlas.
2. *Database Access* → add a database user (username + password).
3. *Network Access* → add your IP address (for a college demo, "Allow access from anywhere" also works, but is not secure for real projects).
4. *Database* → *Connect* → *Drivers* → copy the connection string.

## How to add the MongoDB connection
1. Copy `.env.example` to a new file named `.env` (same folder as `pom.xml`).
2. Put your string in it, replacing the placeholder:
   ```
   MONGODB_URI=mongodb+srv://<username>:<password>@<cluster-host>/?retryWrites=true&w=majority
   ```
3. Replace `<username>` and `<password>`. If the password has special characters, URL-encode them.
4. Never commit `.env` (it is already in `.gitignore`).

The app reads `.env` because `application.properties` contains `spring.config.import=optional:file:.env[.properties]`. You must run the app from the folder containing `.env`.

## How to run the project
```
cd appointal
mvn spring-boot:run
```
Open http://localhost:8080

Or build a JAR and run it:
```
mvn clean package
java -jar target\appointal-1.0.0.jar
```

## Testing
### 1. Automated test
```
mvn test
```
Runs `AppointmentServiceTest` (duplicate booking is blocked; a free slot is saved). No database needed.

### 2. Postman
Base URL: `http://localhost:8080`. For POST/PUT use Body → raw → JSON.

**Add Doctor** – `POST /api/doctors`
```json
{
  "name": "Dr. Ravi Kumar",
  "specialization": "Cardiology",
  "phone": "9876543210",
  "email": "ravi@hospital.com",
  "availableDays": "Mon, Wed, Fri",
  "consultationFee": 500
}
```
Copy the `id` from the response.

**Get Doctors** – `GET /api/doctors` (also try `?name=ravi` and `?specialization=Cardiology`)

**Update Doctor** – `PUT /api/doctors/<DOCTOR_ID>`
```json
{
  "name": "Dr. Ravi Kumar",
  "specialization": "Cardiology",
  "phone": "9876543210",
  "email": "ravi@hospital.com",
  "availableDays": "Mon, Tue, Fri",
  "consultationFee": 700
}
```

**Delete Doctor** – `DELETE /api/doctors/<DOCTOR_ID>` (expect 204 No Content)

**Add Patient** – `POST /api/patients`
```json
{
  "name": "Asha Verma",
  "age": 29,
  "gender": "Female",
  "phone": "9123456780",
  "email": "asha@example.com"
}
```

**Get Patients** – `GET /api/patients` (also `?name=asha`)

**Book Appointment** – `POST /api/appointments` (use real ids from the doctor and patient responses; names are filled in automatically)
```json
{
  "patientId": "<PATIENT_ID>",
  "doctorId": "<DOCTOR_ID>",
  "appointmentDate": "2026-11-10",
  "appointmentTime": "10:30",
  "reason": "Chest pain checkup",
  "status": "Scheduled"
}
```
Send the same request again. You should get **409 Conflict** (duplicate prevention).

**Get Appointments** – `GET /api/appointments` (also `?status=Scheduled`, `?date=2026-11-10`, `?search=ravi`)

**Update Appointment** – `PUT /api/appointments/<APPOINTMENT_ID>`
```json
{
  "patientId": "<PATIENT_ID>",
  "doctorId": "<DOCTOR_ID>",
  "appointmentDate": "2026-11-10",
  "appointmentTime": "11:00",
  "reason": "Chest pain checkup",
  "status": "Completed"
}
```

**Delete Appointment** – `DELETE /api/appointments/<APPOINTMENT_ID>` (expect 204)

### 3. Testing from the HTML interface
1. Open http://localhost:8080 and check the dashboard shows 0/0/0.
2. **Doctors** page: add a doctor, edit it, search by name, filter by specialization, delete one.
3. **Patients** page: add, edit, search, delete.
4. **Appointments** page: book an appointment, then try to book the same doctor/date/time again (a red error should appear). Edit it, cancel it, filter by status/date, search by doctor or patient name, delete it.
5. Return to Home and confirm the counts changed.
6. In Atlas → *Browse Collections* → *Indexes* tab, confirm the indexes exist.

## Deployment
**Build:**
```
mvn clean package
```
This creates `target/appointal-1.0.0.jar`.

**Atlas for deployment:**
- Create a database user with a strong password.
- *Network Access*: add the public IP of the server that will run the app. Use `0.0.0.0/0` only for short demos.
- Copy the connection string.

**Run on any server or VM with Java 17+ installed:**
1. Copy the JAR to the server.
2. Set the environment variable, then run:
   - Windows PowerShell:
     ```
     $env:MONGODB_URI="your-real-connection-string"
     java -jar appointal-1.0.0.jar
     ```
   - Linux:
     ```
     export MONGODB_URI="your-real-connection-string"
     java -jar appointal-1.0.0.jar
     ```
3. Open `http://<server-ip>:8080` (allow port 8080 in the server firewall).

**Hosting platforms:** many cloud platforms can build a Maven project from a GitHub repository. In their dashboard, add `MONGODB_URI` as an environment variable (never upload `.env`) and set the start command to `java -jar target/appointal-1.0.0.jar`. The app uses the `PORT` variable automatically if the platform provides one. Platform features change often, so check the platform's current Java/Spring Boot documentation.

## Future Scope
- Login for admin/doctor/patient (Spring Security)
- Email or SMS appointment reminders
- Doctor availability by time slots
- Unique partial index so duplicate slots are blocked by the database itself
- Pagination for large tables
- Reports (appointments per doctor, per month)
