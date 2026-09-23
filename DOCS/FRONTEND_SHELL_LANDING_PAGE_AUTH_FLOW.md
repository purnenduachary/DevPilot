# FRONTEND SHELL, LANDING PAGE & AUTH FLOW

**Project:** DevPilot  
**Commit:** `b38c4266e839649d2595fe2fcb4caad747633232`  
**Branch:** `main`

## 1. WHAT WE IMPLEMENTED

This implementation turned the frontend into an authenticated developer workspace:

```text
Landing
  ↓
GitHub OAuth
  ↓
/auth/callback
  ↓
/api/auth/me
  ↓
Protected app
  ├─ Repositories
  ├─ Overview
  ├─ Settings
  └─ Chat
```

Main frontend additions:

```text
Pages              → landing, login, callback, dashboard, overview, settings, chat
Shell               → sidebar, navigation, profile menu, header
Auth                → use-auth, RequireAuth, proxy
API                 → typed HTTP client
Repositories        → queries, indexing, filters, cards
Chat                → React Query + SSE streaming
Shared utilities    → query keys, navigation, icons
```

---

# 2. ROUTES

## `client/app/page.tsx` — Landing Page

Public entry point.

### What
- DevPilot branding
- `Continue with GitHub`
- Sign-in link
- Product explanation

The GitHub CTA uses:

```ts
getGithubLoginUrl()
```

### Why
The frontend only needs the backend OAuth entry URL; the OAuth implementation stays on the backend.

### Interview
**Why `<Link>` for sign-in but `<a>` for OAuth?**

`Link` is for internal Next.js navigation. The GitHub button navigates to the backend OAuth endpoint, so a normal anchor is appropriate.

---

## `client/app/login/page.tsx` — Login

### What
- Reads `error` and `next`
- Checks current authentication
- Redirects already-authenticated users
- Starts GitHub OAuth
- Uses `Suspense` around `useSearchParams()`

Example:

```text
/login?next=/dashboard/settings
```

The code only accepts an internal path beginning with `/`; otherwise it uses `/dashboard`.

### Interview
**Why use `Suspense` here?**

The login content depends on `useSearchParams()`, so the parameter-dependent client UI is isolated behind a loading fallback.

---

## `client/app/auth/callback/page.tsx` — OAuth Callback UI

```text
Backend completes GitHub OAuth
            ↓
Frontend reaches /auth/callback
            ↓
useCurrentUser()
       ┌────┴────┐
       ↓         ↓
     user     no user
       ↓         ↓
 /dashboard   /login?error=session
```

The backend performs the OAuth exchange. This page confirms the resulting session and performs the frontend redirect.

### Interview
**Does this page exchange the GitHub authorization code?**

No. Spring Security handles that. The frontend validates the resulting session by calling `/api/auth/me`.

---

## `client/app/dashboard/page.tsx`

Protected repository page:

```text
RequireAuth → AppShell → RepoDashboard
```

Uses `hideHeader` because the repository dashboard has its own header/filter UI.

## `client/app/dashboard/overview/page.tsx`

```text
RequireAuth → AppShell → OverviewDashboard
```

## `client/app/dashboard/settings/page.tsx`

```text
RequireAuth → AppShell → SettingsDashboard
```

## `client/app/chat/[repoId]/page.tsx`

Dynamic route:

```text
/chat/<repoId>
```

`repoId` is resolved from the Next.js `params` promise and passed to `ChatView`.

---

# 3. APP SHELL

## `client/components/layout/app-shell.tsx`

The shared authenticated layout.

### Responsibilities

```text
Sidebar
Navigation
Active route
User profile menu
Settings / logout
Page header
Theme toggle
Page content
```

Reusable props:

```ts
title?
description?
actions?
hideHeader?
children
```

### Important APIs

```ts
usePathname()
```

Reads the current pathname.

```ts
useRouter()
```

Performs programmatic navigation such as Settings and logout redirect.

```ts
React.ReactNode
```

Allows flexible renderable content for `children` and `actions`.

