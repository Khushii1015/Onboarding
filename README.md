# Onboard API

A REST API for tracking new-hire onboarding checklists. HR creates an employee, adds onboarding tasks (set up laptop, sign policies, meet your buddy), marks them complete, and checks each person's progress: percent complete and which tasks are overdue.

**Stack:** Java 21 · Spring Boot 3.5 (Web, Data JPA, Validation) · PostgreSQL · Maven · JUnit 5 + Mockito + MockMvc

## Endpoints

| Method | Path | What it does | Success | Errors |
|---|---|---|---|---|
| POST | `/api/employees` | Create an employee | 201 + `Location` header | 400 |
| GET | `/api/employees` | List all employees | 200 | — |
| GET | `/api/employees/{id}` | Get one employee | 200 | 404, 400 (non-numeric id) |
| POST | `/api/employees/{id}/tasks` | Add a task to an employee | 201 + `Location` header | 400, 404 |
| GET | `/api/employees/{id}/tasks` | List an employee's tasks (by due date) | 200 | 404 |
| PATCH | `/api/tasks/{taskId}/complete` | Mark a task complete (safe to repeat) | 200 | 404 |
| GET | `/api/employees/{id}/progress` | % complete + overdue tasks | 200 | 404 |

A task is **overdue** when it is not completed and its due date is before today (due today is not overdue). With zero tasks, progress is 0%.

Every error has the same JSON shape:

```json
{
  "timestamp": "2026-10-07T16:30:00Z",
  "status": 400,
  "error": "Bad Request",
  "message": "Validation failed",
  "fieldErrors": { "name": "name is required" }
}
```

## Project structure

```
onboard-api/
├── pom.xml                                   Maven: dependencies + build
├── README.md
├── STUDY_GUIDE.md                            How it works, practice tasks, interview prep
├── postman/Onboard-API.postman_collection.json
└── src/
    ├── main/
    │   ├── java/com/onboard/
    │   │   ├── OnboardApplication.java       Entry point (starts the server)
    │   │   ├── config/ClockConfig.java       Provides "today" (swappable in tests)
    │   │   ├── controller/                   HTTP layer: URLs -> methods
    │   │   │   ├── EmployeeController.java
    │   │   │   └── TaskController.java
    │   │   ├── service/                      Business logic
    │   │   │   ├── EmployeeService.java
    │   │   │   └── TaskService.java
    │   │   ├── repository/                   Database access (Spring Data JPA)
    │   │   │   ├── EmployeeRepository.java
    │   │   │   └── TaskRepository.java
    │   │   ├── model/                        Entities = table rows
    │   │   │   ├── Employee.java
    │   │   │   └── Task.java
    │   │   ├── dto/                          JSON request/response shapes
    │   │   │   ├── CreateEmployeeRequest.java
    │   │   │   ├── CreateTaskRequest.java
    │   │   │   ├── EmployeeResponse.java
    │   │   │   ├── TaskResponse.java
    │   │   │   └── ProgressResponse.java
    │   │   └── exception/                    Error -> HTTP status mapping
    │   │       ├── ResourceNotFoundException.java
    │   │       ├── ErrorResponse.java
    │   │       └── GlobalExceptionHandler.java
    │   └── resources/
    │       ├── application.properties        DB connection + settings
    │       └── schema.sql                    The tables, in plain SQL
    └── test/java/com/onboard/
        ├── model/TaskTest.java               3 tests: overdue rule
        ├── service/EmployeeServiceTest.java  2 tests
        ├── service/TaskServiceTest.java      7 tests: tasks + progress math
        ├── controller/EmployeeControllerTest.java  7 tests: HTTP, validation, 400/404
        └── controller/TaskControllerTest.java      5 tests
```

## Setup

You need three things: **Java 21**, **Maven**, and **PostgreSQL**.

### Mac

```bash
# 1. Install tools (Homebrew: https://brew.sh)
brew install openjdk@21 maven postgresql@16
# follow the "sudo ln -sfn ..." line brew prints for openjdk@21, then check:
java -version      # should say 21
mvn -version

# 2. Start Postgres and create the database
brew services start postgresql@16
createdb onboard

# 3. Homebrew's Postgres user is your Mac username (no password), so tell the app:
export DB_USERNAME=$(whoami)

# 4. Run the tests (no database needed for these)
cd onboard-api
mvn test

# 5. Start the API on http://localhost:8080
mvn spring-boot:run
```

### Windows (PowerShell)

