# DEV PILOT — GEMINI + RAG INDEXING BACKEND

## SOURCE / WORK SESSION

This document is based on the `purnenduachary/DevPilot` repository and the latest repository change:

- Commit: `aebae04621cd446319edeec4bbbf8489aca9af34`
- Commit message: `feat: integrate Gemini AI and repository indexing`
- Commit date: 30 September 2026
- Scope: Gemini model integration, PGVector setup, repository indexing pipeline, indexing APIs, code filtering/chunking, RAG retrieval/generation support, citations, progress tracking, and rate limiting.

The attached Excalidraw flow was also used as the explanation reference for the indexing pipeline.

> **Diagram consistency note:** the attached diagram labels the embedding step as an **OpenAI Embedding API**, while the repository code/configuration in this commit uses **Google Gemini** for embeddings. The explanation below follows the repository implementation. Update the diagram label before using it as the final architecture reference.

---

# 1. WHAT WE BUILT TODAY

The main goal was to make a GitHub repository usable as an AI knowledge base.

Previously:

```text
GitHub Repository
       ↓
Fetch repository metadata/files
```

Today's implementation extends this toward:

```text
GitHub Repository
       ↓
Get repository tree
       ↓
Filter useful files
       ↓
Read file contents
       ↓
Split files into chunks
       ↓
Generate embeddings
       ↓
Store vectors in PostgreSQL + PGVector
       ↓
Track indexing progress
       ↓
READY
```

This creates the **Index** side of the RAG pipeline.

The same commit also adds **Retrieve + Generate** support:

```text
User Question
      ↓
Similarity Search in PGVector
      ↓
Relevant Code Chunks
      ↓
Prompt with Code Context
      ↓
Gemini
      ↓
Streamed Answer + Citations
```

---

# 2. THE SIMPLE INTERVIEW EXPLANATION

### What is indexing?

> **Indexing means preparing the codebase so the AI can search it efficiently.**

In DevPilot:

> **We fetch the repository, keep only useful files, split the files into smaller chunks, generate embeddings for those chunks, and store the vectors in PGVector. Later, a user question can be matched against those vectors to retrieve the most relevant code.**

### What is an embedding?

> **An embedding converts text or code into a numerical vector that represents its meaning, which allows semantic similarity search.**

### Why do we need chunks?

Because sending an entire repository to the AI for every question would be inefficient and can exceed the model's context capacity.

Instead:

```text
Large File
   ↓
Small Code Chunks
   ↓
Embeddings
   ↓
Vector Search
```

---

# 3. RAG OVERVIEW

RAG = **Retrieval-Augmented Generation**

DevPilot follows this conceptual flow:

```text
INDEX
  Repository
      ↓
  Filter files
      ↓
  Chunk code
      ↓
  Embed chunks
      ↓
  Store vectors

RETRIEVE
  User question
      ↓
  Similarity search
      ↓
  Top relevant chunks

GENERATE
  Retrieved code + question
      ↓
  Gemini
      ↓
  Answer
```

The distinction to remember:

- **Indexing** prepares the knowledge.
- **Retrieval** finds the relevant knowledge.
- **Generation** uses the retrieved knowledge to produce the answer.

---

# 4. SPRING AI + GEMINI + PGVECTOR

## 4.1 Maven dependencies

`backend/pom.xml` was changed to include PGVector and Gemini.

```xml
<!-- Vector-store integration for PostgreSQL + PGVector. -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-vector-store-pgvector</artifactId>
</dependency>

<!-- Gemini chat model integration. -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-google-genai</artifactId>
</dependency>

<!-- Gemini embedding model integration. -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-starter-model-google-genai-embedding</artifactId>
</dependency>
```

### Why each dependency exists

**PGVector**

```text
Spring AI → VectorStore → PostgreSQL/PGVector
```

It gives the application a vector-store abstraction backed by PostgreSQL/PGVector.

**Google GenAI**

Provides the chat-model integration used for answer generation.

**Google GenAI Embedding**

Provides the embedding-model integration used when indexing code.

---

# 5. APPLICATION CONFIGURATION

`backend/src/main/resources/application.properties`

Relevant configuration in the current repository:

```properties
# Read the Gemini API key from the environment.
# The real key should never be committed to Git.
spring.ai.google.genai.api-key=${GOOGLE_API_KEY}

# Chat model used for answer generation.
spring.ai.google.genai.chat.model=gemini-2.5-flash

# Embedding model used during indexing.
spring.ai.google.genai.embedding.text.model=gemini-embedding-001

# Ask Spring AI to initialize the PGVector schema.
spring.ai.vectorstore.pgvector.initialize-schema=true

# Vector dimension configured for the PGVector store.
spring.ai.vectorstore.pgvector.dimensions=1536

# Configured vector index type.
spring.ai.vectorstore.pgvector.index-type=HNSW

# Configured similarity distance.
spring.ai.vectorstore.pgvector.distance-type=COSINE_DISTANCE
```

### Interview explanation

> **Gemini is used for the model/embedding side, while PostgreSQL with PGVector stores and searches the vector representations.**

---

# 6. API LAYER — START INDEXING

## `RepoController.java`

The repository indexing endpoint is now active.