### `SidebarProvider`

Provides sidebar state/context to components such as `Sidebar`, `SidebarTrigger`, and `SidebarInset`.

### `render={...}`

The project uses composition such as:

```tsx
render={<Link href="..." />}
```

This allows the component to use the supplied element rather than producing another nested interactive element.

That matters because:

```html
<button>
  <button>...</button>
</button>
```

is invalid HTML and can produce hydration errors.

### Interview

**Why have an `AppShell` instead of repeating the sidebar in every page?**

To keep shared layout behavior in one place and make individual pages responsible only for their own content.

---

# 4. AUTHENTICATION

## `client/hooks/use-auth.ts`

Provides:

```ts
useCurrentUser()
useLogout()
```

### `useCurrentUser()`

Calls:

```text
GET /api/auth/me
```

Success:

```text
backend user
   ↓
React Query cache
   ↓
devpilot_auth=1
```

Failure:

```text
request fails
   ↓
devpilot_auth cleared
```

Configuration:

```text
staleTime = 5 minutes
retry = false
```

### `useLogout()`

```text
logout.mutate()
      ↓
POST /api/auth/logout
      ↓
clear auth cookie
      ↓
clear cached user
      ↓
invalidate auth queries
      ↓
/login
```

### Interview

**Why React Query for authentication state?**

The current user is server state. React Query gives caching, loading/error state and invalidation without manually implementing all of that.

**What is `staleTime`?**

How long the query result is considered fresh.

**Why `retry: false`?**

A failed authentication check should not automatically retry repeatedly.

---

# 5. `RequireAuth`

## `client/components/providers/require-auth.tsx`

Component-level guard:

```text
useCurrentUser()
   ├─ loading → spinner
   ├─ no user → /login
   └─ user → children
```

### Why

It avoids duplicating authentication checks in every protected component.

### Interview

**Is `RequireAuth` sufficient for backend security?**

No. It protects the frontend UI. Spring Security must still protect backend API endpoints.

---

# 6. `proxy.ts`

## `client/proxy.ts`

Route-level protection.

Protected:

```text
/dashboard/*
/chat/*
```

Special cases:

```text
/auth/callback → allowed
/login + authenticated → /dashboard
```

Unauthenticated protected requests are redirected to:

```text
/login?next=<requested-path>
```

The proxy checks:

```text
devpilot_auth=1
```

### Important security point

This cookie is a **routing signal**, not a trusted authorization mechanism. A browser-controlled cookie can be changed by the client, so the backend remains the real security boundary.

### Interview

**Why both `proxy.ts` and `RequireAuth`?**

```text
proxy.ts      → route-level protection
RequireAuth   → component/UI protection
Spring Security → actual API authorization
```

---

# 7. API LAYER

## `client/lib/api.ts`

Central frontend HTTP client.

### Types

```text
User
Repository
IndexStatus
IndexStatusResponse
ChatSession
ChatMessage
Citation
```

### `ApiError`

Stores HTTP status with the application error.

### `apiFetch<T>()`

Centralizes:

```text
base URL
fetch()
JSON headers
credentials
non-2xx handling
204 handling
typed response
```

The client sends:

```ts
credentials: "include"
```

so browser session cookies are included.

### API methods

```text
AUTH
  me()
  logout()

REPOSITORIES
  listRepos()
  getRepo()
  startIndex()
  indexStatus()

CHAT
  createSession()
  listSessions()
  getMessages()
```

### Interview

**Why use `apiFetch<T>()`?**

To avoid repeating URL construction, headers, credentials and error handling in every component.

**Does `fetch()` throw on HTTP 500?**

No. The promise resolves with a `Response`; the code explicitly checks `res.ok`.

**Why use a generic `<T>`?**

It lets the same HTTP function return different typed response models.

---

# 8. REPOSITORY HOOKS

## `client/hooks/use-repos.ts`

Exports:

```text
useRepos()
useRepository()
useIndexStatus()
useStartIndexing()
useRefreshRepos()
getRepoProgress()
```

