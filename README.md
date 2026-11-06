# College Event Management System

A Java Swing **desktop application** for managing college events such as technical symposiums, workshops, hackathons, coding contests and cultural programs. Administrators manage events, students and registrations. Students create accounts, browse upcoming events and register for them. All data is stored in **MySQL** and accessed through **JDBC**.

Built as an integrated **Java OOP + DBMS** college project.

---

## Table of contents

- [Features](#features)
- [Technology stack](#technology-stack)
- [Architecture](#architecture)
- [Project structure](#project-structure)
- [Database design](#database-design)
- [Getting started](#getting-started)
- [Creating the first administrator](#creating-the-first-administrator)
- [Security](#security)
- [OOP and DBMS concepts demonstrated](#oop-and-dbms-concepts-demonstrated)
- [Screenshots](#screenshots)
- [Project status](#project-status)
- [Troubleshooting](#troubleshooting)
- [License](#license)

---

## Features

### Student
- Create an account (name, email, department, year, phone, password) with validation
- Log in and see a personal dashboard
- Browse upcoming events, search by name or event ID, and view full event details
- Register for an event with a unique registration ID (for example `REG-000012`)
- Seat availability is checked safely, even when many students register at the same time
- View **My Registrations**, and cancel a registration up to the day before the event
- Record the fee paid for a registration, and remove cancelled registrations from the list

### Administrator
- Dashboard with live statistics read from MySQL: total events, registered students, total registrations, upcoming events
- **Manage Events:** add, update and delete events, search by name or ID, filter by date range, see registrations and seats left
- **Manage Students:** search students, view details, and see the events each student registered for
- **Manage Registrations:** search and filter every registration, see the fee paid, and cancel a registration

### General
- Role-based access: students cannot reach administrator operations
- Input validation on every form, and confirmation dialogs before destructive actions
- Data persists in MySQL after the application is closed

---

## Technology stack

| Area | Technology |
|---|---|
| Language | Java 17 or newer |
| GUI | Java Swing |
| Database | MySQL 8 |
| Database access | JDBC with MySQL Connector/J 8.4.0 |
| Build tool | Apache Maven 3.9 |
| Editor used | Visual Studio Code on Windows |

No web technologies are used. This is a standalone desktop program.

---

## Architecture

The code is split into layers. Each layer only talks to the one below it, so the screens never contain SQL.

```
UI layer (Swing)          com.college.event.ui       windows, tables, dialogs
        |
Service layer             com.college.event.service  validation, authorization, transactions
        |
DAO layer (JDBC)          com.college.event.dao      all SQL, PreparedStatement
        |
MySQL                     college_event_db

model (entities) and util (hashing, validation) are used by all layers
```

---

## Project structure

```
CollegeEventManagement/
├── pom.xml
├── README.md
├── .gitignore
├── database/
│   ├── schema.sql              tables, constraints, indexes and views
│   └── sample_data.sql         sample events for testing
└── src/main/java/com/college/event/
    ├── Main.java               application entry point
    ├── model/                  Person, Student, Admin, Event, Registration,
    │                           RegistrationDetail, StudentSummary, DashboardStats,
    │                           Role, EventStatus, RegistrationStatus, Summarizable
    ├── dao/                    DBConnection, StudentDAO, AdminDAO, EventDAO,
    │                           RegistrationDAO, StatisticsDAO
    ├── service/                AuthenticationService, EventService, RegistrationService,
    │                           StudentService, StatisticsService, ServiceException
    ├── ui/                     LoginFrame, RegisterFrame, AdminDashboard, StudentDashboard,
    │                           EventManagementPanel, StudentManagementPanel,
    │                           RegistrationPanel, EventFormDialog, Sidebar, UITheme,
    │                           BackgroundTask
    └── util/                   PasswordUtil, ValidationUtil, EventValidator, CreateAdminTool
```

---

## Database design

Database name: `college_event_db`

| Table | Purpose |
|---|---|
| `administrators` | administrator accounts (password hash only) |
| `students` | student accounts (unique email, password hash only) |
| `events` | event details, fee, capacity and status |
| `registrations` | links a student to an event, with status and fee paid |

Views: `v_event_summary` (events with registered count and available seats) and `v_registration_details` (registrations joined with student and event details).

```
students 1 ────────< registrations >──────── 1 events
         (one student has many)   (one event has many)

administrators   (stand-alone, used for administrator login)
```

Key rules enforced by the database:
- `UNIQUE (student_id, event_id)` stops the same student registering twice for one event
- Foreign keys use `ON DELETE RESTRICT`, so an event or student with registrations cannot be deleted
- `CHECK` constraints reject negative fees, zero capacity and invalid years
- Seats are calculated (capacity minus active registrations), never stored, so counts cannot drift

---

## Getting started

### 1. Prerequisites

| Software | Version |
|---|---|
| JDK | 17 or newer |
| Apache Maven | 3.9 or newer |
| MySQL Server | 8.0.16 or newer (8.4 LTS recommended) |
| MySQL Workbench | optional, for running the SQL scripts |

Check your setup:

```
java -version
javac -version
mvn -version
```

### 2. Clone the repository

```
git clone https://github.com/<your-username>/CollegeEventManagement.git
cd CollegeEventManagement
```

### 3. Create the database

In MySQL Workbench, open and run these two files **in this order**:

1. `database/schema.sql`
2. `database/sample_data.sql`

Or from a terminal (adjust the path to `mysql` if it is not on your PATH):

```
mysql -u root -p < database/schema.sql
mysql -u root -p < database/sample_data.sql
```

### 4. Create a database user for the application

The application should not log in as `root`. Run this in MySQL, using your own password:

```sql
CREATE USER IF NOT EXISTS 'event_app'@'localhost' IDENTIFIED BY 'choose-your-own-password';
GRANT SELECT, INSERT, UPDATE, DELETE ON college_event_db.* TO 'event_app'@'localhost';
```

### 5. Set the database credentials

Credentials are read from environment variables, so no password is stored in the source code.

| Variable | Required | Meaning |
|---|---|---|
| `DB_USER` | yes | MySQL user, for example `event_app` |
| `DB_PASSWORD` | yes | password of that user |
| `DB_URL` | no | only if MySQL is not on `localhost:3306` |

On Windows (PowerShell):

```
setx DB_USER "event_app"
setx DB_PASSWORD "choose-your-own-password"
```

`setx` only affects programs started afterwards, so **close VS Code or the terminal completely and open it again**. Check without printing the password:

```
echo $env:DB_USER
$env:DB_PASSWORD.Length
```

On Linux or macOS:

```
export DB_USER="event_app"
export DB_PASSWORD="choose-your-own-password"
```

### 6. Build and run

```
mvn clean compile
mvn exec:java
```

The login window opens. The first run downloads the MySQL driver, so it needs an internet connection.

---

## Creating the first administrator

Students cannot create administrator accounts from the application. The first administrator is created with a small console tool that hashes the password before saving it.

PowerShell:

```
mvn clean compile
mvn "-Dmain.class=com.college.event.util.CreateAdminTool" exec:java
```

Command Prompt:

```
mvn -Dmain.class=com.college.event.util.CreateAdminTool exec:java
```

Enter a name, an email and a password (at least 8 characters, with a letter and a digit). Then start the application, choose **Administrator** on the login window and log in.

Students create their own accounts with **New student? Create an account** on the login window.

---

## Security

- **Password hashing:** PBKDF2 (HMAC-SHA256) with a random salt and 120,000 iterations. Plain-text passwords are never stored, and hashes are compared in constant time.
- **SQL injection:** every query that uses user input is a `PreparedStatement`.
- **No secrets in code:** database credentials come from environment variables.
- **Role-based authorization:** services check the user's role before every administrator or student operation, in addition to the screens hiding buttons.
- **Same error for unknown email and wrong password**, so an attacker cannot discover which emails exist.
- **Transactions:** registering and cancelling run inside a transaction with a row lock (`SELECT ... FOR UPDATE`), so two students can never take the last seat together.

---

## OOP and DBMS concepts demonstrated

**Java OOP:** encapsulation, constructors and overloading, inheritance (`Student` and `Admin` extend the abstract `Person`), method overriding, abstract classes, the `Object` class (`toString`, `equals`, `hashCode`), interfaces (`Summarizable`), nested interfaces, enums, records, custom and built-in exceptions with `try / catch / finally`, `try-with-resources`, collections, iterators, lambda expressions, functional interfaces, the Stream API (`filter`, `map`, `sorted`, `reduce`, `collect`), and JDBC.

**DBMS:** ER modelling, primary and foreign keys, normalization (3NF), `INSERT / SELECT / UPDATE / DELETE`, joins (`INNER` and `LEFT`), aggregate functions with `GROUP BY`, nested subqueries, views, indexes, `CHECK` and `UNIQUE` constraints, transactions with `COMMIT` and `ROLLBACK`, ACID properties, row-level locking, and isolation levels.

---

## Screenshots

Add your screenshots to `docs/screenshots/` and show them here, for example:

```markdown
![Login](docs/screenshots/login.png)
![Admin dashboard](docs/screenshots/admin-dashboard.png)
![Manage events](docs/screenshots/manage-events.png)
![Student dashboard](docs/screenshots/student-dashboard.png)
```

---

## Project status

Working: login and sign-up, student dashboard, event registration with transactions, administrator dashboard, Manage Events, Manage Students and Manage Registrations.

Planned: an **Event Statistics** window with registrations per event, available seats, completed events and the most popular event.

Ideas for later: payment verification by an administrator, password reset, email confirmations, CSV or PDF export, and event posters.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Database credentials not found` | Set `DB_USER` and `DB_PASSWORD`, then fully restart the terminal or VS Code. |
| `Access denied for user` | Check the password, and run the `CREATE USER` and `GRANT` statements again. |
| `Communications link failure` | The MySQL service is not running. Start it from `services.msc`. |
| `Unknown database 'college_event_db'` | Run `database/schema.sql` first. |
| `ClassNotFoundException` when running | `exec:java` does not compile. Run `mvn clean compile` first. |
| `Lock wait timeout exceeded` | Close other MySQL Workbench tabs that have an open transaction. |
| Workbench error 1175 on `DELETE` or `UPDATE` | Safe Updates mode is on. Add a `WHERE` on a key column, for example `WHERE registration_id > 0`. |

---

## License

Add a `LICENSE` file of your choice (for example MIT) before publishing. This project was created for educational purposes.