```java
@PostMapping("/{id}/index")
public ResponseEntity<RepositoryResponse> index(@PathVariable UUID id) {

    // Read the authenticated user's ID.
    // Repository operations are scoped to that user.
    UUID userId = currentUser.require().getId();

    // Fast part: validate ownership, reset progress and mark INDEXING.
    Repository repo = indexingService.startIndexing(id, userId);

    // Slow part: start the repository processing in the background.
    indexingService.indexAsync(id, userId);

    // 202 = accepted for asynchronous processing; work is not finished yet.
    return ResponseEntity.status(HttpStatus.ACCEPTED)
            .body(repoService.toResponse(repo));
}
```

### Why `202 ACCEPTED`?

The server has accepted the request, but indexing is still running.

That is appropriate for a job that can involve many GitHub calls and vector operations.

---

# 7. API LAYER — CHECK INDEXING STATUS

```java
@GetMapping("/{id}/status")
public IndexStatusResponse status(@PathVariable UUID id) {

    // Keep the status request scoped to the authenticated user.
    UUID userId = currentUser.require().getId();

    // Return lightweight progress information to the frontend.
    return repoService.status(id, userId);
}
```

The frontend can poll:

```text
GET /api/repos/{id}/status
```

The status lifecycle is:

```text
PENDING
   ↓
INDEXING
   ↓
READY
```

or:

```text
INDEXING
   ↓
FAILED
```

---

# 8. INDEXINGSERVICE — DEEP DIVE

File:

```text
backend/src/main/java/devPilot/backend/services/indexing/IndexingService.java
```

This is the **orchestrator** of the indexing pipeline.

It does not implement every operation itself. Instead, it coordinates the classes responsible for:

```text
Repository DB
     ↓
GitHub authentication
     ↓
GitHub API
     ↓
File filtering
     ↓
Chunking
     ↓
VectorStore
     ↓
Progress tracking
```

The easiest way to remember its responsibility is:

> **Validate → start → fetch → filter → read → chunk → store → track → finish.**

---

