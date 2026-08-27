# DevPilot

> An AI-powered developer assistant built around Retrieval-Augmented Generation (RAG), with a Spring Boot backend, a Next.js/React frontend, PostgreSQL with pgvector, and containerized local infrastructure.

## Project Status

DevPilot is currently under active development.

The repository has established the initial full-stack foundation:

- Spring Boot backend
- Next.js + React client
- PostgreSQL development database
- pgvector-enabled PostgreSQL image
- Spring Data JPA persistence foundation
- User domain model
- User repository
- Initial service layer
- Custom exception types and centralized exception handling
- TanStack React Query provider
- Theme provider and shadcn-based UI foundation
- Spring AI dependency setup

The RAG pipeline, GitHub OAuth flow, developer-facing AI workflows, and production architecture are planned areas of the project rather than completed features at this stage.

---

## 1. What is DevPilot?

DevPilot is intended to provide developers with an AI-assisted workspace that can answer questions and assist with software development using contextual information from indexed code, documents, and knowledge repositories.

The central idea is **Retrieval-Augmented Generation (RAG)**:

```text
Developer
   |
   v
DevPilot Client
   |
   v
Spring Boot API
   |
   +--------------------+
   |                    |
   v                    v
User / Application    RAG Pipeline
Data                  |
                      +--> Document ingestion
                      +--> Chunking
                      +--> Embeddings
                      +--> Vector search
                      +--> Context retrieval
                      |
                      v
                    LLM
                      |
                      v
                   Response
```

The diagram represents the intended direction of the system. Only the foundational pieces are currently implemented.

---

## 2. Repository Structure

```text
DevPilot/
├── backend/
│   ├── .mvn/
│   ├── compose.yaml
│   ├── mvnw
│   ├── mvnw.cmd
│   ├── pom.xml
│   └── src/
│       ├── main/
│       │   ├── java/
│       │   │   └── devPilot/backend/
│       │   │       ├── BackendApplication.java
│       │   │       ├── entiity/
│       │   │       │   └── User.java
│       │   │       ├── exceptions/
│       │   │       │   ├── BadRequestException.java
│       │   │       │   ├── GlobalExceptionHandler.java
│       │   │       │   ├── NotFoundException.java
│       │   │       │   └── UnauthorizedException.java
│       │   │       ├── repository/
│       │   │       │   └── UserRepository.java
│       │   │       └── services/
│       │   │           └── UserService.java
│       │   └── resources/
│       │       └── application.properties
│       └── test/
│
├── client/
│   ├── app/
│   ├── components/
│   │   ├── providers/
│   │   └── ui/
│   ├── hooks/
│   ├── lib/
│   ├── next.config.ts
│   ├── package.json
│   ├── package-lock.json
│   ├── pnpm-lock.yaml
│   └── ...
│
├── DOCKER/
│   └── POSTGRES/
│       └── init-extensions.sql
│
├── docker-compose.yaml
└── README.md
```

### Important naming note

The backend currently uses the package directory:

```text
entiity/
```

This appears to be a spelling mistake for `entity`. It is part of the current source tree and should be renamed deliberately later rather than casually changing package names.

---

# 3. Backend

## 3.1 Technology

The backend currently uses:

| Technology | Purpose |
|---|---|
| Java 21 | Backend language/runtime |
| Spring Boot 4.1.1 | Application framework |
| Maven | Dependency and build management |
| Spring Web MVC | HTTP/API layer |
| Spring Data JPA | ORM/repository abstraction |
| PostgreSQL | Relational database |
| Spring Security | Security foundation |
| OAuth2 Client | OAuth client support |
| Spring AI 2.0.1 | AI integration foundation |
| Lombok | Boilerplate reduction |
| Spring Boot DevTools | Development support |
| Docker Compose integration | Local infrastructure support |

The current Maven configuration contains these dependencies and versions. The Spring AI BOM manages the Spring AI dependency versions.

---

## 3.2 Application Entry Point

The backend starts from:

```text
backend/src/main/java/devPilot/backend/BackendApplication.java
```

This is the Spring Boot application entry point.

---

# 4. Persistence Layer

The first persistence milestone establishes a `User` entity and a Spring Data repository.

