# DevPilot

> An AI-powered developer assistant built around Retrieval-Augmented Generation (RAG), with a Spring Boot backend, a Next.js/React frontend, PostgreSQL with pgvector, and containerized local infrastructure.

## Project Status

DevPilot is currently under active development.

The repository now has a working full-stack foundation plus the first functional application flows:

- Spring Boot backend
- Next.js + React client
- PostgreSQL + pgvector development infrastructure
- Spring Data JPA persistence
- GitHub OAuth2 authentication and session flow
- Protected frontend routes and authentication UI
- Shared frontend application shell
- Typed frontend API client
- GitHub repository fetch and synchronization
- Local repository persistence and ownership checks
- Repository indexing state model and asynchronous indexing entry point
- GitHub repository tree and file-content access
- TanStack React Query state management
- Theme provider and shadcn-based UI foundation
- Spring AI configuration foundation

The RAG retrieval pipeline, production hardening, and full AI developer workflow are still in progress.

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
│   ├── compose.yaml
│   ├── mvnw
│   ├── mvnw.cmd
│   ├── pom.xml
│   └── src/main/java/devPilot/backend/
│       ├── config/
│       ├── controllers/
│       ├── dto/
│       ├── entity/
│       ├── exceptions/
│       ├── repository/
│       ├── security/
│       └── services/
│           ├── github/
│           └── indexing/
│
├── client/
│   ├── app/
│   ├── components/
│   │   ├── dashboard/
│   │   ├── icons/
│   │   ├── layout/
│   │   ├── providers/
│   │   └── ui/
│   ├── hooks/
│   ├── lib/
│   ├── proxy.ts
│   ├── package.json
│   └── ...
│
├── docker/
│   └── postgres/
│       └── init-extensions.sql
│
├── docker-compose.yaml
└── README.md
```

The backend follows the current layered package structure: configuration, controllers, DTOs, entities, exception handling, repositories, security, and services. GitHub-specific integration lives under `services/github/`, while indexing logic is separated under `services/indexing/`.

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

The service layer contains application business logic.

Current services include:

```text
services/
├── UserService
├── RepoService
├── github/
│   ├── GithubApiClient
│   └── GitHubRateLimiter
└── indexing/
    └── indexing-related services
```

### Repository synchronization

`RepoService` now:

- loads the authenticated user
- decrypts the stored GitHub token
- fetches repositories from GitHub
- upserts repository metadata into PostgreSQL
- checks repository ownership
- maps entities to API DTOs
- exposes repository/indexing status

The main flow is:

```text
Controller
   ↓
RepoService
   ↓
GithubApiClient + RepositoryRepository
   ↓
GitHub + PostgreSQL
```

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

The client is a **Next.js application using React and TypeScript**. The current frontend is the application shell and UI foundation; it is not yet the complete DevPilot product.

Current frontend technologies include:

| Technology / Package | Role |
|---|---|
| Next.js | React framework and application routing |
| React | UI component model |
| TypeScript | Static typing |
| Tailwind CSS | Utility-first styling |
| TanStack React Query | Server-state management |
| `next-themes` | Light/dark/system theme management |
| shadcn/ui | Reusable UI component foundation |
| Lucide React | Icons |
| Recharts | Charts |
| date-fns | Date utilities |
| Embla Carousel | Carousel behavior |
| React Day Picker | Date-picker/calendar functionality |
| React Resizable Panels | Resizable layouts |

The repository currently contains the frontend dependencies and supporting UI infrastructure described above. fileciteturn67file2L564-L586

---

# 12. Frontend Structure

The important frontend directories are:

```text
client/
├── app/
│   └── Next.js application routes and layouts
│
├── components/
│   ├── providers/
│   │   ├── query-provider
│   │   └── theme-provider
│   │
│   └── ui/
│       └── reusable UI primitives
│
├── hooks/
│   └── reusable React hooks
│
└── lib/
    └── shared utilities
```

The separation is intentional:

```text
app/
  -> pages, layouts, routing

components/
  -> reusable UI and providers

hooks/
  -> reusable React logic

lib/
  -> utilities / shared helpers