## 8.1 CLASS DECLARATION

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class IndexingService {
```

### `@Service`

Registers the class as a Spring-managed service bean.

### `@RequiredArgsConstructor`

Lombok generates a constructor for all `final` dependencies, giving us constructor-based dependency injection.

### `@Slf4j`

Lombok creates the `log` object used for `info`, `warn`, and `error` logging.

---

## 8.2 DEPENDENCIES — WHY EACH ONE IS HERE

```java
private final RepositoryRepository repositoryRepository;
private final UserService userService;
private final GithubApiClient gitHubApiClient;
private final CodeFileFilter fileFilter;
private final CodeChunker codeChunker;
private final GitHubRateLimiter rateLimiter;
private final VectorStore vectorStore;
```

Think of these as seven tools given to the service.

| Dependency | What `IndexingService` uses it for |
|---|---|
| `RepositoryRepository` | Read and update repository/indexing state in PostgreSQL |
| `UserService` | Load the user and decrypt the stored GitHub access token |
| `GithubApiClient` | Get repository tree and file contents from GitHub |
| `CodeFileFilter` | Decide whether a file should be indexed |
| `CodeChunker` | Split a source file into smaller `Document` chunks |
| `GitHubRateLimiter` | Add a small delay between GitHub requests |
| `VectorStore` | Pass chunks through the configured Spring AI vector-store pipeline |

### Interview wording

> **`IndexingService` is the coordinator. Specialized classes perform GitHub access, filtering, chunking and storage, while this service controls the order and lifecycle of the whole job.**

---

## 8.3 CONSTANTS

```java
private static final int VECTOR_BATCH_SIZE = 32;
private static final int PROGRESS_EVERY_N_FILES = 5;
```

### `VECTOR_BATCH_SIZE`

We do not write every chunk individually.

```text
chunk → store
chunk → store
chunk → store
```

would create many store operations.

Instead:

```text
chunks
  ↓
collect 32
  ↓
vectorStore.add(batch)
```

### `PROGRESS_EVERY_N_FILES`

The database does not need to be updated after every file.

The current implementation updates progress every five files, and also on the last file.

---

## 8.4 MAX FILE SIZE CONFIGURATION

```java
@Value("${app.indexing.max-file-bytes:102400}")
private long maxFileBytes;
```

This reads the configuration property:

```properties
app.indexing.max-file-bytes=102400
```

The current configured value is **100 KB**.

The point is to stop very large files from entering the indexing pipeline.

---

# 9. `startIndexing()` — THE FAST PHASE

```java
public Repository startIndexing(UUID repoId, UUID userId) {
```

This method is intentionally small and fast.

Its job is to **validate the request and prepare the repository state**.

---

## 9.1 STEP 1 — CHECK OWNERSHIP

```java
Repository repo = repositoryRepository.findByIdAndUserId(repoId, userId)
        .orElseThrow(() -> new NotFoundException("Repository not found"));
```

This is more important than it first looks.

We do not simply call:

```java
findById(repoId)
```

We check:

```text
repository ID
      +
authenticated user ID
```

So the lookup effectively says:

> **Find this repository only if it belongs to this logged-in user.**

That prevents cross-user repository access through the indexing endpoint.

---

## 9.2 STEP 2 — STOP DUPLICATE INDEXING

```java
if (repo.getIndexStatus() == IndexStatus.INDEXING) {
    throw new BadRequestException("Repository is already being indexed");
}
```

Without this check, repeated button clicks could start multiple background jobs for the same repository.

The state machine is therefore:

```text
PENDING
   ↓
INDEXING
   ↓
READY
```

and a running job cannot be started again while it is already `INDEXING`.

---

## 9.3 STEP 3 — RESET THE RUN

```java
repo.setIndexStatus(IndexStatus.INDEXING);
repo.setFilesProcessed(0);
repo.setFilesTotal(0);
repo.setChunkCount(0);
repo.setErrorMessage(null);
repo.setUpdatedAt(Instant.now());
```

This resets the state from the previous run.

Example:

```text
Previous run:
READY
50 files
240 chunks
```

becomes:

```text
New run:
INDEXING
0 processed
0 total
0 chunks
no error
```

---

## 9.4 STEP 4 — SAVE IMMEDIATELY

```java
return repositoryRepository.save(repo);
```

This is important because the frontend can now immediately observe:

```text
INDEXING
```

It does **not** have to wait for the complete repository to finish.

---

# 10. `indexAsync()` — MOVE THE LONG JOB OFF THE REQUEST THREAD

```java
@Async("indexingExecutor")
public void indexAsync(UUID repoId, UUID userId) {
    try {
        doIndex(repoId, userId);
    } catch (Exception ex) {
        log.error("Indexing failed for repo {}", repoId, ex);
        markFailed(repoId, ex.getMessage());
    }
}
```

This method is the bridge between the HTTP request and the long-running indexing process.

### What happens?

```text
POST /api/repos/{id}/index
          ↓
startIndexing()
          ↓
202 ACCEPTED
          ↓
indexAsync()
          ↓
background executor
          ↓
doIndex()
```

### Why asynchronous?

Indexing can involve many:

```text
GitHub requests
file-processing operations
embedding/vector operations
database updates
```

A normal HTTP request should not sit open until all of that is finished.

### Interview wording

> **We use asynchronous execution so the API can acknowledge the indexing request immediately while the expensive repository processing continues in the background.**

---

# 11. `doIndex()` — THE ACTUAL ENGINE

```java
private void doIndex(UUID repoId, UUID userId) {
```

This method performs the complete pipeline.

Think of it as:

```text
1. Get repository
2. Get GitHub token
3. Delete previous vectors
4. Get repository tree
5. Filter files
6. Process every eligible file
7. Flush remaining chunks
8. Mark READY
```

---

# 12. STEP 1 — LOAD THE REPOSITORY AGAIN

```java
Repository repo = repositoryRepository.findById(repoId)
        .orElseThrow(() -> new NotFoundException("Repository not found"));
```

Why retrieve it again when `startIndexing()` already had it?

Because `indexAsync()` is running as a separate background operation. It should obtain the current repository state again before doing the expensive work.

The service now has access to:

```text
repo.getOwner()
repo.getName()
repo.getDefaultBranch()
repo.getFullName()
```

which are required for GitHub operations and logging.

---

# 13. STEP 2 — GET AND DECRYPT THE GITHUB TOKEN

```java
String token = userService.decryptAccessToken(
        userService.requiredById(userId));
```

Read this from inside-out:

```text
requiredById(userId)
        ↓
load User
        ↓
decryptAccessToken(...)
        ↓
plain GitHub access token
```

The reason is simple: DevPilot must call GitHub **as the authenticated user** so it can access the repositories that user is allowed to access.

The token is used only inside the service flow; it should not be persisted in plaintext in this method.

---

# 14. STEP 3 — DELETE OLD VECTORS

```java
deleteExistingVectors(repoId.toString());
```

Today's implementation uses a **full re-index** strategy.

That means:

```text
Old vectors for repository
          ↓
Delete
          ↓
Build a fresh index from current GitHub source
```

### Why?

Suppose yesterday the repository contained:

```text
AuthService.java   → old code
```

and today it contains:

```text
AuthService.java   → new code
```

If the old vectors remain, retrieval can see both old and new content.

So the current design rebuilds the repository index from scratch.

---

## 14.1 `deleteExistingVectors()`

```java
private void deleteExistingVectors(String repoId) {
    try {
        var filter = new FilterExpressionBuilder()
                .eq(RagSettings.METADATA_REPO_ID, repoId)
                .build();

        vectorStore.delete(filter);

    } catch (Exception ex) {
        log.warn(
                "Could not delete existing vectors for repo {}: {}",
                repoId,
                ex.getMessage());
    }
}
```

### Important line

```java
.eq(RagSettings.METADATA_REPO_ID, repoId)
```

This means:

> **Delete vectors belonging to this repository, not vectors for every repository.**

That same metadata key is later used during retrieval.

### Important implementation behavior

The current method logs a deletion failure and continues rather than throwing immediately.

That is a deliberate behavior of the current source and is worth knowing when discussing failure handling.

---

# 15. STEP 4 — FETCH THE GITHUB REPOSITORY TREE

```java
Map<String, Object> tree = gitHubApiClient.getRepoTree(
        token,
        repo.getOwner(),
        repo.getName(),
        repo.getDefaultBranch());
```

The GitHub client calls the recursive tree endpoint:

```text
/repos/{owner}/{repo}/git/trees/{branch}?recursive=1
```

The result gives us the repository structure.

Conceptually:

```text
DevPilot/
├── backend/
│   ├── pom.xml
│   ├── UserService.java
│   └── ...
├── client/
│   ├── page.tsx
│   └── ...
├── node_modules/
└── .git/
```

At this point we have **paths and metadata**, not yet all file contents.

That lets us filter before downloading every file.

---

# 16. STEP 5 — BUILD THE LIST OF INDEXABLE FILES

```java
List<String> filePaths = listIndexableFiles(tree);
```

This helper turns the raw GitHub tree into a clean list of files the RAG pipeline actually wants.

### Exact logic

```java
@SuppressWarnings("unchecked")
private List<String> listIndexableFiles(Map<String, Object> tree) {

    // No tree means there is nothing to process.
    if (tree == null || tree.get("tree") == null) {
        return List.of();
    }

    // GitHub returns tree entries as maps.
    List<Map<String, Object>> entries =
            (List<Map<String, Object>>) tree.get("tree");

    return entries.stream()
            // Keep actual files. GitHub identifies files as "blob" entries.
            .filter(entry ->
                    "blob".equals(String.valueOf(entry.get("type"))))

            // Let CodeFileFilter decide whether the file is worth indexing.
            .filter(entry -> {
                String path = String.valueOf(entry.get("path"));

                long size = entry.get("size") instanceof Number n
                        ? n.longValue()
                        : 0L;

                return fileFilter.isEligible(
                        path,
                        size,
                        maxFileBytes);
            })

            // Extract just the repository path.
            .map(entry -> String.valueOf(entry.get("path")))
            .toList();
}
```

### `blob` vs `tree`

In Git's tree representation:

```text
blob = file
 tree = directory
```

So the first filter removes directories and keeps file entries.

---

# 17. STEP 6 — INITIALIZE PROGRESS

```java
updateProgress(
        repoId,
        filePaths.size(),
        0,
        0,
        IndexStatus.INDEXING,
        null);
```

Suppose there are 27 eligible files.

Database becomes approximately:

```text
filesTotal     = 27
filesProcessed = 0
chunkCount     = 0
indexStatus    = INDEXING
```

This gives the frontend the information needed to show real progress.

---

# 18. STEP 7 — CREATE THE BATCH AND COUNTERS

```java
List<Document> batch = new ArrayList<>();
int processed = 0;
int totalChunks = 0;
```

### `batch`

Temporary collection of chunks waiting to be passed to the vector store.

### `processed`

Number of files that have been handled by the loop.

### `totalChunks`

Total number of chunks generated so far.

Example:

```text
AuthService.java      → 3 chunks
SecurityConfig.java   → 2 chunks
UserService.java      → 4 chunks
```

then:

```text
processed   = 3
chunks      = 9
```

---

# 19. STEP 8 — THE MAIN `FOR EACH FILE` LOOP

```java
for (String path : filePaths) {
```

This is where the repository is actually transformed into searchable AI knowledge.

The loop performs four main operations per file:

```text
DOWNLOAD
   ↓
CHUNK
   ↓
BATCH / STORE
   ↓
PROGRESS
```

---

## 19.1 DOWNLOAD FILE CONTENT

```java
String content = gitHubApiClient.getFileContent(
        token,
        repo.getOwner(),
        repo.getName(),
        path);
```

Example:

```text
path = backend/.../AuthService.java
```

GitHub returns the source code as a string.

Now `IndexingService` has the actual content required for chunking.

---

## 19.2 CHUNK THE FILE

```java
List<Document> chunks = codeChunker.chunkFile(
        repoId.toString(),
        path,
        content);
```

`IndexingService` delegates the chunking logic to `CodeChunker`.

The returned objects are Spring AI `Document`s.

Conceptually one file becomes:

```text
AuthService.java
      ↓
Document / Chunk 0
Document / Chunk 1
Document / Chunk 2
...
```

Each chunk carries both:

```text
text/code
+
metadata
```

such as:

```text
repoId
filePath
language
chunkIndex
```

---

# 20. STEP 9 — ADD CHUNKS TO THE BATCH

```java
batch.addAll(chunks);
totalChunks += chunks.size();
```

Suppose the current file produced 4 chunks.

Then:

```text
batch += 4 documents
totalChunks += 4
```

This keeps the service aware of how much searchable content has been produced.

---

# 21. STEP 10 — SEND BATCH TO THE VECTOR STORE

```java
if (batch.size() >= VECTOR_BATCH_SIZE) {
    vectorStore.add(batch);
    batch.clear();
}
```

With:

```java
VECTOR_BATCH_SIZE = 32;
```

the service waits until at least 32 documents are collected.

Then:

```text
32 Documents
      ↓
vectorStore.add(batch)
```

The configured Spring AI vector-store integration processes the documents for vector storage, including the embedding/storage pipeline used by the PGVector setup.

### Why does `IndexingService` not call an embedding API directly?

Because that responsibility is abstracted behind Spring AI's `VectorStore` layer.

The service therefore focuses on orchestration instead of provider-specific embedding HTTP calls.

### Interview wording

> **We keep the indexing service provider-agnostic by handing documents to Spring AI's vector-store abstraction instead of hard-coding an embedding API call inside the orchestration layer.**

---

# 22. STEP 11 — SINGLE-FILE FAILURE HANDLING

The per-file code is wrapped in:

```java
try {
    // download
    // chunk
    // batch
} catch (Exception ex) {
    log.warn(
            "Skipping file {} in {}: {}",
            path,
            repo.getFullName(),
            ex.getMessage());
}
```

This is an important reliability decision.

Suppose:

```text
A.java  ✅
B.java  ✅
C.java  ❌
D.java  ✅
```

The desired behavior is:

```text
C.java fails
    ↓
log warning
    ↓
skip C.java
    ↓
continue with D.java
```

A single unreadable/broken/problematic file should not automatically destroy an otherwise valid repository indexing run.

---

# 23. STEP 12 — UPDATE THE FILE COUNTER

After each loop iteration:

```java
processed++;
```

This tracks how many file entries have been handled by the loop.

One subtle point:

> `processed` is a processed/attempted file count; it is not a separate count of only successfully embedded files.

That distinction matters when one file fails.

---

# 24. STEP 13 — PERIODIC PROGRESS UPDATE

```java
if (processed % PROGRESS_EVERY_N_FILES == 0
        || processed == filePaths.size()) {

    updateProgress(
            repoId,
            filePaths.size(),
            processed,
            totalChunks,
            IndexStatus.INDEXING,
            null);
}
```

With:

```java
PROGRESS_EVERY_N_FILES = 5;
```

updates happen around:

```text
5 files
10 files
15 files
20 files
...
```

and always on the final file.

Example database state:

```text
filesTotal     = 20
filesProcessed = 10
chunkCount     = 36
status         = INDEXING
```

The frontend polls the status endpoint and can display that progress.

---

# 25. STEP 14 — RATE LIMIT BETWEEN FILE OPERATIONS

```java
rateLimiter.pause();
```

The configured value is:

```properties
app.github.api-delay-ms=50
```

So the pipeline introduces a small pause between file-processing iterations.

The goal is to reduce request pressure while making many GitHub API calls.

---

# 26. STEP 15 — FLUSH THE FINAL PARTIAL BATCH

After the loop:

```java
if (!batch.isEmpty()) {
    vectorStore.add(batch);
}
```

This is easy to miss but essential.

Suppose the batch contains only:

```text
18 chunks
```

It never reached 32, so the normal batch condition did not execute.

Without this final flush:

```text
18 chunks
   ↓
remain only in memory
   ↓
never stored
```

So the final block writes the leftover chunks.

---

# 27. STEP 16 — MARK THE REPOSITORY READY

```java
markReady(
        repoId,
        filePaths.size(),
        processed,
        totalChunks,
        repo.getFullName());
```

The repository is now considered successfully indexed.

`markReady()` stores:

```java
repo.setIndexStatus(IndexStatus.READY);
repo.setFilesTotal(totalFiles);
repo.setFilesProcessed(processedFiles);
repo.setChunkCount(totalChunks);
repo.setIndexedAt(Instant.now());
repo.setErrorMessage(null);
repo.setUpdatedAt(Instant.now());
```

The lifecycle becomes:

```text
INDEXING
   ↓
READY
```

The timestamp in `indexedAt` records when the successful indexing completed.

---

# 28. WHOLE-JOB FAILURE

The outer async method catches failures that escape the per-file handling:

```java
catch (Exception ex) {
    log.error("Indexing failed for repo {}", repoId, ex);
    markFailed(repoId, ex.getMessage());
}
```

Then:

```java
protected void markFailed(UUID repoId, String message) {
    repositoryRepository.findById(repoId).ifPresent(repo -> {
        repo.setIndexStatus(IndexStatus.FAILED);
        repo.setErrorMessage(...);
        repo.setUpdatedAt(Instant.now());
        repositoryRepository.save(repo);
    });
}
```

So the two failure strategies are different:

```text
ONE FILE FAILS
     ↓
log + skip + continue
```

versus:

```text
WHOLE JOB FAILS
     ↓
status = FAILED
     ↓
store error message
```

That is a useful resilience pattern to explain in an interview.

---

# 29. `IndexingService` — METHOD-BY-METHOD MENTAL MODEL

```text
startIndexing()
    ↓
Validate ownership
Reject duplicate run
Set INDEXING
Reset counters

indexAsync()
    ↓
Move expensive work to background

 doIndex()
    ↓
Load repository
Get decrypted GitHub token
Delete old vectors
Get GitHub tree
Filter files

FOR EACH FILE
    ↓
Get content
    ↓
Chunk content
    ↓
Add chunks to batch
    ↓
Write full batches to VectorStore
    ↓
Update progress
    ↓
Pause for GitHub rate limiting

After loop
    ↓
Flush remaining batch
    ↓
markReady()

If whole job fails
    ↓
markFailed()
```

---

# 30. WHAT `INDEXINGSERVICE` DOES **NOT** DO ITSELF

This distinction is important for interviews.

### It does not implement GitHub HTTP details

That belongs to:

```text
GithubApiClient
```

### It does not implement file-selection rules

That belongs to:

```text
CodeFileFilter
```

### It does not implement the chunking algorithm

That belongs to:

```text
CodeChunker
```

### It does not implement vector similarity search

That belongs to:

```text
CodeContextRetriever
```

### It orchestrates all of them.

That is why the service is the **pipeline coordinator**.

---

# 31. WHY THE SERVICE IS DESIGNED THIS WAY

The design follows separation of responsibilities:

```text
IndexingService
      │
      ├── GitHub access → GithubApiClient
      ├── Filtering     → CodeFileFilter
      ├── Chunking      → CodeChunker
      ├── Rate limiting → GitHubRateLimiter
      └── Vector store  → VectorStore
```

This makes the indexing service easier to understand and change.

For example, if file-filtering rules change, we can modify `CodeFileFilter` without rewriting the indexing orchestration.

---

# 32. THE MOST IMPORTANT LINE OF THOUGHT

When reading `IndexingService`, ask four questions:

### Where does the source come from?

```text
GitHub
```

### What source do we keep?

```text
CodeFileFilter
```

### How do we prepare it for semantic search?

```text
CodeChunker
→ VectorStore / embedding pipeline
```

### How do we know the job finished?

```text
Repository indexStatus
```

That is the complete mental model.

---

# 33. INTERVIEW ANSWER — `INDEXINGSERVICE`

> **"`IndexingService` is the orchestration layer for repository indexing. It first validates repository ownership, prevents duplicate indexing and marks the repository as `INDEXING`. The heavy work then runs asynchronously: it decrypts the user's GitHub token, removes old vectors, fetches the repository tree, filters eligible files, downloads each file, chunks it using `CodeChunker`, and passes the chunks to the Spring AI `VectorStore` in batches. It periodically updates file and chunk progress, applies GitHub request throttling, flushes the final partial batch, and finally marks the repository `READY`. Individual file failures are skipped and logged, while a failure of the overall job marks the repository `FAILED`."**

---

# 34. 15-SECOND VERSION

> **"The `IndexingService` orchestrates the full repository-to-RAG pipeline: fetch files from GitHub, filter them, chunk them, store their vectors in PGVector, track progress, and mark the repository ready."**

---

# 22. RETRIEVAL — THE R IN RAG

File:

```text
backend/src/main/java/devPilot/backend/services/ai/CodeContextRetriever.java
```

When the user asks a question, DevPilot searches the vector store.

```java
public RetrievedContext retrieve(
        UUID repositoryId,
        String question) {

    // Restrict the search to the requested repository.
    var filter = new FilterExpressionBuilder()
            .eq(
                RagSettings.METADATA_REPO_ID,
                repositoryId.toString())
            .build();

    var search = SearchRequest.builder()

            // The user's natural-language question becomes the search query.
            .query(question)

            // Current implementation asks for the top 8 matching chunks.
            .topK(RagSettings.TOP_K_CHUNKS)

            // Prevent chunks from other repositories being returned.
            .filterExpression(filter)

            .build();

    var documents = vectorStore.similaritySearch(search);

    ...
}
```

Current setting:

```java
public static final int TOP_K_CHUNKS = 8;
```

### Interview explanation

> **We perform semantic similarity search against the indexed vectors and retrieve the top-K relevant code chunks for the requested repository.**

---

# 23. BUILDING THE RETRIEVED CONTEXT

After retrieval, the text from the selected documents is combined:

```java
var contextText = documents.stream()
        .map(Document::getText)
        .collect(Collectors.joining("\n\n---\n\n"));
```

Conceptually:

```text
Code context:

[AuthService.java chunk]

---

[SecurityConfig.java chunk]

---

[GithubOAuthHandler.java chunk]
```

That combined context is then sent to the chat layer.

---

# 24. PROMPT BUILDING

File:

```text
backend/src/main/java/devPilot/backend/services/ai/ChatPromptBuilder.java
```

The class separates system instructions from user input.

```java
public String systemPrompt(String repositoryFullName) {

    return """
            You are DevPilot, an expert assistant for the %s codebase.
            Answer using ONLY the provided code context.
            If the context is insufficient, say you are unsure.
            Cite file paths and line ranges when relevant.
            Be concise and technical.
            """.formatted(repositoryFullName);
}
```

The system prompt establishes the assistant's operating rules.

The user prompt contains retrieved code and the question:

```java
public String userPrompt(
        String codeContext,
        String question) {

    return """
            Code context:
            %s

            User question:
            %s
            """.formatted(codeContext, question);
}
```

The conceptual input becomes:

```text
System rules
     +
Retrieved repository context
     +
User question
     ↓
Configured Spring AI chat model
```

---

# 25. STREAMING THE AI RESPONSE

File:

```text
backend/src/main/java/devPilot/backend/services/ai/ChatStreamHandler.java
```

The current generation layer uses `ChatClient` and `SseEmitter`.

```java
ChatClient.builder(chatModel)
        .build()
        .prompt()
        .system(systemPrompt)
        .user(userPrompt)
        .stream()
        .content()
        .doOnNext(token -> appendToken(emitter, fullReply, token))
        .doOnError(err -> {
            log.error("Chat stream error", err);
            emitter.completeWithError(err);
        })
        .doOnComplete(() ->
                completeStream(emitter, sessionId, fullReply, citations))
        .subscribe();
```

### Why stream?

Without streaming:

```text
User
 ↓
wait...
 ↓
whole answer
```

With streaming:

```text
Gemini
 ↓
token
 ↓
token
 ↓
token
 ↓
token
 ↓
UI updates live
```

The browser can therefore start rendering the response before generation is completely finished.

---

# 26. CITATIONS

File:

```text
backend/src/main/java/devPilot/backend/services/ai/CitationMapper.java
```

Retrieved `Document` metadata is mapped into an API citation object:

```java
return new CitationDto(
        stringVal(meta.get("filePath")),
        intVal(meta.get("startLine")),
        intVal(meta.get("endLine")),
        stringVal(meta.get("language")));
```

### Why citations?

The assistant should be able to connect an answer back to its supporting source.

Conceptually:

```text
Answer
  ↓
Source path
  ↓
Relevant line information
```

That makes the AI response more traceable for a developer.

---

# 27. SAVING THE FINAL AI MESSAGE

Once the streamed response is complete:

```java
ChatMessage assistant = chatMessageRepository.save(
        ChatMessage.builder()
                // Keep the message attached to its chat session.
                .sessionId(sessionId)

                // Mark this database record as an assistant message.
                .role(MessageRole.ASSISTANT)

                // Save the complete response assembled from streamed tokens.
                .content(fullReply.toString())

                // Persist citation information for later display.
                .citations(citationMapper.toJson(citations))

                .build());
```

Then the server sends:

```text
assistant_message
done
```

and completes the SSE emitter.

---

# 28. CENTRAL RAG SETTINGS

File:

```text
backend/src/main/java/devPilot/backend/services/ai/RagSettings.java
```

```java
public final class RagSettings {

    // Number of code chunks retrieved for one question.
    public static final int TOP_K_CHUNKS = 8;

    // Maximum time allowed for an SSE chat stream.
    public static final long STREAM_TIMEOUT_MS = 180_000L;

    // Metadata key used to isolate repository vectors.
    public static final String METADATA_REPO_ID = "repoId";

    private RagSettings() {
        // Utility class: prevent normal instantiation.
    }
}
```

### Why constants?

Instead of scattering magic numbers and strings throughout the code, the pipeline keeps them in one place.

That improves readability and makes tuning easier.

---

# 29. RETRIEVEDCONTEXT

File:

```text
backend/src/main/java/devPilot/backend/services/ai/RetrievedContext.java
```

```java
public record RetrievedContext(
        List<CitationDto> citations,
        String contextText) {
}
```

A Java `record` is used as a compact immutable data carrier.

It groups two related outputs:

```text
retrieved code context
+
citations
```

---

# 30. EXISTING GITHUB CLIENT USED BY THE NEW PIPELINE

The indexing service depends on `GithubApiClient`, which already existed before this indexing commit.

Relevant responsibilities:

### `listUserRepos(...)`

Fetches repositories for the authenticated GitHub user.

### `getRepoTree(...)`

Fetches the recursive tree needed to discover repository file paths.

### `getFileContent(...)`

Fetches a file from GitHub and decodes Base64 content when GitHub returns it encoded.

The indexing flow therefore reuses the existing GitHub integration rather than duplicating API communication inside `IndexingService`.

---

# 31. COMPLETE INDEXING FLOW

```text
User clicks "Index"
       ↓
POST /api/repos/{id}/index
       ↓
RepoController
       ↓
startIndexing(...)
       ↓
Validate repository ownership
       ↓
Set status = INDEXING
       ↓
indexAsync(...)
       ↓
Decrypt GitHub token
       ↓
Delete old repository vectors
       ↓
Get recursive GitHub repository tree
       ↓
Filter eligible files
       ↓
FOR EACH FILE
   ↓
   Get file content from GitHub
   ↓
   Create Document
   ↓
   Split into chunks
   ↓
   Add chunks to batch
   ↓
   Every 32 chunks → VectorStore.add(...)
   ↓
   Update progress every 5 files
   ↓
   Pause between GitHub requests
       ↓
Flush final batch
       ↓
Status = READY
```

---

# 32. COMPLETE RAG QUERY FLOW

```text
User asks a question
       ↓
CodeContextRetriever
       ↓
VectorStore similaritySearch
       ↓
Top 8 relevant chunks
       ↓
CitationMapper
       ↓
ChatPromptBuilder
       ↓
Configured Spring AI chat model / Gemini
       ↓
SSE token stream
       ↓
Save complete ChatMessage
       ↓
Return citations
       ↓
done
```

---

# 33. FILES CHANGED IN TODAY'S COMMIT

| File | Purpose |
|---|---|
| `backend/pom.xml` | Adds PGVector and Gemini AI/embedding dependencies |
| `RepoController.java` | Enables the repository indexing endpoint |
| `ChatPromptBuilder.java` | Builds system and user prompts |
| `ChatStreamHandler.java` | Streams generated responses through SSE and persists the final message |
| `CitationMapper.java` | Maps document metadata to citation data and JSON |
| `CodeContextRetriever.java` | Performs repository-scoped semantic retrieval |
| `RagSettings.java` | Stores shared RAG constants |
| `RetrievedContext.java` | Carries retrieved context and citations together |
| `CodeChunker.java` | Splits repository files into searchable chunks |
| `CodeFileFilter.java` | Filters out irrelevant/unsafe-to-index file paths |
| `IndexingService.java` | Orchestrates the complete indexing job |
| `application.properties` | Adds Gemini, PGVector and indexing configuration |

### Existing components used by today's pipeline

These were already present and are reused:

```text
AppConfig.java
GithubApiClient.java
GitHubRateLimiter.java
UserService.java
RepositoryRepository.java
Repository entity
IndexStatus
```

---

# 34. WHY THESE DESIGN CHOICES?

### Why GitHub Tree API first?

The indexer needs the repository file list before it can decide what should be downloaded.

### Why filter before reading file contents?

It prevents unnecessary GitHub content requests and avoids embedding generated/dependency files.

### Why chunk?

Large files are too coarse for precise semantic retrieval.

### Why metadata?

Embeddings represent semantic similarity; metadata identifies repository/file/chunk origin.

### Why batch vector writes?

Fewer calls and better throughput than adding every chunk individually.

### Why async?

Repository indexing can take too long for a normal blocking HTTP request.

### Why progress tracking?

The frontend needs observable job state and counts.

### Why delete old vectors?

To avoid stale data when re-indexing the same repository.

### Why repository filtering during retrieval?

A query for Repository A should not retrieve chunks belonging to Repository B.

### Why SSE?

It lets the browser display generated content incrementally.

---

# 35. INTERVIEW QUESTIONS YOU SHOULD BE READY FOR

### Q1. What is indexing in your project?

> We prepare a GitHub repository for semantic search by filtering files, splitting the files into chunks, generating embeddings through the vector-store/embedding integration, and storing the resulting vectors with repository metadata in PGVector.

### Q2. What is an embedding?

> An embedding is a numerical vector representation of text or code that captures semantic information, allowing similarity-based retrieval.

### Q3. Why PGVector?

> PostgreSQL is already part of the backend stack, and PGVector lets us store and search vector embeddings in the database.

### Q4. Why use metadata with embeddings?

> Metadata lets us restrict searches to a repository and trace retrieved chunks back to their source files.

### Q5. Why is indexing asynchronous?

> Indexing can involve many GitHub requests and vector operations, so it runs in a background executor instead of blocking the HTTP request.

### Q6. Why return HTTP 202?

> The server accepted the indexing request, but the background job is still running.

### Q7. Why chunk the source code?

> Chunking makes semantic retrieval more precise because the system can retrieve the relevant section rather than a whole large file.

### Q8. What happens if one file fails?

> The current implementation logs the error and continues with the remaining files. A whole-job exception is recorded as `FAILED`.

### Q9. Why batch vector writes?

> It reduces the number of vector-store/model-layer calls and improves indexing efficiency.

### Q10. Difference between indexing and retrieval?

> Indexing prepares and stores searchable knowledge. Retrieval searches that prepared knowledge for a user question.

### Q11. How do you avoid mixing repositories?

> Each chunk stores a `repoId` metadata value, and retrieval applies a repository-specific metadata filter.

### Q12. Why stream the AI response?

> Streaming allows the UI to receive and render tokens as they are generated rather than waiting for the complete answer.

---

# 36. THE 30-SECOND PROJECT EXPLANATION

> **"In DevPilot, we use RAG to let the AI understand a user's GitHub repository. During indexing, we fetch the repository tree from GitHub, filter out irrelevant files, read the source files, split them into smaller chunks, and send those documents through the Spring AI vector-store/embedding layer so their vectors can be stored in PostgreSQL using PGVector. When the user asks a question, we perform semantic similarity search to retrieve the most relevant code chunks, inject that context into a prompt, and send it to the configured Gemini model. The response is streamed back to the frontend using SSE, with source metadata used for citations."**

This is the explanation to memorize.

---

# 37. IMPORTANT IMPLEMENTATION NOTES / CLEANUP

These are observations from the repository snapshot and are separate from the documentation itself.

## 37.1 Update stale provider comments

Two new AI classes still contain comments that mention **OpenAI**, while the current repository configuration uses **Gemini**.

For example, a comment such as:

```java
// Builds the prompts sent to OpenAI.
```

would be more accurate as:

```java
// Builds the prompts used by the configured Spring AI chat model.
```

This avoids misleading documentation after the provider switch.

## 37.2 Keep credentials out of Git

The correct pattern in `application.properties` is:

```properties
spring.ai.google.genai.api-key=${GOOGLE_API_KEY}
```

Actual API keys and OAuth client secrets should never be committed.

## 37.3 Verify vector dimensions

The current configuration explicitly sets:

```properties
spring.ai.vectorstore.pgvector.dimensions=1536
```

The configured embedding model and the PGVector column must agree on vector dimensionality in the real runtime environment.

## 37.4 Full re-index vs incremental indexing

Today's implementation rebuilds repository vectors from scratch by deleting existing vectors before indexing.

A future incremental design could be:

```text
Git commit comparison
        ↓
Only changed files
        ↓
Re-index changed files
```

That could reduce work for large repositories.

## 37.5 Keep the diagram synchronized

The attached Excalidraw shows an **OpenAI Embedding API** box, while the repository implementation uses **Google Gemini embeddings**. Update the diagram before using it in the project presentation.

---

# 38. FINAL MENTAL MODEL

Remember this:

```text
INDEXING
"Prepare the repository."

FETCH
↓
FILTER
↓
CHUNK
↓
EMBED
↓
STORE
```

Then:

```text
RAG QUERY
"Find what matters."

QUESTION
↓
SEARCH
↓
RETRIEVE
↓
PROMPT
↓
GEMINI
↓
ANSWER
```

### One-line memory trick

> **Indexing prepares the code. Retrieval finds the code. Generation explains the code.**

---

## SOURCE REFERENCE

Repository: `purnenduachary/DevPilot`

Relevant commit: `aebae04621cd446319edeec4bbbf8489aca9af34`

Commit message: `feat: integrate Gemini AI and repository indexing`
