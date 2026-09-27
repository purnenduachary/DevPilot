# GITHUB REPOSITORY FETCH & SYNC

**Project:** DevPilot  
**Commit:** `6cbf3e68b21c54c9eeb8a4e18818ff72e5d820e6` — `feat: Github repo fetch and sync`

This document explains the **actual code** added for GitHub repository synchronization, focusing on **annotations, fields, methods, and the reason behind each part**.

---

## 1. `RepoController.java`

```java
@RestController
@RequestMapping("/api/repos")
@RequiredArgsConstructor
public class RepoController {
```

### What these annotations do

**`@RestController`**  
Tells Spring this class handles HTTP requests and that returned objects are written directly as response bodies, normally as JSON.

**`@RequestMapping("/api/repos")`**  
Sets the common URL prefix for every endpoint in the controller.

**`@RequiredArgsConstructor`**  
Lombok generates a constructor for all `final` fields, enabling constructor-based dependency injection.

### Dependencies

```java
private final CurrentUser currentUser;
private final RepoService repoService;
private final IndexingService indexingService;
```

- `CurrentUser` → identifies the authenticated user.
- `RepoService` → repository business logic.
- `IndexingService` → starts repository indexing.

### `GET /api/repos`

```java
@GetMapping
public List<RepositoryResponse> list(
    @RequestParam(name = "refresh", defaultValue = "true") boolean refresh)
```

**`@GetMapping`** → handles HTTP GET.

**`@RequestParam`** → reads a query parameter:

```text
/api/repos?refresh=true
```

The flow is:

```text
refresh=true
 → GitHub fetch + database sync

refresh=false
 → database only
```

The authenticated user is obtained with:

```java
UUID userId = currentUser.require().getId();
```

Then:

```java
repoService.syncAndListRepos(userId);
```

or:

```java
repoService.listStored(userId);
```

### `GET /api/repos/{id}`

```java
@GetMapping("/{id}")
public RepositoryResponse get(@PathVariable UUID id)
```

**`@PathVariable`** extracts `{id}` from the URL.

Important part:

```java
repoService.requireOwned(id, userId)
```

This makes the repository lookup **user-scoped** instead of trusting repository ID alone.

### `POST /api/repos/{id}/index`

```java
@PostMapping("/{id}/index")
```

```java
Repository repo = indexingService.startIndexing(id, userId);
indexingService.indexAsync(id, userId);

return ResponseEntity
    .status(HttpStatus.ACCEPTED)
    .body(repoService.toResponse(repo));
```

**`HttpStatus.ACCEPTED` = HTTP 202**

Meaning:

> The request was accepted, but the work is still running.

This is appropriate for asynchronous indexing.

### `GET /api/repos/{id}/status`

Returns a smaller `IndexStatusResponse` used by the frontend while polling.

### Interview

**Why shouldn't business logic live in the controller?**

Because controllers should mainly translate HTTP requests into service calls. Business rules belong in the service layer.

---

# 2. `Repository.java` — JPA ENTITY

```java
@Entity
@Getter
@Setter
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Table(
    name = "repositories",
    uniqueConstraints =
        @UniqueConstraint(columnNames = {"user_id", "github_repo_id"})
)
public class Repository {
```

### Annotations

**`@Entity`**  
Marks the class as a JPA entity mapped to a database table.

**`@Table(name = "repositories")`**  
Explicitly names the database table.

**`@UniqueConstraint(...)`**  
Prevents the same GitHub repository from being duplicated for the same user.

**`@Getter`, `@Setter`**  
Lombok generates getters/setters.

**`@NoArgsConstructor` / `@AllArgsConstructor`**  
Generates constructors.

**`@Builder`**  
Allows builder-style object creation.

### Primary key

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

**`@Id`** → primary key.

**`@GeneratedValue(...UUID)`** → JPA generates the UUID.

### External identity

```java
private Long githubRepoId;
```

You have two identities:

```text
id
→ DevPilot's internal ID

githubRepoId
→ GitHub's repository ID
```

Keeping them separate is important because the internal application should not depend directly on an external provider's identifier.

### Indexing state

```java
@Enumerated(EnumType.STRING)
private IndexStatus indexStatus = IndexStatus.PENDING;
```

**`@Enumerated(EnumType.STRING)`** stores:

```text
PENDING
INDEXING
READY
FAILED
```

instead of ordinal numbers such as `0,1,2,3`.

### `@Builder.Default`

Used on fields such as:

```java
private int chunkCount = 0;
private int filesTotal = 0;
private int filesProcessed = 0;
```

It keeps those defaults when the Lombok builder is used.

### Lifecycle method

```java
@PrePersist
void onCreate() {
```

**`@PrePersist`** runs before a new entity is inserted.

It initializes:

```text
createdAt
updatedAt
indexStatus
```

### Interview

**Why `EnumType.STRING` instead of ordinal?**

Because database values stay readable and do not break if enum ordering changes.

---

# 3. `IndexStatus.java`

```java
public enum IndexStatus {
    PENDING,
    INDEXING,
    READY,
    FAILED
}
```

This is the repository indexing state machine:

