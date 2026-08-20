# my-task-application

A task management web app built on Vaadin Flow 24 and Spring Boot 3, using the Vaadin "walking skeleton" as its foundation. The UI is written entirely in Java — no separate frontend project, no REST layer between client and server.

Started from the official Vaadin starter and customized under `com.aspect.app`. It's a study project for exploring Vaadin Flow, Spring Security integration, and architecture testing with ArchUnit.

> **Heads up:** the build does not currently compile. See [Build blocker](#build-blocker) before you try to run it.

---

## Stack

| Layer | Choice |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.5.3 |
| UI | Vaadin 24.8.2 (Flow, commercial `vaadin` artifact) |
| Security | Spring Security, with dev and Control Center (OIDC) configurations |
| Persistence | Spring Data JPA + Hibernate |
| Database | PostgreSQL in production, H2 in-memory for local dev |
| Testing | JUnit 5, AssertJ, Spring Security Test, Testcontainers, ArchUnit 1.4.1 |
| Formatting | Spotless — Eclipse formatter for Java, Prettier for TypeScript |
| Ops | Spring Boot Actuator, Dockerfile |
| Build | Maven, wrapper included |

---

## Build blocker

`pom.xml` pins the compiler plugin to Java 8:

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-compiler-plugin</artifactId>
    <configuration>
        <source>8</source>
        <target>8</target>
    </configuration>
</plugin>
```

This overrides the `<java.version>21</java.version>` property, and the code doesn't compile under it. `CurrentUser` uses pattern-matching `instanceof` (Java 16+), `AbstractEntity` uses `String.formatted()` (Java 15+), and `var` is used throughout (Java 10+). Spring Boot 3.5 requires Java 17 as a floor regardless.

Fix: delete the plugin block entirely. The Spring Boot parent POM derives compiler settings from `java.version` on its own.

---

## Running it

### Local development (default)

```bash
./mvnw
```

The default goal is `spring-boot:run` and the default profile is `h2-local-development`, so this starts with an H2 in-memory database and needs no Docker. Hibernate creates the schema on startup (`ddl-auto=update`). The browser opens automatically at `http://localhost:8080`.

Because the database is in-memory, **all tasks are lost on restart**.

### With PostgreSQL

If you have Docker running and want dev to match production:

```bash
./mvnw spring-boot:test-run
```

This uses `TestcontainersConfiguration`, which spins up `postgres:17-alpine` and wires it in via `@ServiceConnection`. The `h2-local-development` profile can be deleted from `pom.xml` once you've committed to this path.

### Development login

Two in-memory users, both with password `123`:

| Username | Name | Roles |
|---|---|---|
| `admin` | Alice Administrator | ADMIN, USER |
| `user` | Ursula User | USER |

These come from `SampleUsers` and only exist under `DevSecurityConfig`, which logs a warning on startup and deactivates itself when `ControlCenterSecurityConfig` is present.

### Production build

```bash
./mvnw -Pproduction package
java -jar target/my-task-application-1.0-SNAPSHOT.jar
```

The `production` profile runs Vaadin's `build-frontend` goal and excludes `vaadin-dev` from the artifact.

### Docker

```bash
./mvnw -Pproduction package
docker build -t my-task-application .
docker run -p 8080:8080 my-task-application
```

The container reads `PORT` from the environment (`server.port=${PORT:8080}`), so it drops into Cloud Run, Heroku, and similar platforms without changes.

### Tests

```bash
./mvnw test                       # unit + architecture tests
./mvnw -Pintegration-test verify  # adds *IT tests via Failsafe
```

`TaskServiceIT` needs Docker — it starts a real PostgreSQL container. Surefire only picks up `*Test` classes, so the `IT` suffix keeps it out of the fast loop by design.

---

## Architecture

The package structure is feature-first, not layer-first. Each feature owns its full vertical slice:

```
com.aspect.app
├── Application.java              Entry point, @Theme("default"), Clock bean
├── base/                         Shared infrastructure
│   ├── domain/AbstractEntity     JPA superclass with proxy-safe equals/hashCode
│   └── ui/
│       ├── component/ViewToolbar Reusable view header
│       └── view/
│           ├── MainLayout        @Layout — app shell, side nav, user menu
│           ├── MainView          @Route("") — landing page
│           └── MainErrorHandler  Session error handler → user-facing notification
├── security/                     Authentication and the user model
│   ├── AppRoles                  ADMIN / USER constants
│   ├── AppUserInfo               The app's user model
│   ├── AppUserPrincipal          Bridge to Spring Security principals
│   ├── CurrentUser               Injectable accessor for the logged-in user
│   ├── CommonSecurityConfig      @EnableMethodSecurity
│   ├── controlcenter/            Production OIDC path via Vaadin Control Center
│   └── dev/                      In-memory users + a dev login view
└── taskmanagement/               The feature
    ├── domain/Task               @Entity
    ├── domain/TaskRepository     JpaRepository + JpaSpecificationExecutor
    ├── service/TaskService       @PreAuthorize("isAuthenticated()")
    └── ui/view/TaskListView      @Route("task-list")
```

Every package carries a `package-info.java` with `@NullMarked` (JSpecify), so null-safety is opt-out rather than opt-in.

A few decisions worth calling out:

**`Clock` is injected, not called statically.** `Application` exposes a `Clock` bean, and `TaskService` takes it as a constructor dependency. That's what makes `TaskServiceIT` able to assert on timestamps deterministically — no `Instant.now()` scattered through the service.

**`AbstractEntity` returns a class-based hash code, not an ID-based one.** Hibernate proxies and pre-persist entities would otherwise change their hash code mid-lifetime and break any `HashSet` holding them. The trade-off — every entity of a type hashing identically — is the right one at this scale.

**Repositories return `Slice`, not `Page`.** `findAllBy(Pageable)` skips the `COUNT` query, which matters once the table is large and the UI doesn't need a total.

**Security is method-level, not just route-level.** `TaskService` is annotated `@PreAuthorize("isAuthenticated()")` at the class level, so the guard survives even if a view forgets `@PermitAll`.

**Navigation is generated, not hardcoded.** `TaskListView` declares `@Menu(order = 0, icon = "vaadin:clipboard-check", title = "Task List")`, and `MainLayout` builds the side nav from `MenuConfiguration.getMenuEntries()`. New views appear in the menu automatically.

### Architecture tests

`ArchitectureTest` enforces boundaries with ArchUnit rather than convention alone:

- Domain classes cannot depend on services or UI
- Services cannot depend on UI
- Repositories can only be touched from domain or service packages
- Repository methods can only be called from `@Transactional` methods
- No circular dependencies between feature packages
- The security package should not depend on other application code

That fourth rule is the interesting one — it catches the classic mistake of calling a repository straight from a Vaadin view with no transaction boundary.

---

## Current feature scope

`TaskListView` provides a description field, an optional due date picker, a create button, and a Grid listing description, due date, and creation date. That's the whole feature.

Not yet built:

- No edit or delete — tasks are create-and-read only
- No task ownership — see below
- No completion status, priority, tags, or assignment
- No search or filtering, though `JpaSpecificationExecutor` is already wired up for it
- No database migrations

---

## Known gaps

**One architecture test silently passes.** `security_package_should_not_depend_on_other_application_classes()` builds its rule but never calls `.check(importedClasses)` — the chain ends on `.because(...)`. Every other test in the class ends with `.check(...)`. As written, this test cannot fail no matter what the security package depends on. Add the missing call, then expect it to actually fail: `security/dev/SampleUsers` imports `AppRoles` and `UserId`, both inside `security`, so it may pass — but verify rather than assume.

**Tasks have no owner.** `Task` has no user reference, and `TaskListView` is `@PermitAll`. Every authenticated user sees and can create tasks in one shared global list. The security infrastructure to fix this is already in place — `CurrentUser.require().getUserId()` — but the domain doesn't use it. If this is ever multi-user for real, an `ownerId` column plus a filter in `TaskService.list()` is the minimum, and `@PostAuthorize`/specification-level filtering is the more robust version.

**A test hook is wired into production code.** `TaskService.createTask()` opens with:

```java
if ("fail".equals(description)) {
    throw new RuntimeException("This is for testing the error handler");
}
```

That's starter scaffolding for exercising `MainErrorHandler`. Remove it before this goes anywhere real.

**`ddl-auto=update` is on.** The starter's own comment in `application.properties` says not to do this in production. Hibernate's schema updater never drops or alters columns safely and gives you no migration history. Flyway is the documented next step and should land before the schema gets complicated.

**Sample user IDs are regenerated every restart.** `SampleUsers.ADMIN_ID` and `USER_ID` call `UUID.randomUUID()` at class-initialization time. Harmless while nothing is keyed to user ID, but the moment tasks get an owner, every H2 restart orphans the previous run's data. Hardcode the UUIDs.

**Dockerfile is single-stage and runs as root.** `COPY target/*.jar app.jar` also assumes exactly one jar matches. A multi-stage build that runs Maven inside the image, plus a non-root `USER`, would be sturdier — and a `.dockerignore` is missing entirely.

**Starter TODOs remain throughout.** `MainView` ("TODO Replace with your own main view"), `MainLayout` ("TODO Replace with real application logo"), `TestcontainersConfiguration`, and `ArchitectureTest` all still carry their scaffolding markers.

---

## License

Released into the public domain — see `LICENSE.md`.