```

The current repository structure explicitly separates providers, UI components, hooks, and shared utilities. fileciteturn67file2L590-L612

---

# 13. Next.js App Router

The client uses the Next.js `app` directory.

The root layout is the application-level composition point. It is responsible for global concerns such as:

- fonts
- global CSS
- theme provider
- React Query provider

This means individual pages do not need to recreate these global providers.

Conceptually:

```text
Next.js Application
        |
        v
     layout
        |
        +--> ThemeProvider
        |
        +--> QueryProvider
        |
        v
     page/routes
```

The current project uses the App Router structure and places application-wide setup in the root layout. fileciteturn67file2L616-L625

---

# 14. TanStack React Query

DevPilot includes a custom `QueryProvider`.

The provider creates a `QueryClient` and makes it available to the React component tree through `QueryClientProvider`.

The basic structure is:

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import React from "react";

const QueryProvider = ({ children }: { children: React.ReactNode }) => {
    const [query] = React.useState(() => new QueryClient());

    return (
        <QueryClientProvider client={query}>
            {children}
        </QueryClientProvider>
    );
};

export default QueryProvider;
```

## What is `QueryClient`?

`QueryClient` is the central object used by TanStack React Query to manage server state.

It handles things such as:

- fetching data
- caching responses
- request lifecycle
- refetching
- tracking loading/error states

Conceptually:

```text
React Components
       |
       v
QueryProvider
       |
       v
QueryClient
       |
       +--> fetch
       +--> cache
       +--> refetch
       +--> loading/error state
```

React Query is **not a replacement for `useState`**.

A useful distinction is:

```text
useState
  -> local UI/application state

React Query
  -> remote/server state
```

For DevPilot, React Query becomes especially useful when the frontend starts consuming Spring Boot API endpoints. fileciteturn68file0L16-L51

---

# 15. Theme Provider

The frontend uses `next-themes` through a custom theme provider.

Its purpose is to make the application's theme available globally.

The project supports the concept of:

```text
Light
Dark
System
```

The provider is placed near the root of the application so individual components can access theme state without manually passing it through props.

Conceptually:

```text
Root Layout
     |
     v
ThemeProvider
     |
     +--> Header
     +--> Sidebar
     +--> Pages
     +--> UI components
```

The current client also contains a mode-toggle component for changing the theme. fileciteturn68file0L55-L67

---

# 16. UI Component System

The client contains reusable UI components under:

```text
components/ui/
```

These components are primarily built around the shadcn/ui ecosystem.

Important point:

> These components are **infrastructure**, not DevPilot business logic.

For example:

```text
button
  -> reusable button primitive

card
  -> reusable content container

dialog
  -> reusable modal primitive

form
  -> reusable form structure

sidebar
  -> reusable navigation/layout primitive

table
  -> reusable tabular-data component
```

The repository includes UI primitives for areas such as:

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

The purpose of this layer is to avoid rebuilding common UI behavior every time a DevPilot screen is created. fileciteturn68file0L71-L91

### Why this matters

Instead of:

```text
Page A -> custom button
Page B -> another custom button
Page C -> another custom button
```

the application can use:

```text
                 UI primitive
                      |
          +-----------+-----------+
          |           |           |
        Page A      Page B      Page C
```

This keeps the UI more consistent and makes future changes easier.

---

# 17. Frontend Libraries — Why They Were Added

The frontend contains several libraries that may initially look unrelated. Their roles are different:

| Library | Why it exists |
|---|---|
| TanStack Query | API/server-state management |
| next-themes | Theme management |
| Lucide React | Consistent icon set |
| Recharts | Data visualization |
| date-fns | Date manipulation/formatting |
| Embla Carousel | Carousel interactions |
| React Day Picker | Calendar/date selection |
| React Resizable Panels | Resizable UI layouts |
| shadcn/ui components | Reusable application UI |

Not every dependency represents a completed DevPilot feature. Some are foundational components available for future screens.

---

# 18. Backend Configuration — `application.properties`

The backend configuration is located at:

```text
backend/src/main/resources/application.properties
```

Current configuration:

```properties
spring.application.name=backend

# Defaults match docker-compose.yml (postgres on host port 5433)
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

---

## 18.1 Application Name

```properties
spring.application.name=backend
```

This gives the Spring Boot application its application name.

It is primarily application metadata/configuration and does not determine the Java package name.

---

## 18.2 Database URL

```properties
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5433/devpilot}
```

This uses Spring's environment-variable/default-value syntax:

```text
${VARIABLE:default}
```

Therefore:

```text
If DB_URL exists:
    use DB_URL