```text
PENDING
   ↓
INDEXING
   ↓
READY

INDEXING
   ↓
FAILED
```

The frontend uses these same states to decide what UI to show.

---

# 4. `RepositoryRepository.java`

```java
public interface RepositoryRepository
        extends JpaRepository<Repository, UUID> {
```

### What `JpaRepository` gives you

Spring Data provides common operations such as:

```text
save()
findById()
findAll()
delete()
```

So you do not need to manually write basic CRUD SQL.

### Query derivation

```java
findByUserIdOrderByFullNameAsc(UUID userId)
```

Spring Data reads the method name and builds the query.

Conceptually:

```sql
WHERE user_id = ?
ORDER BY full_name ASC
```

Another important method:

```java
findByIdAndUserId(UUID id, UUID userId)
```

This is what enforces repository ownership at the database-query level.

And:

```java
findByUserIdAndGithubRepoId(UUID userId, Long githubRepoId)
```

is used to detect an existing GitHub repository before saving.

### Interview

**Why is `RepositoryRepository` an interface?**

Spring Data generates the implementation automatically at runtime.

---

# 5. `RepoService.java`

```java
@Service
@RequiredArgsConstructor
public class RepoService {
```

### Annotations

**`@Service`**  
Marks the class as a Spring service/component containing business logic.

**`@RequiredArgsConstructor`**  
Generates constructor injection for its final dependencies.

```java
private final RepositoryRepository repositoryRepository;
private final UserService userService;
private final GithubApiClient gitHubApiClient;
```

---

## `syncAndListRepos()`

```java
@Transactional
public List<RepositoryResponse> syncAndListRepos(UUID userId)
```

**`@Transactional`**  
Defines a database transaction around the operation.

### Step 1 — load user

```java
User user = userService.requiredById(userId);
```

### Step 2 — decrypt token

```java
String token = userService.decryptAccessToken(user);
```

The token is stored encrypted in the database and decrypted only when it is needed for GitHub API access.

### Step 3 — fetch GitHub repositories

```java
List<Map<String, Object>> remoteRepos =
        gitHubApiClient.listUserRepos(token);
```

### Step 4 — process every remote repository

```java
for (Map<String, Object> remote : remoteRepos)
```

GitHub data arrives as maps because the client currently works with JSON-like `Map<String,Object>` structures.

### Step 5 — find existing record

```java
Repository repo =
    repositoryRepository
        .findByUserIdAndGithubRepoId(userId, githubRepoId)
        .orElseGet(Repository::new);
```

This is the **upsert pattern**:

```text
found
 → update it

not found
 → create it
```

### Step 6 — copy GitHub metadata

Examples:

```java
repo.setFullName(fullName);
repo.setPrivate(...);
repo.setDefaultBranch(...);
repo.setLanguage(...);
repo.setHtmlUrl(...);
repo.setDescription(...);
```

The local database becomes a cached application representation of GitHub repository metadata.

### Step 7 — save

```java
saved.add(repositoryRepository.save(repo));
```

### Step 8 — convert entity → DTO

```java
.map(this::toResponse)
```

The database entity is not returned directly.

---

## `listStored()`

```java
@Transactional(readOnly = true)
```

Reads repositories from PostgreSQL without calling GitHub.

**`readOnly = true`** communicates that the transaction is intended only for reads.

---

## `requireOwned()`

```java
repositoryRepository
    .findByIdAndUserId(repoId, userId)
```

This is a critical security-oriented method.

It means:

```text
repository ID matches
AND
authenticated user owns it
```

If not found:

```java
.orElseThrow(
    () -> new NotFoundException("Repository not found")
)
```

---

## `status()`

Creates a small:

```java
IndexStatusResponse
```

containing only indexing information.

---

## `toResponse()`

Maps:

```text
Repository entity
        ↓
RepositoryResponse DTO
```

### Interview

**Why map entity to DTO?**

Because the database model and API contract should remain separate.

---

# 6. `RepositoryResponse.java`

```java
public record RepositoryResponse(
```

A Java `record` is used as a compact immutable data carrier.

It represents the API response instead of exposing the entity.

Important fields include:

```text
id
githubRepoId
fullName
isPrivate
indexStatus
filesTotal
filesProcessed
chunkCount
errorMessage
```

### `@JsonProperty("isPrivate")`

Ensures the serialized JSON property is:

```json
"isPrivate"
```

which matches the frontend model.

### Interview

**Why use a record for DTOs?**

DTOs are mainly data carriers, and records provide a concise way to define immutable data with generated accessors, constructor, `equals`, `hashCode`, and `toString`.

---

# 7. `IndexStatusResponse.java`

A smaller status DTO:

```text
repositoryId
indexStatus
filesTotal
filesProcessed
chunkCount
indexedAt
errorMessage
```

The frontend can poll this without requesting unnecessary repository metadata.

---

# 8. `GithubApiClient.java`

```java
@Service
@RequiredArgsConstructor
public class GithubApiClient
```

This class isolates **all GitHub HTTP communication**.

That means:

```text
RepoService
→ business logic

GithubApiClient
→ GitHub-specific HTTP logic
```

