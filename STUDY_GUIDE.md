# Onboard API — Study Guide

Goal: be able to explain every file in this project and change it without help. Read this with the code open next to it. Run the app with the console visible, because seeing the SQL print as you make requests is the fastest way to understand JPA.

Contents:
1. The big picture: one request, start to finish
2. The SQL in schema.sql, line by line
3. The queries JPA runs for you
4. Annotation cheat sheet
5. Validation and errors
6. What the tests check (and what they don't)
7. Practice tasks (do these without AI)
8. Interview questions with model answers
9. Resume bullets

---

## 1. The big picture

If you've used Next.js API routes with Supabase, you already know this shape. A route handler receives the request, does some logic, calls the database, returns JSON. Spring splits that one handler into three layers, each with one job:

```
HTTP request
    │
    ▼
Controller   (controller/)   "Which URL? What's in the body? Is it valid?"     ≈ Next.js route handler
    │  calls
    ▼
Service      (service/)      "What should happen?" Business rules, 404s,         ≈ a lib/ helper function
    │  calls                   transactions, progress math
    ▼
Repository   (repository/)   "Turn this into SQL and run it."                    ≈ supabase.from('tasks')...
    │
    ▼
PostgreSQL   (schema.sql)    The actual tables
```

**Who creates these objects?** You never write `new TaskService(...)`. At startup Spring scans the `com.onboard` package, finds every class marked `@RestController`, `@Service`, `@Repository` or `@Configuration`, creates one of each (a *bean*), and passes them into each other's constructors. This is **dependency injection**. `TaskController`'s constructor asks for a `TaskService`, so Spring hands it the one it made. `TaskService` asks for a `TaskRepository`, an `EmployeeService` and a `Clock`, and gets those.

Why bother? Because a test can call the same constructor with fakes: `new TaskService(fakeRepo, fakeEmployeeService, fixedClock)`. That's exactly what `TaskServiceTest` does.

### Trace one request: `POST /api/employees/1/tasks` with `{"title":"Set up laptop","dueDate":"2026-10-15"}`

1. Tomcat (the web server inside Spring Boot) receives the request.
2. Spring matches the URL and method to `TaskController.addTask` via `@PostMapping("/employees/{employeeId}/tasks")` under the class's `@RequestMapping("/api")`.
3. `@PathVariable` pulls `1` out of the URL as a `Long`. If the URL said `abc`, conversion fails → `MethodArgumentTypeMismatchException` → **400**.
4. `@RequestBody` uses Jackson to turn the JSON into a `CreateTaskRequest` record. `"2026-10-15"` becomes a `LocalDate`. A bad date like `"10/15/2026"` fails here → **400**.
5. `@Valid` checks the record's annotations (`@NotBlank title`, `@NotNull dueDate`). Any failure → `MethodArgumentNotValidException` → **400** with `fieldErrors`. The controller method body never runs.
6. The controller calls `taskService.addTask(1, request)`.
7. `@Transactional` opens a database transaction.
8. The service calls `employeeService.getEmployeeOrThrow(1)` → `employeeRepository.findById(1)` → `SELECT ... FROM employees WHERE id = 1`. No row → `ResourceNotFoundException` → **404**.
9. It builds a `Task` entity and calls `taskRepository.save(task)` → `INSERT INTO tasks ...`. Postgres generates the id.
10. The method returns, the transaction commits.
11. The service converts the entity to a `TaskResponse` DTO; the controller wraps it in `ResponseEntity.created(...)` → **201** with a `Location: /api/tasks/10` header. Jackson writes the record as JSON.

If you can say those steps out loud without looking, you understand the project.

---

## 2. schema.sql, line by line

```sql
CREATE TABLE IF NOT EXISTS employees (
    id          BIGSERIAL    PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    role        VARCHAR(100) NOT NULL,
    start_date  DATE         NOT NULL
);
```

| Piece | Meaning |
|---|---|
| `CREATE TABLE IF NOT EXISTS` | Make the table, unless it's already there. This is why the file can run on every startup without wiping data. It also means **changing this file does not change an existing table** (important for Practice Task 1). |
| `BIGSERIAL` | A 64-bit integer that the database fills in automatically: 1, 2, 3… Maps to Java `Long`. |
| `PRIMARY KEY` | Unique and never null; the row's identity. Postgres automatically indexes it. |
| `VARCHAR(100)` | Text up to 100 characters. Longer text is rejected by the database. Our `@Size(max = 100)` catches it earlier with a friendly 400. |
| `NOT NULL` | The column must have a value. |
| `DATE` | A calendar date with no time or timezone. Maps to Java `LocalDate`. |

```sql
CREATE TABLE IF NOT EXISTS tasks (
    id           BIGSERIAL    PRIMARY KEY,
    title        VARCHAR(200) NOT NULL,
    due_date     DATE         NOT NULL,
    completed    BOOLEAN      NOT NULL DEFAULT FALSE,
    employee_id  BIGINT       NOT NULL REFERENCES employees(id) ON DELETE CASCADE
);
CREATE INDEX IF NOT EXISTS idx_tasks_employee_id ON tasks(employee_id);
```

| Piece | Meaning |
|---|---|
| `DEFAULT FALSE` | If an INSERT doesn't mention `completed`, it's false. |
| `BIGINT` | Same size as BIGSERIAL but not auto-generated. A foreign key column just stores another table's id. |
| `REFERENCES employees(id)` | **Foreign key.** Postgres refuses any task whose `employee_id` isn't a real employee id. Even if our Java code had a bug, the database protects the data. |
| `ON DELETE CASCADE` | Deleting an employee automatically deletes their tasks, instead of erroring. |
| `CREATE INDEX` | A lookup structure, like a book index, so `WHERE employee_id = ?` doesn't scan every row. Postgres does **not** auto-index foreign keys, so we add one. Almost every endpoint filters by `employee_id`. |

The relationship is **one-to-many**: one employee, many tasks. The "many" side holds the foreign key.

### Try the SQL yourself

Connect with `psql -d onboard` (Mac) or `psql -U postgres -d onboard` (Windows) after creating some data through the API:

```sql
\dt                                   -- list tables
\d tasks                              -- show columns, constraints, indexes
SELECT * FROM employees;
SELECT * FROM tasks WHERE employee_id = 1 ORDER BY due_date;

-- Progress, by hand
SELECT count(*) AS total,
       count(*) FILTER (WHERE completed) AS done
FROM tasks WHERE employee_id = 1;

-- Overdue, by hand
SELECT id, title, due_date FROM tasks
WHERE employee_id = 1 AND completed = false AND due_date < CURRENT_DATE
ORDER BY due_date;

-- Watch the foreign key reject bad data
INSERT INTO tasks (title, due_date, employee_id) VALUES ('Orphan', '2026-10-10', 999);
-- ERROR: insert or update on table "tasks" violates foreign key constraint "tasks_employee_id_fkey"

-- A JOIN: each task with its employee's name
SELECT t.title, t.due_date, e.name
FROM tasks t
JOIN employees e ON e.id = t.employee_id;
```

---

## 3. The queries JPA runs for you

**JPA** is the Java standard for mapping objects to tables. **Hibernate** is the library that implements it. **Spring Data JPA** sits on top and writes repository code for you.

`@Entity` classes (`Employee`, `Task`) describe how a Java object maps to a row. Repositories are just interfaces; Spring generates the implementation at startup.

### Methods you get from `JpaRepository`

| Java call | SQL (simplified) |
|---|---|
| `employeeRepository.save(newEmployee)` | `INSERT INTO employees (name, role, start_date) VALUES (?, ?, ?)` and reads back the generated id |
| `employeeRepository.findAll()` | `SELECT id, name, role, start_date FROM employees` |
| `employeeRepository.findById(1L)` | `SELECT ... FROM employees WHERE id = ?` → returns `Optional<Employee>` (empty if no row) |

The `?` marks are **parameters**. Values are sent separately from the SQL text, which is what prevents SQL injection.

### Derived queries in `TaskRepository`

Spring Data reads the method name and builds the query. `EmployeeId` means "the `employee` field, then its `id`", which is the `employee_id` column.

| Method name | SQL |
|---|---|
| `findByEmployeeIdOrderByDueDateAsc(id)` | `SELECT ... FROM tasks WHERE employee_id = ? ORDER BY due_date ASC` |
| `countByEmployeeId(id)` | `SELECT count(*) FROM tasks WHERE employee_id = ?` |
| `countByEmployeeIdAndCompletedTrue(id)` | `SELECT count(*) FROM tasks WHERE employee_id = ? AND completed = true` |
| `findByEmployeeIdAndCompletedFalseAndDueDateBeforeOrderByDueDateAsc(id, today)` | `SELECT ... FROM tasks WHERE employee_id = ? AND completed = false AND due_date < ? ORDER BY due_date ASC` |

Keywords to recognise: `findBy`, `countBy`, `And`, `Or`, `True`, `False`, `Before`, `After`, `Between`, `Containing`, `OrderBy…Asc/Desc`. If a name gets too long, the alternative is writing the query yourself with `@Query("SELECT t FROM Task t WHERE ...")`.

### What you'll actually see in the console

Hibernate gives tables short aliases, so the printed SQL looks like this (exact text varies by version):

```sql
select t1_0.id, t1_0.completed, t1_0.due_date, t1_0.employee_id, t1_0.title
from tasks t1_0
where t1_0.employee_id=? and t1_0.completed=false and t1_0.due_date<?
order by t1_0.due_date
```

### The UPDATE you never wrote

Look at `TaskService.completeTask`: it loads the task, calls `task.markComplete()`, and returns. There is no `save()`. Yet the console shows:

```sql
select ... from tasks t1_0 where t1_0.id=?
update tasks set completed=?, due_date=?, employee_id=?, title=? where id=?
```

This is **dirty checking**. Inside a `@Transactional` method, Hibernate remembers every entity it loaded. When the transaction commits, it compares each one to its original state and writes an UPDATE for anything that changed. (It updates every column by default, not just the changed one.)

### LAZY loading

`Task.employee` is `@ManyToOne(fetch = LAZY)`. Loading a task does **not** load its employee. Hibernate puts a placeholder (a *proxy*) there and only runs `SELECT ... FROM employees` if you call something like `task.getEmployee().getName()`. Calling `getId()` is free because the id is already in the `employee_id` column. That's why `TaskResponse.from(task)` doesn't trigger extra queries.

The classic bug this avoids is the **N+1 problem**: load 50 tasks (1 query), then touch each task's employee name (50 more queries).

---

## 4. Annotation cheat sheet

| Annotation | Where | What it does |
|---|---|---|
| `@SpringBootApplication` | `OnboardApplication` | Turns on component scanning + auto-configuration. The app's starting point. |
| `@Configuration` | `ClockConfig` | A class that defines beans with `@Bean` methods. |
| `@Bean` | `ClockConfig.clock()` | "Call this once; give the result to anyone who needs that type." |
| `@RestController` | controllers | Handles HTTP; return values become JSON. |
| `@RequestMapping("/api/...")` | controller class | URL prefix for every method in the class. |
| `@GetMapping` / `@PostMapping` / `@PatchMapping` | controller methods | Which HTTP method + path runs this method. |
| `@PathVariable` | method param | Copies `{id}` from the URL into the parameter. |
| `@RequestBody` | method param | Converts the JSON body into a Java object. |
| `@Valid` | method param | Runs the validation annotations on that object before the method runs. |
| `@Service` | services | Marks a business-logic bean. |
| `@Transactional` | service methods | One database transaction for the whole method; rolls back on exceptions. `readOnly = true` for reads. |
| `@Repository` | repositories | Marks a database-access bean. |
| `@Entity` | `Employee`, `Task` | This class maps to a table. |
| `@Table(name = "...")` | entities | Which table. |
| `@Id` | entity field | Primary key. |
| `@GeneratedValue(strategy = IDENTITY)` | entity id | The database generates the id (BIGSERIAL). |
| `@Column(name, nullable, length)` | entity fields | Which column, and its rules. |
| `@ManyToOne(fetch = LAZY)` | `Task.employee` | Many tasks → one employee; load the employee only when needed. |
| `@JoinColumn(name = "employee_id")` | `Task.employee` | The foreign-key column for that relationship. |
| `@NotBlank` / `@NotNull` / `@Size` | DTO fields | Validation rules. |
| `@RestControllerAdvice` | `GlobalExceptionHandler` | Catches exceptions from all controllers. |
| `@ExceptionHandler(X.class)` | handler methods | "When X is thrown, return this response instead." |

Test annotations are in section 6.

---

## 5. Validation and errors

Two different kinds of "bad request" are handled in two different places:

- **The request is malformed** (missing field, blank name, unparseable date, non-numeric id). Caught *before* the service runs, by `@Valid` and Jackson. → **400**.
- **The request is well-formed but refers to something that doesn't exist** (employee 999). Only the service can know that, because it has to ask the database. It throws `ResourceNotFoundException`. → **404**.

`GlobalExceptionHandler` maps each exception type to a status code and the shared `ErrorResponse` JSON. Without it, a `ResourceNotFoundException` would surface as a **500 Internal Server Error**, which tells the client "our bug" when it's really "your bad id".

Layered defense: `@Size(max = 100)` gives a friendly 400 first; `VARCHAR(100)` in the database is the backstop if something gets past it. Same idea with `@NotNull` and `NOT NULL`, and with the service's 404 check and the foreign key.

---

## 6. What the tests check

Run with `mvn test`. 24 tests in 5 classes. None need a database, so they run in seconds.

### Test vocabulary

| Thing | Meaning |
|---|---|
| `@Test` | JUnit 5: this method is a test. It passes if nothing throws. |
| `@BeforeEach` | Runs before every test, so each starts fresh. |
| `assertThat(x).isEqualTo(y)` | AssertJ assertion. Fails the test if false. |
| `assertThatThrownBy(() -> ...).isInstanceOf(...)` | Passes only if the code throws that exception. |
| `@ExtendWith(MockitoExtension.class)` | Lets JUnit create Mockito mocks. |
| `@Mock` | A fake object. Returns null/empty/0 unless scripted. |
| `when(mock.method(args)).thenReturn(value)` | Script the fake. |
| `verify(mock, never()).save(any())` | Check how the fake was (or wasn't) used. |
| `@InjectMocks` | Build the real class, passing the mocks to its constructor. |
| `@WebMvcTest(X.class)` | Start only Spring's web layer with controller X and the exception handler. No database. |
| `MockMvc` | Sends fake HTTP requests to the controller and checks the response. |
| `@MockitoBean` | Replace a real bean (the service) with a mock inside Spring. |
| `jsonPath("$.fieldErrors.name")` | Read a value out of the JSON response. |

Most tests follow **Arrange / Act / Assert**: set up the fakes, call the thing, check the result.

### The tests

**`TaskTest`**: plain Java, no mocks.
- Incomplete task due yesterday → overdue.
- Due today → not overdue (the boundary case).
- Completed task with an old due date → not overdue.

**`EmployeeServiceTest`**: service + mock repository.
- `create` trims whitespace and returns the id the "database" assigned.
- `findById` on a missing id throws `ResourceNotFoundException("Employee 99 not found")`.

**`TaskServiceTest`**: service + mock repository + **fixed clock** (today = 2026-10-07).
- `addTask` saves and returns a task linked to the employee, `completed = false`.
- `addTask` for an unknown employee throws **and never calls `save`**.
- `completeTask` flips `completed` to true.
- `completeTask` on an unknown id throws.
- Progress: 3 of 4 done → 75%, with the one overdue task listed.
- Progress with 0 tasks → 0% (no divide-by-zero).
- Rounding: 1/3 → 33, 2/3 → 67, 3/3 → 100.

**`EmployeeControllerTest`**: real HTTP handling, mock services.
- Valid POST → 201, `Location: /api/employees/1`, date serialized as `"2026-10-14"`.
- Blank name + missing date → 400 with both field errors, and the service is never called.
- Date `"10/14/2026"` → 400 with the date-format message.
- GET list → 200 with 2 items.
- Unknown id → 404 with the error JSON.
- `/api/employees/abc` → 400.
- Progress endpoint returns the service's numbers as JSON.

**`TaskControllerTest`**
- Add task → 201 with `Location`.
- Missing `dueDate` → 400.
- Unknown employee → 404.
- PATCH complete → 200, `completed: true`.
- PATCH unknown task → 404.

### What the tests don't cover

Every test mocks the repository, so **no automated test runs real SQL**. If you misspelled a derived query name, Spring would fail at startup, but a wrong `WHERE` clause would only show up when you run the app. The next step is an integration test that uses a real Postgres, usually with `@DataJpaTest` plus Testcontainers (a library that starts a throwaway Postgres in Docker for the test run). Being able to name this gap is a strong interview answer.

---

## 7. Practice tasks: do these without AI

Do them in order. For each, the finish line is: `mvn test` passes, and you've called the endpoint with curl and seen the right response. Commit after each one.

### Task 1: Add an `email` column to employees

Every employee should have an email. It's required and must look like an email. Return it in responses.

Hints:
- Files to touch: `schema.sql`, `Employee.java`, `CreateEmployeeRequest.java`, `EmployeeResponse.java`, `EmployeeService.java`.
- Your existing table won't change just because you edited `CREATE TABLE IF NOT EXISTS`. Think about what that means. You have two options: drop the tables (fine, it's local data), or learn `ALTER TABLE ... ADD COLUMN` and think about what happens to rows that already exist when a new column is `NOT NULL`.
- Hibernate is set to `validate`. If you change the entity but not the database, startup fails. Read that error message; it tells you exactly what's missing.
- Jakarta Validation has an annotation for email format. Look at the imports in `CreateEmployeeRequest` for where it would live.
- After adding a field to a record, the compiler will point at every test that builds that record with the old number of arguments. Fix them one by one.
- Stretch: should two employees be allowed the same email? Which SQL keyword prevents it, and what HTTP status should the API return when it happens?

### Task 2: Add `DELETE /api/tasks/{taskId}`

Delete one task. Return **204 No Content** on success and **404** if the task doesn't exist.

Hints:
- Look at how `TaskController` maps PATCH and find the DELETE equivalent.
- A 204 has no body. `ResponseEntity` has a builder method for it; the controller method's return type will need to change.
- `JpaRepository` already has the methods you need. One checks whether an id exists; one deletes by id.
- Where should the "does it exist?" check live: controller or service? Look at how `completeTask` handles it.
- Watch the console: what SQL does the delete run?

### Task 3: Write one new test

Write a test in `TaskServiceTest` proving that `listTasks` for an unknown employee throws `ResourceNotFoundException` **and never queries the task table**.

Hints:
- Copy the structure of `addTaskForUnknownEmployeeThrowsAndSavesNothing`. Arrange / Act / Assert.
- You need the mock `EmployeeService` to throw. You've seen `thenThrow` already.
- To prove the task table was never touched, verify that a specific `TaskRepository` method was `never()` called. Which method does `listTasks` call?
- Make sure your test can fail: temporarily delete the `getEmployeeOrThrow` line from `listTasks`, rerun, watch it go red, then put the line back.
- Stretch: if you did Task 2, write a `TaskControllerTest` for the 404 case of your DELETE endpoint.

---

## 8. Interview questions

Practice saying these out loud. The model answers are short on purpose; expand from your own understanding.

**1. Walk me through what happens when someone adds a task.**
The request hits `TaskController`, which reads the employee id from the path and validates the JSON body with `@Valid`. If the body is invalid we return 400 before any logic runs. The controller calls `TaskService.addTask`, which in a transaction looks up the employee (404 if missing), creates a `Task` entity, and saves it through `TaskRepository`, which runs the INSERT. We return 201 with the new task and a `Location` header.

**2. Why split controller, service and repository?**
Each layer has one reason to change. The controller only knows HTTP, the service holds business rules, the repository only talks to the database. It also makes testing easy: I test the service with a mock repository and the controller with a mock service.

**3. Why use DTOs instead of returning the entity?**
The entity mirrors the table; the DTO is the API contract. Separating them stops clients from setting fields like `id`, lets me change the table without breaking clients, and avoids serializing lazy relationships by accident. I used Java records for DTOs because they're immutable and concise.

**4. How do you decide between 400 and 404?**
400 means the request itself is invalid: missing fields, bad date format, non-numeric id. That's caught by validation before the service runs. 404 means the request is valid but the employee or task doesn't exist, which only the service knows after querying. A `@RestControllerAdvice` maps each exception to its status and returns one consistent error JSON.

**5. How does Spring Data know what SQL to run for `countByEmployeeIdAndCompletedTrue`?**
It parses the method name into a query at startup: count tasks where `employee.id` equals the parameter and `completed` is true. Hibernate turns that into `SELECT count(*) FROM tasks WHERE employee_id = ? AND completed = true`, with the value passed as a bound parameter. If a name got too complex I'd switch to `@Query`.

**6. What does the foreign key do? Why the index?**
`employee_id REFERENCES employees(id)` means Postgres rejects a task for an employee that doesn't exist, so the data stays consistent even if the app has a bug. `ON DELETE CASCADE` removes an employee's tasks with them. Postgres doesn't index foreign keys automatically, and nearly every query filters by `employee_id`, so I added an index.

**7. `completeTask` never calls `save()`. How does the update happen?**
The method is `@Transactional`. Hibernate tracks every entity loaded in the transaction, and at commit it compares them to their original state and issues an UPDATE for changes. That's dirty checking. The endpoint is also idempotent: completing a task twice leaves it complete without an error, which fits PATCH semantics.

**8. How do you compute progress, and how did you make "overdue" testable?**
Counts happen in the database with `count(*)` queries instead of loading every task into memory. Percent is completed × 100 / total, rounded, and 0 when there are no tasks to avoid dividing by zero. Overdue means not completed and due before today. "Today" comes from an injected `Clock`, so tests pass a fixed date and the result is deterministic.

**9. What do your tests cover, and what don't they?**
24 JUnit 5 tests at three levels: plain unit tests for the overdue rule, service tests with Mockito mocks for business logic, and `@WebMvcTest` tests with MockMvc for routing, JSON, validation and 400/404 responses. The gap is that no automated test runs real SQL, because repositories are mocked. I'd add `@DataJpaTest` with Testcontainers to run the queries against a real Postgres.

**10. What would you add next for production?**
Pagination on the list endpoints, authentication, schema migrations with Flyway instead of `schema.sql` so changes are versioned, the integration tests above, OpenAPI docs, and a frontend. For larger data, I'd watch for N+1 queries; the LAZY relationship already helps.

---

## 9. Resume bullets (Jake's Resume style)

Only use these once you've run `mvn test` yourself, completed the practice tasks, and can answer the questions above. Update the numbers if you add endpoints or tests (after Tasks 2 and 3, it's 8 endpoints and 25+ tests).

```latex
\resumeProjectHeading
  {\textbf{Onboard API} $|$ \emph{Java, Spring Boot, PostgreSQL, JUnit 5}}{Oct 2026}
  \resumeItemListStart
    \resumeItem{Built a REST API in Java 21 and Spring Boot 3 with 7 endpoints for tracking new-hire onboarding tasks, backed by a 2-table PostgreSQL schema with foreign-key constraints and Spring Data JPA}
    \resumeItem{Wrote 24 JUnit 5 tests with Mockito and MockMvc covering input validation, 400/404 error handling and progress/overdue calculations, using an injected clock for deterministic date logic}
  \resumeItemListEnd
```

Numbers in these bullets, checked against the code: 7 endpoints (README table), 2 tables (`schema.sql`), 24 tests (3 + 2 + 7 + 7 + 5).