### Polling

When the repository status is:

```text
INDEXING
```

React Query periodically refetches.

```text
indexing backend
      ↓
status changes
      ↓
query polling
      ↓
UI updates
```

### Start indexing

Calls:

```text
POST /api/repos/{id}/index
```

Then updates relevant query cache data and invalidates the status query.

### Progress

```text
filesProcessed / filesTotal × 100
```

with protection against zero and a maximum of 100%.

### Interview

**Why poll only while indexing?**

Because continuous polling is unnecessary after indexing reaches a stable state.

**Why update query cache after a mutation?**

So the UI can reflect the new repository state immediately.

---

# 9. REPOSITORY DASHBOARD

## `client/components/dashboard/repo-dashboard.tsx`

Handles:

```text
fetch
search
visibility filters
status filters
sync
loading
errors
empty states
RepoCard rendering
```

The filtered result is derived with `useMemo()`.

### Interview

**Why `useMemo()` here?**

The filtered array is derived from existing state. Memoization avoids recalculating it when unrelated values change.

---

## `client/components/dashboard/dashboard-header.tsx`

Contains:

```text
search
Sync
visibility filters
status filters
repository counts
theme toggle
sidebar trigger
```

`FilterPill` keeps filter buttons reusable.

---

## `client/components/dashboard/repo-card.tsx`

Shows:

```text
owner / repo
language
branch
visibility
index status
chunks
index progress
errors
GitHub link
Chat / Open / Index / Retry
```

Main state flow:

```text
READY     → Open chat
PENDING   → Index
INDEXING  → Show progress
FAILED    → Show error + Retry
```

---

## `client/components/dashboard/repo-status.tsx`

Contains:

```text
indexStatusLabel()
IndexStatusBadge
languageColor()
RepoMeta
```

Keeps status-specific UI logic separated from `RepoCard`.

---

## `client/components/dashboard/index-error-alert.tsx`

Summarizes long indexing errors and lets the user expand the full message.

---

## `client/components/dashboard/language-badge.tsx`

Small wrapper around `LanguageIcon` for consistent language display.

---

# 10. OVERVIEW + SETTINGS

## `client/components/dashboard/overview-dashboard.tsx`

Calculates:

```text
total repositories
ready count
indexing count
failed count
total chunks
recent repositories
```

Uses loading skeletons and empty states.

## `client/components/dashboard/settings-dashboard.tsx`

Shows:

```text
GitHub profile
avatar
username
authentication method
dark mode
theme selector
logout
```

Theme state comes from `next-themes`.

---

# 11. CHAT

## `client/hooks/use-chat.ts`

Provides:

```text
useChatSessions()
useChatMessages()
useCreateChatSession()
useStreamChat()
```

### Optimistic message

The user's message is inserted into the React Query cache before the server response completes.

```text
User sends
   ↓
temporary message in UI
   ↓
server request
   ↓
replace/reconcile with real message
```

### Streaming state

The hook manages:

```text
streaming
streamText
AbortController
```

so the UI can show tokens while they arrive and stop the stream.

---

## `client/lib/stream-chat.ts`

Parses the streaming response.

```text
POST message endpoint
       ↓
ReadableStream
       ↓
TextDecoder
       ↓
SSE-style events
       ↓
token / user_message / assistant_message / done
```

### Interview

**Why `ReadableStream` instead of `response.json()`?**

Because `json()` waits for the complete response. A stream lets the UI process output incrementally.

**Why `AbortController`?**

To cancel an in-progress request.

---

# 12. QUERY KEYS

## `client/lib/query-keys.ts`

Central cache structure:

```text
auth
 └─ me

repos
 ├─ list
 ├─ detail(id)
 └─ status(id)

chat
 ├─ sessions(repositoryId)
 └─ messages(sessionId)
```

### Why?

Avoids scattered magic strings and makes `setQueryData()` / `invalidateQueries()` predictable.

### Interview

**Why structured query keys?**

They uniquely identify related pieces of server state while allowing broad or specific cache operations.