```powershell
# 1. Install tools
winget install EclipseAdoptium.Temurin.21.JDK
# Maven: download the binary zip from https://maven.apache.org/download.cgi, unzip it
#   (e.g. C:\tools\apache-maven), and add its \bin folder to your PATH.
# PostgreSQL: run the installer from https://www.postgresql.org/download/windows/
#   and remember the password you set for the "postgres" user.
# Open a NEW terminal, then check:
java -version
mvn -version

# 2. Create the database (psql is in C:\Program Files\PostgreSQL\16\bin)
psql -U postgres -c "CREATE DATABASE onboard;"

# 3. Tell the app your postgres password (for this terminal session)
$env:DB_PASSWORD = "the-password-you-chose"

# 4. Tests, then run
cd onboard-api
mvn test
mvn spring-boot:run
```

### Using a hosted database instead (Neon or Supabase)

Copy the connection details from the provider's dashboard and set three environment variables before `mvn spring-boot:run`:

```bash
export DB_URL="jdbc:postgresql://<host>:5432/<database>?sslmode=require"
export DB_USERNAME="<user>"
export DB_PASSWORD="<password>"
```

The URL must start with `jdbc:postgresql://`. Supabase and Neon show `postgresql://user:pass@host/db`; move the user and password into the two other variables and add the `jdbc:` prefix.

### What happens at startup

1. Spring runs `schema.sql`, which creates the tables if they don't exist.
2. Hibernate checks that the Java entities match those tables (`ddl-auto=validate`) and stops with an error if they don't.
3. The server listens on port 8080. Every SQL query is printed to the console (`show-sql=true`) so you can see what JPA does.

To reset all data: `psql -d onboard -c "DROP TABLE tasks, employees;"` then restart the app.

## Try it (curl)

On Windows, use Git Bash for these commands, or import the Postman collection in `postman/`. Example responses assume today is 2026-10-07; dates and ids will differ for you.

**1. Create an employee** → 201

```bash
curl -i -X POST http://localhost:8080/api/employees \
  -H "Content-Type: application/json" \
  -d '{"name": "Ana Lopez", "role": "Software Engineer", "startDate": "2026-10-01"}'
```
```
HTTP/1.1 201
Location: /api/employees/1
{"id":1,"name":"Ana Lopez","role":"Software Engineer","startDate":"2026-10-01"}
```

**2. List employees** → 200

```bash
curl http://localhost:8080/api/employees
```

**3. Add three tasks** → 201 each (the first one is already past due)

```bash
curl -X POST http://localhost:8080/api/employees/1/tasks -H "Content-Type: application/json" \
  -d '{"title": "Set up laptop", "dueDate": "2026-10-02"}'
curl -X POST http://localhost:8080/api/employees/1/tasks -H "Content-Type: application/json" \
  -d '{"title": "Meet onboarding buddy", "dueDate": "2026-10-03"}'
curl -X POST http://localhost:8080/api/employees/1/tasks -H "Content-Type: application/json" \
  -d '{"title": "Sign policies", "dueDate": "2026-10-20"}'
```

**4. List Ana's tasks** → 200, sorted by due date

```bash
curl http://localhost:8080/api/employees/1/tasks
```

**5. Complete task 2** → 200 with `"completed": true`

```bash
curl -X PATCH http://localhost:8080/api/tasks/2/complete
```

**6. Check progress** → 200

```bash
curl http://localhost:8080/api/employees/1/progress
```
```json
{"employeeId":1,"employeeName":"Ana Lopez","totalTasks":3,"completedTasks":1,"percentComplete":33,
 "overdueCount":1,"overdueTasks":[{"id":1,"title":"Set up laptop","dueDate":"2026-10-02","completed":false,"employeeId":1}]}
```

**7. See the errors work**

```bash
# 400: blank name and missing startDate
curl -i -X POST http://localhost:8080/api/employees -H "Content-Type: application/json" \
  -d '{"name": "", "role": "Engineer"}'

# 400: wrong date format
curl -i -X POST http://localhost:8080/api/employees -H "Content-Type: application/json" \
  -d '{"name": "Ben", "role": "Analyst", "startDate": "10/14/2026"}'

# 400: id isn't a number
curl -i http://localhost:8080/api/employees/abc

# 404: no such employee / task
curl -i http://localhost:8080/api/employees/999
curl -i -X POST http://localhost:8080/api/employees/999/tasks -H "Content-Type: application/json" \
  -d '{"title": "x", "dueDate": "2026-10-10"}'
curl -i -X PATCH http://localhost:8080/api/tasks/999/complete
```

While you run these, watch the app's console: each request prints the SQL it ran.

## Tests

```bash
mvn test
```

24 JUnit 5 tests, none of which need a database:

- **Unit tests** (`TaskTest`): the overdue rule on its own.
- **Service tests** (`EmployeeServiceTest`, `TaskServiceTest`): business logic with a Mockito fake repository and a fixed clock.
- **Web tests** (`EmployeeControllerTest`, `TaskControllerTest`): real HTTP routing, JSON, validation and error handling through MockMvc, with fake services.

The repository queries themselves are exercised by the curl walkthrough against a real database, not by an automated test yet (see STUDY_GUIDE.md, "What the tests don't cover").