## 4.1 User Entity

Current entity:

```java
@Entity
@Table(name = "users")
public class User {
    ...
}
```

The entity currently contains:

| Field | Type | Purpose |
|---|---|---|
| `id` | `UUID` | Internal DevPilot primary key |
| `githubId` | `Long` | GitHub user's identifier |
| `githubUsername` | `String` | GitHub username |
| `displayName` | `String` | User display name |
| `avatarUrl` | `String` | GitHub/avatar URL |
| `accessToken` | `String` | OAuth access token |
| `tokenScope` | `String` | Granted token scope |
| `createdAt` | `Instant` | Account creation timestamp |

The primary key uses:

```java
@GeneratedValue(strategy = GenerationType.UUID)
```

`githubId` is unique and non-null.

`createdAt` is populated through `@PrePersist`.

### Why two IDs?

The distinction is important:

```text
id
└── DevPilot's internal database identifier
    Type: UUID

githubId
└── GitHub's identifier for the external user
    Type: Long
```

These identifiers represent different domains and should not be confused.

---

## 4.2 Lombok

The entity currently uses Lombok annotations including:

```text
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
```

This removes repetitive getter, setter, constructor, and builder code.

The Maven compiler configuration also enables Lombok annotation processing.

---

## 4.3 User Repository

Current repository:

```java
public interface UserRepository extends JpaRepository<User, UUID> {

    Optional<User> findByGithubId(Long githubId);

}
```

The repository therefore gets the standard CRUD operations from:

```text
JpaRepository<User, UUID>
```

and adds a derived query:

```text
findByGithubId(...)
```

Spring Data JPA derives the database query from the method name.

Conceptually:

```text
findByGithubId(12345L)
        |
        v
SELECT user
FROM users
WHERE github_id = 12345
```

The repository returns:

```java
Optional<User>
```

so callers can explicitly handle the "user does not exist" case.

---

# 5. Service Layer

The current service is:

```text
services/UserService.java
```

It is annotated with:

```java
@Service
@RequiredArgsConstructor
```

The service currently contains:

- access to `UserRepository`
- access to a `TextEncryptor`
- a read-only transactional lookup method
- access-token decryption support
- a helper for converting values to `Long`

### Current implementation note

The method currently named:

```java
requiredByGithubId(UUID id)
```

uses:

```java
userRepository.findById(id)
```

This means the current implementation is actually looking up the **internal UUID primary key**, not `githubId`.

This should be corrected when the GitHub authentication flow is implemented. The repository already has the intended method:

```java
findByGithubId(Long githubId)
```

Documentation intentionally records the current implementation instead of pretending the service is already complete.

---

# 6. Exception Handling

DevPilot currently has three custom runtime exceptions:

```text
BadRequestException
NotFoundException
UnauthorizedException
```

They represent common HTTP/API failure categories:

| Exception | HTTP status |
|---|---:|
| `BadRequestException` | 400 |
| `NotFoundException` | 404 |
| `UnauthorizedException` | 401 |

The project also has:

```text
GlobalExceptionHandler
```

using:

```java
@RestControllerAdvice
```

This provides centralized exception-to-response handling.

The current handler covers:

- not-found errors
- bad-request errors
- unauthorized errors
- validation errors
- unexpected exceptions

The standardized response currently contains:

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "...",
  "timestamp": "..."
}
```

The exact message depends on the exception.

### Why centralize exception handling?

Without a global handler, every controller would need repetitive `try/catch` logic.

Instead:

```text
Controller / Service
        |
        | throws exception
        v
GlobalExceptionHandler
        |
        v
HTTP status + structured response
```

This keeps business logic separate from HTTP error formatting.

---

# 7. Database

DevPilot uses PostgreSQL for local development.

The root Docker Compose configuration uses:

```text
Image: pgvector/pgvector:pg16
Database: devpilot
Username: postgres
Password: postgres
Container: devpilot-postgres
```

The container maps:

```text
Host:      5433
Container: 5432
```

Therefore the backend connects to:

```text
jdbc:postgresql://localhost:5433/devpilot
```

A named Docker volume is used:

```text
devpilot_pg_data
```

so database data survives container recreation.

A PostgreSQL initialization script is also mounted into the container to initialize required extensions.

---

# 8. Spring Configuration

The backend currently uses:

```properties
spring.application.name=backend

spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5433/devpilot}
spring.datasource.username=${DB_USERNAME:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.datasource.driver-class-name=org.postgresql.Driver

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false

spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-4o-mini
spring.ai.openai.embedding.model=text-embedding-3-small
```

## Environment variable pattern

The datasource configuration uses Spring's default-value syntax:

```text
${VARIABLE:default}
```

For example:

```properties
spring.datasource.username=${DB_USERNAME:postgres}
```

means:

```text
Use DB_USERNAME if it exists.
Otherwise use postgres.
```

This makes local development easier while allowing deployment environments to provide their own configuration.

The OpenAI API key does **not** have a default value:

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
```

It should therefore be supplied through the environment.

Never commit actual API keys, OAuth secrets, or access tokens to Git.

---

# 9. Docker Development

Start PostgreSQL from the repository root:

```powershell
docker compose up -d
```

Check running containers:

```powershell
docker ps
```

Stop the services:

```powershell
docker compose down
```

To stop and remove the database volume as well:

```powershell
docker compose down -v
```

**Warning:** removing the volume deletes the local PostgreSQL data stored in that Docker volume.

---

# 10. Maven Development

The backend contains the Maven Wrapper:

```text
mvnw
mvnw.cmd
```

On Windows PowerShell:

```powershell
cd backend
.\mvnw.cmd clean compile
```

Run tests:

```powershell
.\mvnw.cmd test
```

Run the application:

```powershell
.\mvnw.cmd spring-boot:run
```

The Maven Wrapper is preferable to requiring every developer to install a matching global Maven version.

---

# 11. Frontend

The client is a Next.js application using React and TypeScript.

Current package versions include:

- Next.js 16.3.2
- React 19.2.8
- React DOM 19.2.8
- TypeScript 5
- Tailwind CSS 4
- TanStack React Query 5
- next-themes
- shadcn
- Lucide React
- Recharts
- date-fns
- Embla Carousel
- React Day Picker
- React Resizable Panels
- various supporting UI packages

The frontend currently provides the application shell and UI foundation rather than the complete DevPilot product.

---

# 12. Frontend Architecture

The current client structure separates several responsibilities:

```text
client/
├── app/
│   └── Application routes / layouts
│
├── components/
│   ├── providers/
│   │   ├── query-provider
│   │   └── theme-provider
│   │
│   └── ui/
│       └── reusable UI components
│
├── hooks/
│   └── reusable React hooks
│
└── lib/
    └── shared utilities
```

---

## 12.1 Next.js App Router

The application uses the Next.js `app` directory.

The root layout is responsible for global application setup such as:

- fonts
- global CSS
- theme handling
- React Query context

---

# 13. TanStack React Query

The frontend contains a custom:

```text
QueryProvider
```

It creates a `QueryClient` and exposes it through:

```tsx
<QueryClientProvider client={query}>
    {children}
</QueryClientProvider>
```

Conceptually:

```text
React Application
       |
       v
QueryProvider
       |
       v
QueryClient
       |
       +--> server-state fetching
       +--> caching
       +--> request lifecycle
       +--> refetching
```

React Query is intended to manage server state rather than replacing normal React state.

For DevPilot, this will become useful once the frontend begins communicating with the Spring Boot API.

---

# 14. Theme System

The client uses `next-themes` through a theme provider.

The root layout wraps the application with:

```text
ThemeProvider
```

The current frontend also contains a mode toggle component.

This establishes light/dark/system theme support at the application level.

---

# 15. UI System

The client contains a collection of reusable UI components under:

```text
components/ui/
```

These components are largely infrastructure generated/configured around the shadcn ecosystem.

Examples include components for:

- buttons
- cards
- dialogs
- forms
- inputs
- menus
- navigation
- alerts
- avatars
- calendars
- carousels
- charts
- tables
- tooltips

These components should be treated as **UI building blocks**, not individual DevPilot business features.

---

# 16. Intended RAG Architecture

The repository description identifies RAG as the core direction of DevPilot.

The intended pipeline is:

```text
Source
  |
  v
Document / Repository ingestion
  |
  v
Text extraction
  |
  v
Chunking
  |
  v
Embedding model
  |
  v
Vector storage
  |
  v
Similarity search
  |
  v
Relevant context
  |
  v
Prompt construction
  |
  v
LLM
  |
  v
Developer response
```

PostgreSQL is already configured using a pgvector-enabled image, which provides the infrastructure needed for vector storage.

The actual ingestion, embedding, retrieval, and generation pipeline is not yet implemented in the current repository.

---

# 17. AI Model Configuration

The current backend configuration is prepared for OpenAI:

```properties
spring.ai.openai.chat.model=gpt-4o-mini
spring.ai.openai.embedding.model=text-embedding-3-small
```

Spring AI dependencies are already present in `pom.xml`.

The project may later evaluate local/free models such as Ollama-backed models. That decision should be documented separately once an actual model is selected and integrated.

Do not document a model as part of the production architecture until it has been tested inside DevPilot.

---

# 18. Authentication Direction

The backend already includes:

```text
Spring Security
OAuth2 Client
```

The `User` entity also contains GitHub-specific fields:

```text
githubId
githubUsername
avatarUrl
accessToken
tokenScope
```

This strongly establishes the intended GitHub authentication/integration direction.

The complete OAuth login flow is not yet implemented in the current repository.

Expected future flow:

```text
User
 |
 v
DevPilot Login
 |
 v
GitHub OAuth
 |
 v
GitHub authorization
 |
 v
Authorization callback
 |
 v
User lookup / creation
 |
 v
Persist user
 |
 v
Authenticated DevPilot session
```

---

# 19. Security Considerations

The `User` entity currently contains an `accessToken` field.

That makes token handling a security-sensitive part of the architecture.

The current service contains support for decrypting the stored access token using `TextEncryptor`.

Future implementation must ensure:

- tokens are never logged
- tokens are never returned to the frontend unnecessarily
- encryption keys are supplied through secure configuration
- secrets are not committed to Git
- OAuth scopes are minimized
- access tokens are handled only where required
- production secrets are managed outside source control

The current local PostgreSQL credentials are development defaults and should not be treated as production credentials.

---

# 20. Development History

This section is intentionally chronological. It should be updated after every meaningful development milestone.

## 2026-08-25 — Initial Project Setup

The repository was initialized with the initial DevPilot project structure.

Initial foundation included the backend and repository-level development setup.

---

## 2026-08-25 — Client Source Added

Commit:

```text
Track client source code
```

The client source was added to the repository.

The frontend foundation includes Next.js, React, TypeScript, Tailwind CSS, shadcn-based UI infrastructure, theme support, and the initial application structure.

---

## 2026-08-27 — JPA User Persistence and Exception Handling

Commit:

```text
feat: set up JPA user persistence and exception handling
```

The backend persistence foundation was introduced.

### Added

- `User` JPA entity
- PostgreSQL/JPA integration
- `UserRepository`
- `UserService`
- `BadRequestException`
- `NotFoundException`
- `UnauthorizedException`
- `GlobalExceptionHandler`
- validation error handling
- standardized API error response structure

The `User` entity was later corrected so that:

```text
githubId -> Long
```

and:

```java
findByGithubId(Long githubId)
```

matches the entity field type.

The distinction between the internal `UUID id` and external `Long githubId` was explicitly corrected during development.

---

# 21. Current Architecture Snapshot

As of the current repository state:

```text
                       DEV PILOT
                           |
             +-------------+-------------+
             |                           |
             v                           v
        Next.js Client              Spring Boot
             |                           |
             |                           +----------------+
             |                           |                |
             v                           v                v
      React Query / UI             Service Layer    Exception Layer
                                         |
                                         v
                                  Spring Data JPA
                                         |
                                         v
                                    PostgreSQL
                                         |
                                         v
                                      pgvector

Spring AI
    |
    v
AI / RAG integration foundation
```

This is a **foundation snapshot**, not a claim that all displayed components are already connected end-to-end.

---

# 22. Planned Development Roadmap

The roadmap should evolve as implementation progresses.

## Phase 1 — Foundation

- [x] Repository initialized
- [x] Backend created
- [x] Client created
- [x] Docker/PostgreSQL setup
- [x] JPA dependency
- [x] User entity
- [x] User repository
- [x] Initial service layer
- [x] Exception layer
- [x] React Query provider
- [x] Theme provider
- [x] UI component foundation

## Phase 2 — Authentication

- [ ] GitHub OAuth configuration
- [ ] OAuth callback
- [ ] User creation/update
- [ ] Secure token handling
- [ ] Authentication/session strategy
- [ ] Protected backend endpoints
- [ ] Frontend authentication state

## Phase 3 — Repository Integration

- [ ] Connect GitHub repositories
- [ ] Repository metadata
- [ ] File retrieval
- [ ] File filtering
- [ ] Repository synchronization
- [ ] Ingestion pipeline

## Phase 4 — RAG

- [ ] Document model
- [ ] Chunk model
- [ ] Embedding generation
- [ ] pgvector storage
- [ ] Similarity search
- [ ] Retrieval service
- [ ] Context construction
- [ ] LLM generation

## Phase 5 — Developer Assistant

- [ ] Chat interface
- [ ] Context-aware answers
- [ ] Code explanation
- [ ] Repository questions
- [ ] Error/debugging assistance
- [ ] Codebase navigation
- [ ] Source references/citations

## Phase 6 — Production Readiness

- [ ] Configuration management
- [ ] Database migrations
- [ ] Security hardening
- [ ] Observability
- [ ] Automated tests
- [ ] CI/CD
- [ ] Containerized deployment
- [ ] Production database
- [ ] Performance testing

---

# 23. Current Known Issues / Technical Debt

These are deliberately recorded so future development does not lose track of them.

### 1. `entiity` package naming

Current package:

```text
devPilot.backend.entiity
```

Expected conventional spelling:

```text
devPilot.backend.entity
```

This should be renamed carefully because it affects Java package declarations and imports.

### 2. `UserService` GitHub lookup mismatch

Current service method:

```java
requiredByGithubId(UUID id)
```

currently calls:

```java
userRepository.findById(id)
```

This is an internal UUID lookup, not a GitHub ID lookup.

The repository already exposes:

```java
findByGithubId(Long githubId)
```

The service should be aligned with that when the authentication flow is implemented.

### 3. Docker path casing

The repository stores the Docker directory as:

```text
DOCKER/POSTGRES/
```

while the root Compose file references:

```text
./docker/postgres/init-extensions.sql
```

Windows is generally case-insensitive, so this can work locally. Linux filesystems are commonly case-sensitive.

The path should be normalized before production/Linux CI use.

### 4. Multiple frontend lockfiles

The client currently contains both:

```text
package-lock.json
pnpm-lock.yaml
```

The project should eventually standardize on one package manager and one lockfile.

### 5. Database schema strategy

The current configuration uses:

```properties
spring.jpa.hibernate.ddl-auto=update
```

This is convenient during development but should not be treated as the final production database migration strategy.

A migration system such as Flyway should be established before production deployment.

---

# 24. Git Commit Convention

Use concise, descriptive conventional commits.

Recommended format:

```text
type: short description
```

Examples:

```text
feat: add GitHub OAuth authentication
feat: add repository ingestion pipeline
feat: implement vector similarity search

fix: correct GitHub user identifier type
fix: handle missing user in service layer

refactor: rename entiity package to entity

test: add user repository tests

docs: update development journal
```

The commit message should describe **what changed**, while this README records **why it changed and how it fits into the architecture**.

---

# 25. How to Explain DevPilot in an Interview

A concise technical explanation:

> DevPilot is a full-stack AI developer assistant designed around Retrieval-Augmented Generation. The frontend is built with Next.js, React, TypeScript, Tailwind, and a reusable component system. The backend uses Spring Boot with Spring Data JPA and PostgreSQL. PostgreSQL is configured with pgvector so repository and document embeddings can eventually be stored and searched using vector similarity. Spring AI provides the AI integration layer. The application is being built incrementally, starting with authentication and persistence before implementing repository ingestion and the RAG pipeline.

### Architecture explanation

If asked:

**"Why RAG?"**

Answer:

> A general-purpose LLM does not automatically know the current contents of a user's private codebase or documentation. RAG allows DevPilot to retrieve relevant project context and provide that context to the model before generation.

**"Why PostgreSQL?"**

> DevPilot needs relational persistence for application data such as users, repositories, and metadata. Using PostgreSQL with pgvector also provides a path to storing and querying embeddings without introducing a separate vector database initially.

**"Why Spring Data JPA?"**

> It provides a clean repository abstraction over PostgreSQL and reduces boilerplate for CRUD and query operations while keeping persistence logic separate from service logic.

**"Why a service layer?"**

> The service layer keeps business logic between controllers and repositories. Controllers handle HTTP concerns, repositories handle persistence, and services coordinate application behavior.

---

# 26. Development Principles

The project should follow these principles as it grows:

1. **Do not mix business logic with controllers.**
2. **Keep persistence concerns inside repositories/entities.**
3. **Use services for application/business logic.**
4. **Keep API error handling centralized.**
5. **Never commit secrets.**
6. **Prefer explicit architecture over unnecessary abstractions.**
7. **Document architectural decisions, not every line of code.**
8. **Record problems encountered during development.**
9. **Test important business behavior instead of only testing framework wiring.**
10. **Do not mark planned functionality as implemented.**

---

# 27. Documentation Journal Rules

This README is intended to become the long-term technical record for DevPilot.

After every significant development session, update the development history with:

```text
Date
Commit
What changed
Why it changed
How it works
Important decisions
Problems encountered
How they were solved
Remaining work
```

Example:

```text
## YYYY-MM-DD — Feature Name

### What changed
...

### Why
...

### How it works
...

### Problems encountered
...

### Solution
...

### Remaining work
...
```

This makes the README useful both as project documentation and as a future interview/preparation reference.

---

# 28. Local Development Checklist

Before starting backend development:

```text
[ ] Docker Desktop running
[ ] PostgreSQL container running
[ ] Database accessible on localhost:5433
[ ] Java 21 available
[ ] Backend dependencies resolved
[ ] Environment variables configured
```

Before starting frontend development:

```text
[ ] Node.js installed
[ ] Chosen package manager installed
[ ] Dependencies installed
[ ] Client development server running
```

---

# 29. Useful Commands

## PostgreSQL

```powershell
docker compose up -d
docker ps
docker compose logs postgres
docker compose down
```

## Backend

```powershell
cd backend

.\mvnw.cmd clean compile
.\mvnw.cmd test
.\mvnw.cmd spring-boot:run
```

## Frontend

From `client/`, the available scripts are:

```powershell
npm run dev
npm run build
npm run start
npm run lint
```

If the project is standardized on pnpm later, use the equivalent `pnpm` commands and keep only the selected package manager's lockfile.

---

# 30. Final Architecture Goal

The eventual DevPilot system should evolve toward:

```text
                         +-------------------+
                         |      Developer    |
                         +---------+---------+
                                   |
                                   v
                         +-------------------+
                         |   Next.js Client  |
                         +---------+---------+
                                   |
                              HTTP / API
                                   |
                                   v
                         +-------------------+
                         | Spring Boot API   |
                         +---------+---------+
                                   |
                 +-----------------+------------------+
                 |                 |                  |
                 v                 v                  v
          Authentication       Application        RAG Pipeline
                 |                 |                  |
                 v                 v                  v
            GitHub OAuth       PostgreSQL        Ingestion
                                                     |
                                                     v
                                                  Chunking
                                                     |
                                                     v
                                                 Embeddings
                                                     |
                                                     v
                                                  pgvector
                                                     |
                                                     v
                                              Similarity Search
                                                     |
                                                     v
                                               Context Builder
                                                     |
                                                     v
                                                   LLM
                                                     |
                                                     v
                                                Response
```

The important architectural goal is not simply "add an LLM."

The goal is to build a system where the model can reason over **relevant, retrieved, user-specific software context** while the backend handles authentication, persistence, security, retrieval, and orchestration.

---

## License

No project license is currently declared in the repository. Add a license when the project's distribution terms are decided.

---

## Repository

DevPilot is maintained in the `purnenduachary/DevPilot` GitHub repository.

This document describes the repository's current implementation and explicitly separates implemented functionality from planned architecture.