---

# 13. NAVIGATION + ICONS

## `client/lib/dashboard-nav.ts`

Navigation is configuration-driven:

```text
Workspace
 ├─ Overview
 └─ Repositories

Account
 └─ Settings
```

`isDashboardNavActive()` supports exact and nested route matching.

## Icons

```text
devpilot-icon.tsx
→ DevPilotIcon / DevPilotLogo

github-icon.tsx
→ reusable GitHub SVG

language-icon.tsx
→ language → Simple Icon mapping
```

Example:

```text
Java       → SiOpenjdk
Python     → SiPython
TypeScript → SiTypescript
Rust       → SiRust
Dockerfile → SiDocker
```

Unknown languages fall back to `Code2`.

---

# 14. FRONTEND ARCHITECTURE

The implementation follows a clear separation:

```text
Pages
  ↓
Components
  ↓
Hooks
  ↓
API / streaming utilities
  ↓
Spring Boot API
```

More specifically:

```text
Pages
→ routing and composition

Components
→ UI and presentation

Hooks
→ React Query state + UI behavior

lib/api.ts
→ HTTP communication

lib/stream-chat.ts
→ streaming protocol handling

Spring Boot
→ authentication, authorization, repository data and backend logic
```

---

# 15. MOST IMPORTANT INTERVIEW QUESTIONS

### Next.js

**What does `"use client"` do?**  
It marks a component as a Client Component so it can use client-side hooks and event handlers.

**`Link` vs `useRouter()`?**  
`Link` is declarative navigation in rendered UI. `useRouter()` is programmatic navigation triggered by logic/events.

**What is a dynamic route?**  
`[repoId]` creates a route parameter that can represent different repository IDs.

### React

**Why `useMemo()`?**  
To memoize a derived calculation such as filtered repositories.

**Why `ReactNode`?**  
It represents renderable React content and makes components such as `AppShell` reusable.

### React Query

**What is server state?**  
Data owned by the backend that the frontend fetches and caches.

**What does invalidation do?**  
Marks matching query data stale so it can synchronize with the server.

**What is optimistic UI?**  
Showing the expected result immediately before the server confirms it.

### Authentication

**Where is the real security boundary?**  
Spring Security on the backend.

**Why does `/api/auth/me` matter?**  
It confirms the session and provides the authenticated user's data.

**Why not trust `devpilot_auth` for authorization?**  
Because the browser can modify client-side cookies.

### HTTP / API

**Why centralize `fetch()`?**  
To avoid duplicating request/response/error logic.

**Why `credentials: "include"`?**  
Because the backend uses browser session cookies.

### Streaming

**Why streaming?**  
To display generated output incrementally.

**What does `AbortController` do?**  
Cancels an active request.

---

# 16. ONE-MINUTE EXPLANATION

> “I structured the frontend as a layered Next.js application. Users enter through a landing/login flow and authenticate with GitHub OAuth handled by Spring Security. After authentication, the frontend checks `/api/auth/me` and protects dashboard routes using a Next.js proxy and a reusable `RequireAuth` component. `AppShell` provides the shared sidebar, navigation and account controls. React Query manages authentication, repository, indexing and chat server state. Repository indexing is polled while active, and chat responses are consumed incrementally through a `ReadableStream` with SSE-style event parsing and `AbortController` cancellation. The API layer is centralized in `lib/api.ts`, keeping HTTP details separate from UI components.”

---

# 17. THE MENTAL MODEL

Remember this:

```text
AUTH
GitHub
  ↓
Spring Security session
  ↓
/api/auth/me
  ↓
React Query
  ↓
RequireAuth + proxy

DATA
Component
  ↓
Hook
  ↓
api.ts
  ↓
Spring Boot

CHAT
Chat UI
  ↓
useStreamChat
  ↓
stream-chat.ts
  ↓
ReadableStream / SSE events
  ↓
UI
```

**Don't memorize Tailwind classes. Be able to explain the responsibility of each file, the data flow, and the security boundary.**