Otherwise:
    jdbc:postgresql://localhost:5433/devpilot
```

The default points to PostgreSQL exposed by Docker on host port `5433`.

The distinction is:

```text
Host machine
localhost:5433
      |
      v
Docker container
5432
```

---

## 18.3 Database Credentials

```properties
spring.datasource.username=${DB_USERNAME:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
```

The same pattern allows local defaults while allowing deployment environments to override them.

Local default:

```text
username = postgres
password = postgres
```

These are development credentials only.

Production credentials should be provided through secure environment/configuration management.

---

## 18.4 PostgreSQL JDBC Driver

```properties
spring.datasource.driver-class-name=org.postgresql.Driver
```

This tells Spring/JDBC which driver implementation should be used to communicate with PostgreSQL.

---

## 18.5 Hibernate Schema Management

```properties
spring.jpa.hibernate.ddl-auto=update
```

During development, Hibernate can compare the entity model with the database schema and update the schema when possible.

Current project strategy:

```text
Java Entity
     |
     v
Hibernate / JPA
     |
     v
PostgreSQL schema
```

This is convenient during early development.

It should eventually be replaced with a controlled migration strategy using Flyway.

---

## 18.6 SQL Logging

```properties
spring.jpa.show-sql=false
```

SQL statements are not printed through Hibernate's basic SQL display mechanism.

This keeps normal development logs cleaner.

---

## 18.7 SQL Formatting

```properties
spring.jpa.properties.hibernate.format_sql=true
```

When Hibernate SQL is displayed through supported logging/configuration, this requests formatted SQL rather than a compressed single-line representation.

---

## 18.8 Open Session in View

```properties
spring.jpa.open-in-view=false
```

This disables Spring's Open Session in View pattern.

The intention is to avoid keeping the persistence context open through the entire web request and instead make database access boundaries more explicit.

This encourages the application to load the data it needs inside the appropriate service/transaction boundary.

---

# 19. Spring AI Configuration

The backend is currently configured for OpenAI:

```properties
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-4o-mini
spring.ai.openai.embedding.model=text-embedding-3-small
```

The API key is read from:

```text
OPENAI_API_KEY
```

No API key should be committed to Git.

There are two different AI responsibilities here:

```text
Chat model
    |
    v
Generate responses

Embedding model
    |
    v
Convert text into vectors
```

This distinction becomes important when implementing the RAG pipeline.

The project already includes Spring AI dependencies and has pgvector-backed PostgreSQL infrastructure, but the complete RAG pipeline is still future work.

---

# 20. Environment Variables

The project uses environment variables where configuration should be changeable without modifying source code.

Current examples:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
OPENAI_API_KEY
```

The database properties provide development defaults:

```properties
${DB_URL:default}
${DB_USERNAME:default}
${DB_PASSWORD:default}
```

The OpenAI key intentionally has no hard-coded fallback:

```properties
${OPENAI_API_KEY}
```

### Security rule

Never commit:

```text
API keys
OAuth client secrets
Access tokens
Production passwords
Encryption keys
```

to the repository.

---

# 21. Development History

## 2026-08-25 — Initial Project Foundation

### Added

- Spring Boot backend
- Next.js/React client
- Docker/PostgreSQL development setup
- Initial project structure

### Purpose

Establish the full-stack foundation before implementing application-specific functionality.

---

## 2026-08-25 — Client Source and UI Foundation

### Added

- Next.js/React/TypeScript frontend structure
- Tailwind CSS
- shadcn-based UI primitives
- `next-themes`
- TanStack React Query
- QueryProvider
- ThemeProvider

### Purpose

Create reusable frontend infrastructure for the application.

---

## 2026-08-27 — JPA User Persistence and Exception Handling

### Added

- `User` entity
- `UserRepository`
- `UserService`
- PostgreSQL/JPA integration
- custom exceptions
- `GlobalExceptionHandler`

### Purpose

Establish persistent user data and standardized API error handling.

---

## 2026-09-16 — GitHub OAuth2 Authentication

### Added

- GitHub OAuth2 client integration
- authenticated user persistence
- `AppUserPrincipal`
- current-user access
- token encryption support
- authentication success/failure handling
- session/logout configuration

### Purpose

Connect DevPilot authentication to GitHub and establish the authenticated application session.

---

## 2026-09-23 — Frontend Shell, Landing Page & Auth Flow

### Added

- public landing page
- GitHub login page
- OAuth callback page
- reusable `AppShell`
- protected dashboard routes
- `RequireAuth`
- Next.js route guard
- authentication/API hooks
- repository dashboard UI
- overview/settings pages
- chat route and streaming client foundation
- frontend navigation and custom icons

### Purpose

Turn the client into a usable authenticated application shell.

---

## 2026-09-27 — GitHub Repository Fetch & Sync

### Added

- `RepoController`
- `Repository` entity
- `IndexStatus`
- `RepositoryRepository`
- `RepositoryResponse`
- `IndexStatusResponse`
- `RepoService`
- `GithubApiClient`
- `GitHubRateLimiter`

### Purpose

Connect authenticated users to their GitHub repositories, synchronize repository metadata into PostgreSQL, expose repository/status APIs, and establish the ingestion path for future code indexing.

### Current repository flow

```text
Authenticated User
      ↓
RepoController
      ↓
RepoService
      ↓
GithubApiClient
      ↓
GitHub REST API
      ↓
RepositoryRepository
      ↓
PostgreSQL
```

The repository tree and file-content access now provide the backend building blocks for the next indexing/RAG stage.

---

# 22. Current Architecture Snapshot

```text
                         DEV PILOT
                            |
             +--------------+--------------+
             |                             |
             v                             v
       Next.js Client                 Spring Boot
             |                             |
      +------+-------+              +------+-------+
      |              |              |              |
      v              v              v              v
   App Shell     React Query     Security      Service Layer
      |              |              |              |
      |              |              |        +-----+------+
      |              |              |        |            |
      |              |              |        v            v
      |              |              |   GitHub Client  JPA Repo
      |              |              |        |            |
      |              |              |        v            v
      |              |              |      GitHub     PostgreSQL
      |              |
      +--------------+-----------------------------+
                            |
                            v
                     RAG / AI foundation
```

Authentication and repository synchronization are now implemented. The RAG/AI pipeline is the next major application layer.

---

# 23. Known Technical Debt

### 1. Secrets and environment configuration

Production credentials, OAuth secrets, access tokens, and encryption keys must remain outside source control.

### 2. GitHub rate limiting

A configurable `GitHubRateLimiter` helper is present. It should be integrated consistently wherever indexing creates bursts of GitHub API requests.

### 3. Database migrations

The project currently uses:

```properties
spring.jpa.hibernate.ddl-auto=update
```

This is convenient during rapid development. Before production, introduce Flyway migrations and move toward controlled schema validation.

### 4. Frontend package-manager consistency

Standardize on one package manager and keep only its lockfile.

---

# 24. What Is Implemented vs Planned

## Implemented

### Backend

- Spring Boot backend foundation
- JPA user persistence
- centralized exception handling
- GitHub OAuth2 authentication
- authenticated session flow
- GitHub REST integration
- repository synchronization
- repository persistence
- repository ownership checks
- repository/indexing status API
- asynchronous indexing entry point
- repository tree retrieval
- file-content retrieval

### Frontend

- Next.js application shell
- landing page
- GitHub login page
- OAuth callback page
- authentication hooks
- `RequireAuth`
- Next.js `proxy.ts` route guard
- sidebar/navigation
- repository dashboard
- repository search/filtering
- indexing status/progress UI
- overview and settings screens
- typed API client
- React Query hooks/cache
- SSE-style chat streaming foundation

## Planned / In Progress

- repository ingestion pipeline
- source-file filtering
- code chunking
- embedding generation
- pgvector persistence/search
- retrieval service
- context construction
- grounded AI responses
- complete repository-aware chat workflow
- production migrations and deployment hardening

---

# 25. How to Explain the Frontend

If asked **"Why Next.js?"**:

> Next.js provides the React application framework, routing, layouts, and application structure required for the DevPilot client.

If asked **"Why React Query?"**:

> React Query manages server state such as API responses, caching, loading states, errors, and refetching. It is different from local React state, which is better suited for UI state.

If asked **"Why a QueryProvider?"**:

> TanStack React Query needs a QueryClient available to the component tree. The provider creates and supplies that client at the application level.

If asked **"Why shadcn?"**:

> It provides reusable UI primitives that can be composed into application-specific interfaces without rebuilding common components from scratch.

If asked **"Why next-themes?"**:

> It provides centralized theme management so light, dark, and system themes can be applied consistently across the application.

---

# 26. How to Explain the Backend

If asked **"Why JPA?"**:

> JPA provides the persistence model while Spring Data JPA provides repository abstractions and derived queries, reducing boilerplate database access code.

If asked **"Why UUID for the internal ID?"**:

> The UUID is DevPilot's internal identifier and is deliberately separate from the external GitHub identifier.

If asked **"Why a separate githubId?"**:

> GitHub owns that identifier. DevPilot should not use an external provider's identifier as its own database primary key.

If asked **"Why a service layer?"**:

> The service layer coordinates business logic between controllers and repositories. This prevents controllers from becoming tightly coupled to persistence details.

If asked **"Why GlobalExceptionHandler?"**:

> It centralizes HTTP error mapping so controllers and services can throw meaningful exceptions without duplicating response-formatting logic.

---

# 27. Future RAG Architecture

The intended RAG pipeline is:

```text
GitHub Repository / Documents
             |
             v
       Ingestion Service
             |
             v
        Text Extraction
             |
             v
           Chunking
             |
             v
      Embedding Generation
             |
             v
         PostgreSQL
          + pgvector
             |
             v
       Similarity Search
             |
             v
      Relevant Context
             |
             v
      Prompt Construction
             |
             v
            LLM
             |
             v
       Developer Answer
```

The important point is that DevPilot is not intended to simply send every user question directly to an LLM.

The system should first retrieve relevant information from the user's software context and then use that context to produce a grounded answer.

---

# 28. Development Documentation Rules

After every meaningful development session, add an entry containing:

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

```markdown
## YYYY-MM-DD — Feature Name

### What changed

### Why

### How it works

### Important decisions

### Problems encountered

### Solution

### Remaining work
```

This README is intended to become the project's long-term technical record and an explanation reference for future development and interviews.

---

# 29. Useful Development Commands

## Docker / PostgreSQL

```powershell
docker compose up -d
docker ps
docker compose logs postgres
docker compose down
```

Remove containers and the development database volume:

```powershell
docker compose down -v
```

Use the `-v` option carefully because it removes the local database volume.

## Backend

```powershell
cd backend

.\mvnw.cmd clean compile
.\mvnw.cmd test
.\mvnw.cmd spring-boot:run
```

## Frontend

From `client/`:

```powershell
npm run dev
npm run build
npm run start
npm run lint
```

If pnpm becomes the standardized package manager, use the equivalent pnpm commands.

---

# 30. Final Architecture Goal

```text
                         +-------------------+
                         |     Developer     |
                         +---------+---------+
                                   |
                                   v
                         +-------------------+
                         |   Next.js Client  |
                         +---------+---------+
                                   |
                         React Query / UI
                                   |
                                   v
                         +-------------------+
                         | Spring Boot API   |
                         +---------+---------+
                                   |
              +--------------------+--------------------+
              |                    |                    |
              v                    v                    v
        Authentication       Application Logic       RAG
              |                    |                    |
              v                    v                    v
        GitHub OAuth          PostgreSQL           Ingestion
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

The final goal is a developer assistant that can reason over **retrieved, user-specific software context**, rather than simply acting as a generic chatbot.

---

## License

No project license is currently declared.

---

## Repository

DevPilot is maintained in the `purnenduachary/DevPilot` repository.

This README intentionally distinguishes between **implemented functionality, development history, technical debt, and planned architecture**.


# 31. Development Roadmap

The roadmap tracks DevPilot from its initial foundation to a production-ready, repository-aware AI developer assistant.

```text
Phase 1 — Foundation       [██████████] 100%
Phase 2 — Authentication   [██████████] 100%
Phase 3 — GitHub           [████████░░]  80%
Phase 4 — RAG              [░░░░░░░░░░]   0%
Phase 5 — AI Assistant     [░░░░░░░░░░]   0%
Phase 6 — Frontend/UX      [██████░░░░]  60%
Phase 7 — Production       [░░░░░░░░░░]   0%
```

## Phase 1 — Foundation

**Status: 100%**

- [x] Initialize repository
- [x] Set up Spring Boot backend
- [x] Set up Next.js/React frontend
- [x] Configure TypeScript and Tailwind CSS
- [x] Add reusable UI component foundation
- [x] Add ThemeProvider
- [x] Add TanStack React Query / QueryProvider
- [x] Configure PostgreSQL with Docker
- [x] Configure pgvector
- [x] Configure JPA
- [x] Create initial `User` entity
- [x] Create `UserRepository`
- [x] Create `UserService`
- [x] Add application exception layer
- [x] Add global exception handling
- [x] Add Spring AI foundation

## Phase 2 — Authentication

**Status: 100%**

- [x] Configure GitHub OAuth
- [x] Implement OAuth login flow
- [x] Implement OAuth callback
- [x] Create authenticated session
- [x] Persist authenticated GitHub user
- [x] Add authentication/security configuration
- [x] Protect authenticated API routes
- [x] Add logout flow
- [x] Add frontend authentication guard

## Phase 3 — GitHub Integration

**Status: 80%**

- [x] Connect GitHub API
- [x] Retrieve user's repositories
- [x] Display repository list
- [x] Select repository for analysis/indexing
- [x] Retrieve repository tree and file contents
- [x] Handle repository metadata
- [x] Implement repository synchronization
- [ ] Fully integrate rate-limit handling into all indexing requests

## Phase 4 — RAG

**Status: 0%**

- [ ] Design document model
- [ ] Design chunk model
- [ ] Implement repository ingestion
- [ ] Extract source files
- [ ] Filter unsupported/generated files
- [ ] Chunk source code intelligently
- [ ] Generate embeddings
- [ ] Store embeddings in pgvector
- [ ] Implement similarity search
- [ ] Build retrieval service
- [ ] Construct relevant context for prompts
- [ ] Add source/file references to retrieved context

## Phase 5 — AI Assistant

**Status: 0%**

- [ ] Implement chat API
- [ ] Connect retrieval to chat
- [ ] Build prompt/context pipeline
- [ ] Add repository-aware questions
- [ ] Explain code
- [ ] Answer architecture questions
- [ ] Summarize files/classes
- [ ] Identify relevant source locations
- [ ] Handle conversation history
- [ ] Complete streaming response integration
- [ ] Add safeguards against unsupported claims

## Phase 6 — Frontend / UX

**Status: 60%**

- [x] Build authentication screens
- [x] Build application shell
- [x] Build repository overview/dashboard
- [x] Add repository search/filtering
- [x] Add indexing status/progress UI
- [x] Add settings and theme controls
- [x] Add chat route and streaming client foundation
- [ ] Complete AI chat interface
- [ ] Display retrieved source references
- [x] Add loading/error/empty states
- [x] Add responsive layouts
- [x] Polish dark/light themes
- [ ] Improve accessibility
- [ ] Add more developer-focused interactions

## Phase 7 — Production

**Status: 0%**

- [ ] Replace development database configuration
- [ ] Introduce controlled Flyway migrations
- [ ] Change Hibernate schema strategy from `update` to `validate`
- [ ] Secure secrets and environment configuration
- [ ] Configure production OAuth credentials
- [ ] Add proper CORS/security configuration
- [ ] Add automated tests
- [ ] Add integration tests
- [ ] Add observability/logging
- [ ] Add health checks
- [ ] Add CI/CD
- [ ] Containerize production services
- [ ] Review database indexes and query performance
- [ ] Review vector-search performance
- [ ] Perform security review
- [ ] Deploy production version

---

# 32. Roadmap Progress Rules

The progress bars represent **actual implemented functionality**, not planned code.

A phase should only be marked complete when its core functionality is implemented and tested.

For example:

```text
Added dependency
      ≠
Feature completed
```

and:

```text
Created database table
      ≠
Authentication implemented
```

This prevents the roadmap from becoming artificially optimistic.

As development progresses, update the percentage and checklist based on actual commits and working functionality.

---

# 33. Project End Goal

The final DevPilot workflow should look like:

```text
Developer
   |
   v
GitHub Login
   |
   v
Select Repository
   |
   v
Repository Sync
   |
   v
Repository Ingestion
   |
   v
Source Code
   |
   v
Chunking + Embeddings
   |
   v
PostgreSQL + pgvector
   |
   v
User Question
   |
   v
Similarity Search
   |
   v
Relevant Repository Context
   |
   v
AI Model
   |
   v
Grounded Developer Answer
   |
   v
Source References
```

The current implementation has completed the **authentication and repository synchronization stages**. The next major milestone is turning retrieved repository files into a searchable RAG knowledge base.