---

## `client()`

```java
private RestClient client(String accessToken)
```

Creates Spring's `RestClient` with common headers:

```text
Authorization: Bearer <token>
Accept: application/vnd.github+json
X-GitHub-Api-Version: 2022-11-28
User-Agent: DevPilot
```

This prevents those headers from being repeated in every request.

---

## `listUserRepos()`

```java
public List<Map<String, Object>> listUserRepos(String accessToken)
```

Calls:

```text
GET /user/repos
```

and paginates:

```text
per_page = 100
page = 1...
```

The loop stops when:

```text
no results
OR
page contains < 100 repos
OR
page reaches 10
```

Therefore the current implementation can retrieve up to:

```text
10 × 100 = 1000 repositories
```

per sync.

---

## `getRepoTree()`

```java
GET /repos/{owner}/{repo}/git/trees/{branch}?recursive=1
```

Returns the repository tree.

This is the future bridge to indexing:

```text
tree
 ↓
file paths
 ↓
file content
 ↓
chunking
 ↓
embeddings
```

---

## `getFileContent()`

Calls:

```text
GET /repos/{owner}/{repo}/contents/{path}
```

GitHub may return file content encoded as Base64.

The code:

```java
Base64.getDecoder().decode(raw)
```

decodes it and then:

```java
new String(..., StandardCharsets.UTF_8)
```

turns the bytes into text.

### Interview

**Why `ParameterizedTypeReference`?**

Because Java's generic type information is erased at runtime, Spring needs explicit type information for generic response bodies such as:

```text
List<Map<String,Object>>
```

**Why separate `GithubApiClient` from `RepoService`?**

This is separation of concerns: provider-specific HTTP logic stays in the client, while repository synchronization rules stay in the service.

---

# 9. `GitHubRateLimiter.java`

```java
@Component
public class GitHubRateLimiter
```

**`@Component`** makes it a Spring-managed bean.

Constructor:

```java
public GitHubRateLimiter(
    @Value("${app.github.api-delay-ms:50}") long delayMs)
```

**`@Value`** injects a configuration property.

Default:

```text
50 ms
```

The value is protected with:

```java
Math.max(0, delayMs)
```

so negative delays are not allowed.

### `pause()`

```java
Thread.sleep(delayMs);
```

adds a configurable delay between operations.

If interrupted:

```java
Thread.currentThread().interrupt();
```

restores the thread's interrupt status.

### Interview

**Why restore the interrupt flag?**

Because catching `InterruptedException` clears the flag. Restoring it preserves the interruption signal for higher-level code.

---

# 10. COMPLETE FLOW

```text
GET /api/repos?refresh=true
            ↓
      RepoController
            ↓
        RepoService
            ↓
       decrypt token
            ↓
      GithubApiClient
            ↓
       GitHub REST API
            ↓
     repository JSON
            ↓
  RepositoryRepository
            ↓
       PostgreSQL
            ↓
    RepositoryResponse
            ↓
         Frontend
```

For indexing:

```text
POST /api/repos/{id}/index
            ↓
   startIndexing()
            ↓
      HTTP 202
            ↓
       indexAsync()
```

---

# 11. KEY INTERVIEW CONCEPTS FROM THIS CODE

**`@RestController`** → REST endpoint class.  
**`@Service`** → business logic bean.  
**`@Repository` / `JpaRepository`** → persistence layer.  
**`@Entity`** → database-mapped class.  
**DTO / record** → API data contract.  
**`@Transactional`** → transaction boundary.  
**`@Transactional(readOnly = true)`** → read-oriented transaction.  
**`@PathVariable`** → value from URL path.  
**`@RequestParam`** → value from query string.  
**`@Value`** → inject configuration.  
**`@PrePersist`** → run before entity insertion.  
**`@Enumerated(EnumType.STRING)`** → store enum names.  
**Upsert** → update existing row or insert a new one.  
**Constructor injection** → dependencies supplied through constructor.  
**HTTP 202** → request accepted while work continues asynchronously.

---

# 12. THE ARCHITECTURE IN ONE VIEW

```text
Controller
    │
    ▼
Service
    │
    ├── UserService
    ├── GithubApiClient
    └── RepositoryRepository
              │
              ▼
          PostgreSQL

GithubApiClient
      │
      ▼
 GitHub REST API
```

### The one-minute explanation

> “I implemented GitHub repository synchronization using a layered Spring Boot design. `RepoController` handles the HTTP endpoints, `RepoService` contains the synchronization and ownership logic, `GithubApiClient` isolates GitHub REST communication, and `RepositoryRepository` handles persistence through Spring Data JPA. GitHub repositories are upserted using the authenticated user ID plus GitHub repository ID, then exposed through DTOs. I also added explicit indexing states, an asynchronous indexing endpoint returning 202, recursive Git tree retrieval for future indexing, Base64 file-content decoding, and configurable request pacing.”

**Main thing to remember:**  
`Controller → Service → Repository/External Client` is the core flow; annotations mainly tell Spring **what each class is, how it should be managed, and where it fits in the application.**
