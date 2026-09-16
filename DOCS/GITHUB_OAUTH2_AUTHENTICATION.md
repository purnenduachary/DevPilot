# DevPilot — GitHub OAuth2 Authentication
## Presentation + Line-by-Line Learning Guide

> **Purpose of this document:** Understand the GitHub OAuth2 implementation in DevPilot well enough to explain it in a presentation, even if you are still learning Spring Security and the codebase.
>
> The code examples below are based on the current DevPilot implementation. Comments are added to explain **what each important line does, why it exists, and what happens next**.

---

# 1. What We Implemented

DevPilot now has the foundation for **GitHub OAuth2 authentication**.

The goal is:

```text
User
  ↓
Clicks "Login with GitHub"
  ↓
DevPilot sends user to GitHub
  ↓
GitHub authenticates the user
  ↓
GitHub redirects back to DevPilot
  ↓
Spring Security receives GitHub user information
  ↓
GitHubOAuth2UserService processes that information
  ↓
UserService creates OR updates the DevPilot user
  ↓
OAuth access token is encrypted before storage
  ↓
AppUserPrincipal represents the logged-in DevPilot user
  ↓
Spring Security stores authentication in the session
  ↓
Frontend receives the callback
  ↓
Frontend can call authenticated APIs
```

This is the central story to remember.

---

# 2. What Changed Today

The authentication implementation spans these packages:

```text
devPilot.backend
│
├── config/
│   ├── AppConfig.java
│   ├── CorsConfig.java
│   ├── CryptoConfig.java
│   └── SecurityConfig.java
│
├── controllers/
│   └── AuthController.java
│
├── dto/
│   └── UserResponse.java
│
├── entity/
│   └── User.java
│
├── repository/
│   └── UserRepository.java
│
├── security/
│   ├── AppUserPrincipal.java
│   ├── CurrentUser.java
│   └── GitHubOAuth2UserService.java
│
└── services/
    └── UserService.java
```

Also updated:

```text
backend/src/main/resources/application.properties
backend/pom.xml
```

---

# 3. The Big Picture: Who Does What?

| File | Simple meaning |
|---|---|
| `SecurityConfig` | Tells Spring Security how authentication and authorization should work |
| `GitHubOAuth2UserService` | Receives the GitHub user returned by OAuth2 and converts it into a DevPilot user |
| `UserService` | Creates/updates the user and stores the encrypted GitHub token |
| `UserRepository` | Talks to the database for `User` records |
| `User` | Represents the user table in PostgreSQL |
| `AppUserPrincipal` | Wraps our `User` so Spring Security knows who is logged in |
| `CurrentUser` | Gives application code an easy way to get the current logged-in user |
| `AuthController` | Provides frontend-facing authentication endpoints |
| `UserResponse` | Safe response object returned to the frontend |
| `CryptoConfig` | Creates the token encryption object |
| `CorsConfig` | Allows the frontend and backend to communicate across origins |
| `AppConfig` | Provides shared infrastructure beans such as `RestClient.Builder` |
| `application.properties` | Provides OAuth2, session, CORS, crypto, database and app configuration |

---

# 4. Package: `config`

## 4.1 `SecurityConfig.java`

### What this file is responsible for

This is the **main security control center**.

It defines:

- which endpoints are public
- which endpoints require authentication
- how GitHub OAuth2 login works
- what happens after successful login
- what happens after failed login
- how logout works
- how the HTTP session is handled
- how unauthenticated requests are handled

### Code with learning comments

```java
package devPilot.backend.config;

// Lombok generates a constructor for all final fields.
import lombok.RequiredArgsConstructor;

// Reads values such as app.frontend-url from application.properties.
import org.springframework.beans.factory.annotation.Value;

// Lets Spring discover this class as a configuration class.
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

// Used to allow HTTP OPTIONS requests.
// Browsers commonly send OPTIONS requests for CORS preflight.
import org.springframework.http.HttpMethod;

// Provides HTTP status constants such as 401 and 204.
import org.springframework.http.HttpStatus;

// Provides helper configuration such as CORS defaults.
import org.springframework.security.config.Customizer;

// Main Spring Security HTTP configuration API.
import org.springframework.security.config.annotation.web.builders.HttpSecurity;

// Enables Spring Web Security configuration.
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;

// Controls whether Spring creates an HTTP session and how it behaves.
import org.springframework.security.config.http.SessionCreationPolicy;

// Represents the configured Spring Security filter chain.
import org.springframework.security.web.SecurityFilterChain;

// Authentication success/failure handlers and related classes.
import org.springframework.security.web.authentication.*;

// Our custom GitHub OAuth2 user service.
import devPilot.backend.security.GitHubOAuth2UserService;


@Configuration
// Tells Spring that this class contains application configuration.

@EnableWebSecurity
// Enables Spring Security's web security support.

@RequiredArgsConstructor
// Lombok generates a constructor for the final fields below.
public class SecurityConfig {

    // Spring injects our custom GitHub OAuth2 service here.
    // This service will process the user returned by GitHub.
    private final GitHubOAuth2UserService gitHubOAuth2UserService;

    // Handler that runs after GitHub authentication succeeds.
    private final AuthenticationSuccessHandler oauth2successHandler;

    // Handler that runs if OAuth2 authentication fails.
    private final AuthenticationFailureHandler oauth2FailureHandler;


    @Bean
    // Registers the returned SecurityFilterChain as a Spring bean.
    SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {

        http
                // Uses the application's CORS configuration.
                .cors(Customizer.withDefaults())

                // Disables CSRF protection for this session-based API setup.
                .csrf(csrf -> csrf.disable())

                // We need a session because the OAuth2 login is session based.
                .sessionManagement(session ->
                        session.sessionCreationPolicy(
                                SessionCreationPolicy.IF_REQUIRED
                        )
                )

                // Defines which URLs require authentication.
                .authorizeHttpRequests(auth -> auth

                        // These endpoints must be publicly accessible.
                        .requestMatchers(
                                "/api/auth/login-url",
                                "/oauth2/**",
                                "/login/oauth2/**",
                                "/error"
                        ).permitAll()

                        // CORS preflight requests are allowed.
                        .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()

                        // Everything under /api requires an authenticated user.
                        .requestMatchers("/api/**").authenticated()

                        // Any remaining endpoint is public.
                        .anyRequest().permitAll()
                )

                // If an unauthenticated user accesses a protected endpoint,
                // return HTTP 401 instead of redirecting to a login page.
                .exceptionHandling(ex ->
                        ex.authenticationEntryPoint(
                                new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED)
                        )
                )

                // Enables OAuth2 Login.
                .oauth2Login(oauth -> oauth

                        // Tells Spring which service should load/process
                        // the GitHub user's information.
                        .userInfoEndpoint(userInfo ->
                                userInfo.userService(gitHubOAuth2UserService)
                        )

                        // Runs after authentication succeeds.
                        .successHandler(oauth2successHandler)

                        // Runs if authentication fails.
                        .failureHandler(oauth2FailureHandler)
                )

                // Defines the logout behaviour.
                .logout(logout -> logout

                        // Frontend/backend will use this endpoint for logout.
                        .logoutUrl("/api/auth/logout")

                        // Return HTTP 204 after successful logout.
                        .logoutSuccessHandler(
                                (request, response, authentication) ->
                                        response.setStatus(
                                                HttpStatus.NO_CONTENT.value()
                                        )
                        )

                        // Destroy the HTTP session.
                        .invalidateHttpSession(true)

                        // Clear the authentication object.
                        .clearAuthentication(true)

                        // Remove our custom session cookie.
                        .deleteCookies("DEVPILOT_SESSION")
                );

        // Builds the final Spring Security filter chain.
        return http.build();
    }


    @Bean
    // Registers the OAuth2 success handler as a Spring bean.
    AuthenticationSuccessHandler oauth2SuccessHandler(
            @Value("${app.frontend-url}") String frontendUrl
    ) {
        // Spring handler that redirects the browser to a URL.
        SimpleUrlAuthenticationSuccessHandler handler =
                new SimpleUrlAuthenticationSuccessHandler();

        // After GitHub login, redirect the browser to the frontend callback.
        handler.setDefaultTargetUrl(
                frontendUrl + "/auth/callback"
        );

        return handler;
    }


    @Bean
    // Registers the OAuth2 failure handler as a Spring bean.
    AuthenticationFailureHandler oauth2FailureHandler(
            @Value("${app.frontend-url}") String frontendUrl
    ) {
        // Spring handler that redirects the browser after failure.
        SimpleUrlAuthenticationFailureHandler handler =
                new SimpleUrlAuthenticationFailureHandler();

        // Add an error query parameter to the frontend login page.
        handler.setDefaultFailureUrl(
                frontendUrl + "/login?error=oauth_failed"
        );

        return handler;
    }
}
```

### Important thing to remember

The nesting is intentional:

```text
oauth2Login()
│
├── userInfoEndpoint()
│     └── userService()
│
├── successHandler()
│
└── failureHandler()
```

`successHandler()` and `failureHandler()` belong to the OAuth2 login configuration, **not** inside `userInfoEndpoint()`.

---

# 5. Package: `security`

## 5.1 `GitHubOAuth2UserService.java`

### Why this class exists

GitHub returns an OAuth2 user.

DevPilot needs to:

1. receive that GitHub user
2. read the access token
3. read the granted scopes
4. save or update the corresponding DevPilot user
5. return a Spring Security principal

This class is the bridge between:

```text
GitHub OAuth2
      ↓
DevPilot User
```

### Code with learning comments

```java
package devPilot.backend.security;

// Our own User entity.
import devPilot.backend.entity.User;

// Business logic for creating/updating users.
import devPilot.backend.services.UserService;

// Generates the required constructor.
import lombok.RequiredArgsConstructor;

// Default implementation supplied by Spring Security.
import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;

// Contains information about the current OAuth2 request.
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;

// Interface implemented by OAuth2 user services.
import org.springframework.security.oauth2.client.userinfo.OAuth2UserService;

// Exception type used by OAuth2 authentication.
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;

// Represents the authenticated OAuth2 user.
import org.springframework.security.oauth2.core.user.OAuth2User;

// Makes this class a Spring service bean.
import org.springframework.stereotype.Service;


@Service
// Spring automatically creates this class as a bean.

@RequiredArgsConstructor
// Generates a constructor for userService.
public class GitHubOAuth2UserService
        implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

    // We do not save directly to the repository here.
    // UserService owns the user persistence/business logic.
    private final UserService userService;

    // Spring's standard OAuth2 service does the actual user-info request.
    private final DefaultOAuth2UserService delegate =
            new DefaultOAuth2UserService();


    @Override
    // This method is called by Spring Security during OAuth2 login.
    public OAuth2User loadUser(
            OAuth2UserRequest userRequest
    ) throws OAuth2AuthenticationException {

        // Ask Spring's default service to retrieve the GitHub user.
        OAuth2User githubUser = delegate.loadUser(userRequest);

        // Extract the raw OAuth2 access token returned by GitHub.
        String accessToken =
                userRequest.getAccessToken().getTokenValue();

        // Read the scopes granted to the token.
        // Example: "read:user,repo".
        String scopes =
                userRequest.getAccessToken().getScopes() != null
                        ? String.join(
                                ",",
                                userRequest.getAccessToken().getScopes()
                        )
                        : "read:user,repo";

        // Send GitHub data to our application service.
        // UserService creates or updates the database record
        // and encrypts the token before saving it.
        User user = userService.upsertFromGitHub(
                githubUser.getAttributes(),
                accessToken,
                scopes
        );

        // Wrap our database user inside a Spring Security principal.
        // This becomes the identity of the authenticated DevPilot user.
        return new AppUserPrincipal(
                user,
                githubUser.getAttributes()
        );
    }
}
```

### Key idea

This class does **not** directly save to PostgreSQL.

Instead:

```text
GitHubOAuth2UserService
        ↓
UserService
        ↓
UserRepository
        ↓
PostgreSQL
```

This separation keeps security logic and persistence logic cleaner.

---

# 6. `AppUserPrincipal.java`

## Why this class exists

Spring Security needs an object representing the currently authenticated user.

GitHub gives us an `OAuth2User`, but DevPilot also has its own `User` entity.

`AppUserPrincipal` connects the two.

### Code with learning comments

```java
package devPilot.backend.security;

// Used to represent a collection of authorities/roles.
import java.util.Collection;

// Map containing GitHub OAuth2 attributes.
import java.util.Map;

// Our database entity.
import devPilot.backend.entity.User;

// Used for the internal UUID.
import java.util.UUID;

// Spring Security permission interface.
import org.springframework.security.core.GrantedAuthority;

// Helper for creating authorities.
import org.springframework.security.core.authority.AuthorityUtils;

// Interface for OAuth2 users.
import org.springframework.security.oauth2.core.user.OAuth2User;


public class AppUserPrincipal implements OAuth2User {

    // Our actual DevPilot database user.
    private final User user;

    // Original GitHub attributes such as id, login, avatar_url, etc.
    private final Map<String, Object> attributes;


    // Constructor used when GitHub login finishes.
    public AppUserPrincipal(
            User user,
            Map<String, Object> attributes
    ) {
        this.user = user;
        this.attributes = attributes;
    }


    // Gives application code access to the internal DevPilot UUID.
    public UUID getId() {
        return user.getId();
    }


    // Gives application code the complete DevPilot User entity.
    public User getUser() {
        return user;
    }


    @Override
    // Spring Security can access the original OAuth2 attributes.
    public Map<String, Object> getAttributes() {
        return attributes;
    }


    @Override
    // Defines the role/permission of this authenticated user.
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return AuthorityUtils.createAuthorityList("ROLE_USER");
    }


    @Override
    // Spring Security uses this as the principal name.
    // DevPilot chooses its own internal UUID rather than GitHub username.
    public String getName() {
        return user.getId().toString();
    }
}
```

### Why `ROLE_USER`?

Right now every successfully authenticated DevPilot user is treated as:

```text
ROLE_USER
```

This gives us a simple authorization foundation for later.

---

# 7. `CurrentUser.java`

## Why this class exists

Without this helper, every controller/service that needs the current user would have to repeat:

```java
SecurityContextHolder.getContext().getAuthentication()
```

`CurrentUser` centralizes that logic.

### Code with learning comments

```java
package devPilot.backend.security;

// Custom exception used by DevPilot.
import devPilot.backend.exceptions.UnauthorizedException;

// Represents the currently authenticated Spring Security object.
import org.springframework.security.core.Authentication;

// Gives access to the current security context.
import org.springframework.security.core.context.SecurityContextHolder;

// Registers this helper as a Spring component.
import org.springframework.stereotype.Component;


@Component
public class CurrentUser {

    public AppUserPrincipal require() {

        // Read the current authentication from Spring Security.
        Authentication auth =
                SecurityContextHolder
                        .getContext()
                        .getAuthentication();

        // Verify that:
        // 1. authentication exists
        // 2. the principal is actually our AppUserPrincipal
        if (auth == null ||
                !(auth.getPrincipal()
                        instanceof AppUserPrincipal principal)) {

            // Stop the request if the user is not authenticated.
            throw new UnauthorizedException("Not authenticated");
        }

        // Return the authenticated DevPilot user.
        return principal;
    }
}
```

### Why this is useful

Controllers can now simply do:

```java
AppUserPrincipal principal = currentUser.require();
```

instead of manually dealing with the Spring Security context.

---

# 8. Package: `services`

## 8.1 `UserService.java`

### Why this file matters

This is where GitHub user information becomes a persistent DevPilot user.

The major operation is:

```text
upsertFromGitHub()
```

**Upsert** means:

```text
If user exists → update it
If user does not exist → create it
```

### Code with learning comments

```java
package devPilot.backend.services;

// Used for GitHub attribute maps.
import java.util.Map;

// Used for internal DevPilot UUID lookups.
import java.util.UUID;

// Our database entity.
import devPilot.backend.entity.User;

// Encrypts the OAuth access token before storage.
import org.springframework.security.crypto.encrypt.TextEncryptor;

// Marks this class as a service bean.
import org.springframework.stereotype.Service;

// Defines transaction boundaries.
import org.springframework.transaction.annotation.Transactional;

// Database access layer.
import devPilot.backend.repository.UserRepository;

// Generates constructor injection.
import lombok.RequiredArgsConstructor;


@Service
// Spring creates UserService as a managed bean.

@RequiredArgsConstructor
// Constructor injection for the repository and encryptor.
public class UserService {

    // Database repository used for user operations.
    public final UserRepository userRepository;

    // Encryptor used to protect GitHub access tokens.
    public final TextEncryptor tokenEncryptor;


    @Transactional
    // The create/update/save operation runs inside a database transaction.
    public User upsertFromGitHub(
            Map<String, Object> attributes,
            String accessToken,
            String scopes
    ) {

        // GitHub's stable user ID.
        Long githubId = toLong(attributes.get("id"));

        // GitHub username/login.
        String login = String.valueOf(
                attributes.get("login")
        );

        // GitHub display name.
        // If GitHub does not provide a name, use the login.
        String name =
                attributes.get("name") != null
                        ? String.valueOf(attributes.get("name"))
                        : login;

        // GitHub profile/avatar URL.
        // Can be null if GitHub does not send it.
        String avatarUrl =
                attributes.get("avatar_url") != null
                        ? String.valueOf(
                                attributes.get("avatar_url")
                        )
                        : null;

        // NEVER store the raw OAuth access token in the database.
        // Encrypt it before saving.
        String encryptedToken =
                tokenEncryptor.encrypt(accessToken);

        // Search for an existing user using GitHub's user ID.
        //
        // Existing user → return that user.
        // Missing user → create a new User object.
        User user =
                userRepository.findByGithubId(githubId)
                        .orElseGet(User::new);

        // Set GitHub identity.
        user.setGithubId(githubId);

        // Save GitHub username.
        user.setGithubUsername(login);

        // Save DevPilot display name.
        user.setDisplayName(name);

        // Save avatar URL.
        user.setAvatarUrl(avatarUrl);

        // Save the ENCRYPTED access token.
        user.setAccessToken(encryptedToken);

        // Save the granted OAuth2 scopes.
        user.setTokenScope(scopes);

        // Insert a new row OR update the existing row.
        return userRepository.save(user);
    }


    @Transactional(readOnly = true)
    // Read-only transaction for lookup operations.
    public User requiredById(UUID id) {

        // Find the user using DevPilot's internal UUID.
        return userRepository.findById(id)
                .orElseThrow(() ->
                        new IllegalArgumentException(
                                "User not found"
                        )
                );
    }


    // Decrypt the GitHub access token when DevPilot needs to use it.
    public String decryptAccessToken(User user) {
        return tokenEncryptor.decrypt(
                user.getAccessToken()
        );
    }


    // GitHub attribute "id" can arrive as a Number or another object type.
    // This helper safely converts it into a Long.
    private static Long toLong(Object value) {

        if (value instanceof Number number) {
            return number.longValue();
        }

        return Long.parseLong(
                String.valueOf(value)
        );
    }
}
```

### The most important sequence

```text
GitHub attributes
      ↓
githubId / login / name / avatar
      ↓
Encrypt access token
      ↓
findByGithubId(githubId)
      ↓
existing user OR new User()
      ↓
set fields
      ↓
save()
```

### One naming detail

The entity field is:

```java
private String tokenScope;
```

Therefore Lombok generates:

```java
setTokenScope(...)
```

not:

```java
setTokenScopes(...)
```

---

# 9. Package: `repository`

## 9.1 `UserRepository.java`

### Purpose

This interface gives Spring Data JPA database operations for the `User` entity.

### Code with learning comments

```java
package devPilot.backend.repository;

// Database entity this repository manages.
import devPilot.backend.entity.User;

// Provides CRUD operations automatically.
import org.springframework.data.jpa.repository.JpaRepository;

// Used for Optional return values.
import java.util.Optional;

// Internal User UUID type.
import java.util.UUID;


public interface UserRepository
        extends JpaRepository<User, UUID> {

    // Find a DevPilot user using GitHub's unique user ID.
    //
    // Spring Data JPA reads the method name and builds
    // the query automatically.
    Optional<User> findByGithubId(Long githubId);
}
```

### Why two IDs exist

DevPilot has:

```text
id
↓
Internal UUID generated by DevPilot

githubId
↓
External identifier supplied by GitHub
```

The internal UUID is the primary key.

The `githubId` lets us reconnect the GitHub identity to the same DevPilot user.

---

# 10. Package: `entity`

## 10.1 `User.java`

### Purpose

This is the JPA entity representing the `users` table.

### Code with learning comments

```java
package devPilot.backend.entity;

import jakarta.persistence.*;

// Lombok annotations such as @Getter, @Setter and @Builder.
import lombok.*;

// Timestamp used for created_at.
import java.time.Instant;

// Internal primary key type.
import java.util.UUID;


@Entity
// Tells JPA that this Java class represents a database entity.

@Getter
// Lombok generates getter methods.

@Setter
// Lombok generates setter methods.

@NoArgsConstructor
// Generates a no-argument constructor required by JPA.

@AllArgsConstructor
// Generates a constructor containing every field.

@Table(name = "users")
// Explicitly maps this entity to the "users" database table.

@Builder
// Allows convenient object creation using the builder pattern.
public class User {

    @Id
    // Marks this field as the primary key.

    @GeneratedValue(strategy = GenerationType.UUID)
    // Hibernate generates the UUID automatically.

    private UUID id;


    @Column(
            name = "github_id",
            unique = true,
            nullable = false
    )
    // GitHub ID is unique because one GitHub account
    // should map to one DevPilot user.

    private Long githubId;


    @Column(
            name = "github_username",
            nullable = false,
            length = 100
    )
    private String githubUsername;


    @Column(
            name = "display_name",
            nullable = false,
            length = 200
    )
    private String displayName;


    @Column(
            name = "avatar_url",
            length = 500
    )
    private String avatarUrl;


    @Column(
            name = "access_token",
            nullable = false,
            columnDefinition = "TEXT"
    )
    // The token is stored as TEXT because it is encrypted data.
    // UserService stores the encrypted version, not the raw token.
    private String accessToken;


    @Column(
            name = "token_scope",
            length = 500
    )
    // Stores the granted OAuth scopes.
    private String tokenScope;


    @Column(
            name = "created_at",
            nullable = false,
            updatable = false
    )
    // Records when the user was first created.
    private Instant createdAt;


    @PrePersist
    // Runs immediately before a new entity is inserted.
    void onCreate() {

        // Only set the timestamp if it has not already been assigned.
        if (createdAt == null) {

            // Store the current instant.
            createdAt = Instant.now();
        }
    }
}
```

### Why `@PrePersist`?

It means:

```text
New User object
      ↓
Before INSERT
      ↓
onCreate()
      ↓
createdAt = now
      ↓
INSERT into users
```

---

# 11. Package: `controllers`

## 11.1 `AuthController.java`

### Purpose

This provides backend endpoints the frontend can call for authentication-related operations.

### Code with learning comments

```java
package devPilot.backend.controllers;

// Database user.
import devPilot.backend.entity.User;

// Safe DTO for returning user information.
import devPilot.backend.dto.UserResponse;

// Generates the controller constructor.
import lombok.RequiredArgsConstructor;

// Spring HTTP response wrapper.
import org.springframework.http.ResponseEntity;

// REST endpoint annotations.
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

// Our helper for getting the authenticated user.
import devPilot.backend.security.AppUserPrincipal;
import devPilot.backend.security.CurrentUser;

// Used for the login URL response.
import java.util.Map;


@RestController
// Marks this class as a REST controller.

@RequestMapping("/api/auth")
// Every endpoint in this class starts with /api/auth.

@RequiredArgsConstructor
// Constructor injection for CurrentUser.
public class AuthController {

    // Helper used to access the logged-in user.
    private final CurrentUser currentUser;


    @GetMapping("/login-url")
    // GET /api/auth/login-url
    public Map<String, String> loginUrl() {

        // Tell the frontend where it should start GitHub OAuth2 login.
        return Map.of(
                "url",
                "/oauth2/authorization/github"
        );
    }


    @GetMapping("/me")
    // GET /api/auth/me
    public ResponseEntity<UserResponse> me() {

        // Get the authenticated DevPilot principal.
        AppUserPrincipal principal =
                currentUser.require();

        // Extract our actual database user.
        User user = principal.getUser();

        // Return a safe DTO rather than the entity itself.
        return ResponseEntity.ok(
                new UserResponse(
                        user.getId(),
                        user.getGithubId(),
                        user.getGithubUsername(),
                        user.getDisplayName(),
                        user.getAvatarUrl()
                )
        );
    }
}
```

### Important point

The controller does **not** return:

```text
accessToken
tokenScope
```

That is intentional.

The frontend gets only the user information it needs.

---

# 12. Package: `dto`

## 12.1 `UserResponse.java`

### Purpose

This is a DTO (Data Transfer Object).

It defines the data DevPilot wants to send back to the frontend.

### Code with learning comments

```java
package devPilot.backend.dto;

import java.util.UUID;


// A Java record automatically provides:
// - constructor
// - accessors
// - equals()
// - hashCode()
// - toString()
public record UserResponse(

        // DevPilot internal UUID.
        UUID id,

        // GitHub's user ID.
        Long githubId,

        // GitHub username.
        String githubUsername,

        // Human-readable display name.
        String displayName,

        // GitHub profile avatar URL.
        String avatarUrl
) {
}
```

### Why use a DTO?

Because returning the JPA entity directly could expose fields that should remain server-side.

For example:

```text
User entity
├── id
├── githubId
├── githubUsername
├── displayName
├── avatarUrl
├── accessToken        ← sensitive
├── tokenScope
└── createdAt
```

But the frontend receives:

```text
UserResponse
├── id
├── githubId
├── githubUsername
├── displayName
└── avatarUrl
```

---

# 13. Package: `config`

## 13.1 `CryptoConfig.java`

### Purpose

Creates the `TextEncryptor` bean used to encrypt and decrypt GitHub access tokens.

### Code with learning comments

```java
package devPilot.backend.config;

// Reads crypto settings from application.properties.
import org.springframework.beans.factory.annotation.Value;

// Configuration annotations.
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

// Spring's encryption utility.
import org.springframework.security.crypto.encrypt.Encryptors;

// Interface used by UserService.
import org.springframework.security.crypto.encrypt.TextEncryptor;


@Configuration
// Tells Spring this class contains bean definitions.
public class CryptoConfig {

    @Bean
    // Registers the TextEncryptor as a Spring bean.
    TextEncryptor tokenEncryptor(

            // Read the encryption password from configuration.
            @Value("${app.token-encryptor-password}")
            String password,

            // Read the salt from configuration.
            @Value("${app.token-encryptor-salt}")
            String salt
    ) {

        // Create and return Spring's text encryptor.
        return Encryptors.text(password, salt);
    }
}
```

### How it connects to `UserService`

```text
CryptoConfig
    ↓ creates TextEncryptor bean
    ↓
Spring dependency injection
    ↓
UserService.tokenEncryptor
    ↓
encrypt(accessToken)
```

---

# 14. Package: `config`

## 14.1 `CorsConfig.java`

### Purpose

Allows the frontend origin to make requests to the backend.

For local development:

```text
Frontend → http://localhost:3000
Backend  → different origin/port
```

Browsers enforce CORS rules, so Spring needs to explicitly allow the frontend.

### Code with learning comments

```java
package devPilot.backend.config;

// Used to convert the comma-separated configuration
// into a Java list.
import java.util.Arrays;
import java.util.List;

// Reads allowed origins from configuration.
import org.springframework.beans.factory.annotation.Value;

// Spring configuration annotations.
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

// Spring CORS classes.
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;


@Configuration
// Marks this as a configuration class.
public class CorsConfig {

    @Bean
    // Makes the CORS configuration available to Spring Security.
    CorsConfigurationSource corsConfigurationSource(

            // Read allowed origins from application.properties.
            @Value("${app.cors.allowed-origins}")
            String allowedOrigins
    ) {

        // Create a new CORS configuration object.
        CorsConfiguration config =
                new CorsConfiguration();

        // Support multiple comma-separated origins.
        List<String> origins =
                Arrays.stream(
                        allowedOrigins.split(",")
                )
                .map(String::trim)
                .filter(s -> !s.isEmpty())
                .toList();

        // Tell Spring which origins are allowed.
        config.setAllowedOrigins(origins);

        // Allow these HTTP methods.
        config.setAllowedMethods(
                List.of(
                        "GET",
                        "POST",
                        "PUT",
                        "PATCH",
                        "DELETE",
                        "OPTIONS"
                )
        );

        // Allow all request headers.
        config.setAllowedHeaders(
                List.of("*")
        );

        // Allow credentials such as cookies/session data.
        config.setAllowCredentials(true);

        // Browser can cache the CORS preflight response for 1 hour.
        config.setMaxAge(3600L);

        // Create the URL-based CORS configuration source.
        UrlBasedCorsConfigurationSource source =
                new UrlBasedCorsConfigurationSource();

        // Apply this configuration to every endpoint.
        source.registerCorsConfiguration(
                "/**",
                config
        );

        // Return the configured CORS source.
        return source;
    }
}
```

---

# 15. Package: `config`

## 15.1 `AppConfig.java`

### Purpose

Provides general application infrastructure.

This file is not the core OAuth2 class, but it was added as part of the current backend configuration and supports other application functionality.

### Code with learning comments

```java
package devPilot.backend.config;

// Java executor abstraction.
import java.util.concurrent.Executor;

// Spring bean/configuration support.
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

// Enables asynchronous execution.
import org.springframework.scheduling.annotation.EnableAsync;

// Thread-pool implementation.
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

// Spring HTTP client builder.
import org.springframework.web.client.RestClient;


@Configuration
// Spring configuration class.

@EnableAsync
// Enables asynchronous method execution in the application.
public class AppConfig {

    @Bean
    // Creates a RestClient.Builder bean.
    RestClient.Builder restClientBuilder() {

        // Returns Spring's REST client builder.
        return RestClient.builder();
    }


    @Bean(name = "indexingExecutor")
    // Registers a named thread pool for indexing work.
    Executor indexingExecutor() {

        // Creates a thread pool executor.
        ThreadPoolTaskExecutor executor =
                new ThreadPoolTaskExecutor();

        // Start with 2 worker threads.
        executor.setCorePoolSize(2);

        // Can scale up to 4 worker threads.
        executor.setMaxPoolSize(4);

        // Queue up to 50 waiting tasks.
        executor.setQueueCapacity(50);

        // Names threads using the "index-" prefix.
        executor.setThreadNamePrefix("index-");

        // Initializes the executor.
        executor.initialize();

        // Returns the executor to Spring.
        return executor;
    }
}
```

### Important presentation point

You can say:

> `AppConfig` provides reusable infrastructure beans. `RestClient.Builder` can be reused for external API calls, while the `indexingExecutor` gives repository/indexing work its own thread pool.

---

# 16. `application.properties`

This file provides the runtime configuration for the authentication system.

> **Security note:** the repository currently contains default-looking/credential-like GitHub OAuth values. Never keep real client secrets in Git. Use environment variables and rotate any secret that has been exposed.

### Relevant configuration

```properties
# Application name
spring.application.name=backend

# PostgreSQL connection
spring.datasource.url=${DB_URL:jdbc:postgresql://localhost:5433/devpilot}
spring.datasource.username=${DB_USERNAME:postgres}
spring.datasource.password=${DB_PASSWORD:postgres}
spring.datasource.driver-class-name=org.postgresql.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.open-in-view=false

# Spring AI
spring.ai.openai.api-key=${OPENAI_API_KEY}
spring.ai.openai.chat.model=gpt-4o-mini
spring.ai.openai.embedding.model=text-embedding-3-small

# GitHub OAuth2
spring.security.oauth2.client.registration.github.client-id=${GITHUB_CLIENT_ID}
spring.security.oauth2.client.registration.github.client-secret=${GITHUB_CLIENT_SECRET}
spring.security.oauth2.client.registration.github.scope=read:user,repo
spring.security.oauth2.client.provider.github.user-name-attribute=id

# Session
server.servlet.session.cookie.name=DEVPILOT_SESSION
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.same-site=lax
server.servlet.session.timeout=7d

# App settings
app.frontend-url=${FRONTEND_URL:http://localhost:3000}
app.cors.allowed-origins=${CORS_ALLOWED_ORIGINS:http://localhost:3000}
app.token-encryptor-password=${TOKEN_ENCRYPTOR_PASSWORD:devpilot-local-encrypt-key-change-me}
app.token-encryptor-salt=${TOKEN_ENCRYPTOR_SALT:deadbeefcafebabe}

# Indexing configuration
app.indexing.max-file-bytes=102400
app.indexing.chunk-size=800
app.indexing.chunk-overlap=100

# GitHub API delay
app.github.api-delay-ms=50
```

## What the important properties mean

### GitHub client ID

```properties
spring.security.oauth2.client.registration.github.client-id=...
```

Identifies DevPilot to GitHub.

### GitHub client secret

```properties
spring.security.oauth2.client.registration.github.client-secret=...
```

Private credential used during OAuth2 authentication.

### Scopes

```properties
spring.security.oauth2.client.registration.github.scope=read:user,repo
```

These define what access DevPilot asks GitHub to grant.

In the current project:

```text
read:user
repo
```

### GitHub user name attribute

```properties
spring.security.oauth2.client.provider.github.user-name-attribute=id
```

The application treats GitHub's `id` attribute as the user's OAuth2 name attribute.

### Session cookie

```properties
server.servlet.session.cookie.name=DEVPILOT_SESSION
```

Spring's session cookie is given a DevPilot-specific name.

### HTTP-only

```properties
server.servlet.session.cookie.http-only=true
```

Helps prevent JavaScript from reading the session cookie.

### SameSite

```properties
server.servlet.session.cookie.same-site=lax
```

Adds browser restrictions to cross-site cookie sending.

### Frontend URL

```properties
app.frontend-url=${FRONTEND_URL:http://localhost:3000}
```

Uses the `FRONTEND_URL` environment variable if available.

Otherwise:

```text
http://localhost:3000
```

is used.

---

# 17. Maven Dependencies

The OAuth2 implementation depends on Spring Security.

Relevant dependencies in `pom.xml`:

```xml
<!-- Core Spring Security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- OAuth2 client support -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security-oauth2-client</artifactId>
</dependency>
```

### Why both?

```text
spring-boot-starter-security
        ↓
Core security framework

spring-boot-starter-security-oauth2-client
        ↓
OAuth2 client + OAuth2 login support
```

Without the OAuth2 client starter, the GitHub login configuration would not have the necessary OAuth2 client functionality.

---

# 18. Complete Authentication Flow

This is the most important section for your presentation.

## Step 1 — Frontend asks for the login URL

The frontend calls:

```http
GET /api/auth/login-url
```

`AuthController` responds:

```json
{
  "url": "/oauth2/authorization/github"
}
```

The frontend can then send the browser to:

```text
/oauth2/authorization/github
```

---

## Step 2 — Spring starts OAuth2 login

`SecurityConfig` enables:

```java
.oauth2Login(...)
```

Spring Security sees the GitHub registration in:

```properties
spring.security.oauth2.client.registration.github...
```

and starts the GitHub OAuth2 flow.

---

## Step 3 — User authenticates on GitHub

GitHub handles:

```text
Identity
Authorization
Consent
```

The user does not send a GitHub password to DevPilot.

---

## Step 4 — GitHub redirects back

GitHub redirects back to the application's OAuth2 callback:

```text
/login/oauth2/code/github
```

Spring Security handles this part.

---

## Step 5 — Spring loads the GitHub user

Spring calls:

```java
GitHubOAuth2UserService.loadUser(...)
```

That class calls:

```java
delegate.loadUser(userRequest)
```

to retrieve the GitHub user information.

---

## Step 6 — Extract GitHub token and scopes

The service gets:

```text
accessToken
scopes
attributes
```

from the OAuth2 request/user.

---

## Step 7 — UserService performs the upsert

The following method is called:

```java
upsertFromGitHub(
    githubUser.getAttributes(),
    accessToken,
    scopes
)
```

The service extracts:

```text
GitHub ID
username
name
avatar URL
```

---

## Step 8 — Token encryption

Before database storage:

```java
tokenEncryptor.encrypt(accessToken)
```

The raw token is transformed into encrypted text.

The database should therefore contain:

```text
Encrypted token
```

not:

```text
Raw GitHub token
```

---

## Step 9 — Find existing user

The repository executes:

```java
findByGithubId(githubId)
```

Two possibilities:

### Existing user

```text
GitHub ID found
      ↓
Load existing User
      ↓
Update current information
```

### New user

```text
GitHub ID not found
      ↓
new User()
      ↓
Set fields
```

---

## Step 10 — Save user

Finally:

```java
userRepository.save(user)
```

JPA performs the appropriate database operation.

---

## Step 11 — Create `AppUserPrincipal`

The service returns:

```java
new AppUserPrincipal(
    user,
    githubUser.getAttributes()
)
```

Spring Security now has a principal representing the DevPilot user.

---

## Step 12 — Session authentication

Because the project uses:

```java
SessionCreationPolicy.IF_REQUIRED
```

Spring can maintain the authenticated state in an HTTP session.

The session cookie is:

```text
DEVPILOT_SESSION
```

---

## Step 13 — Success redirect

The success handler redirects to:

```text
{frontend-url}/auth/callback
```

For local development:

```text
http://localhost:3000/auth/callback
```

---

# 19. What Happens When `/api/auth/me` Is Called?

The frontend calls:

```http
GET /api/auth/me
```

Because `SecurityConfig` says:

```java
.requestMatchers("/api/**").authenticated()
```

the request must be authenticated.

Then:

```java
CurrentUser.require()
```

reads the current `Authentication`.

It checks whether the principal is:

```java
AppUserPrincipal
```

Then:

```java
principal.getUser()
```

returns the DevPilot `User`.

Finally the controller converts it into:

```java
UserResponse
```

and returns safe user information.

---

# 20. Logout Flow

The frontend calls:

```http
POST /api/auth/logout
```

Spring Security:

```text
logout()
  ↓
invalidate session
  ↓
clear authentication
  ↓
delete DEVPILOT_SESSION cookie
  ↓
HTTP 204 No Content
```

No redirect is required.

---

# 21. Why the Design Is Split Into Multiple Classes

The implementation intentionally separates responsibilities.

## SecurityConfig

Knows:

```text
How security works
```

It should not know:

```text
How users are stored in PostgreSQL
```

---

## GitHubOAuth2UserService

Knows:

```text
How GitHub OAuth2 user data enters DevPilot
```

It should not directly contain database queries.

---

## UserService

Knows:

```text
How a DevPilot user is created/updated
```

---

## UserRepository

Knows:

```text
How users are queried from PostgreSQL
```

---

## AppUserPrincipal

Knows:

```text
How DevPilot represents the authenticated user to Spring Security
```

---

## CurrentUser

Knows:

```text
How to retrieve the current authenticated DevPilot user
```

---

## AuthController

Knows:

```text
What authentication information the frontend can request
```

This separation is an important software-engineering point to mention in a presentation.

---

# 22. Presentation Explanation — 60 Second Version

Use this when someone asks:

> **"Explain your authentication implementation."**

Say:

> "DevPilot uses Spring Security with GitHub OAuth2 for authentication. I configured `SecurityConfig` to allow the OAuth2 endpoints, protect the rest of the `/api` routes, and define success, failure and logout behaviour. After GitHub authenticates the user, Spring calls my custom `GitHubOAuth2UserService`. That service retrieves the GitHub user attributes and access token and passes them to `UserService`. `UserService` performs an upsert using GitHub's unique user ID, encrypts the OAuth access token before storing it, and persists the user through `UserRepository`. After that, the user is wrapped in a custom `AppUserPrincipal`, which becomes the authenticated principal in Spring Security. I also created `CurrentUser` as a small helper for retrieving the logged-in user and `UserResponse` so sensitive fields like the access token are not returned to the frontend."

---

# 23. Presentation Explanation — File-by-File

If someone asks:

### "Why do you have `SecurityConfig`?"

> "It is the security entry point. It defines authorization rules, OAuth2 login, session behaviour, success/failure handling and logout."

### "Why a custom `GitHubOAuth2UserService`?"

> "Spring can load the GitHub user, but DevPilot needs to map that identity into its own database user. The custom service is the bridge between GitHub's OAuth2 user and DevPilot's user model."

### "Why `UserService`?"

> "I keep user business logic and persistence orchestration separate from OAuth2-specific code."

### "Why `UserRepository`?"

> "Spring Data JPA generates the database interaction. I only need to declare `findByGithubId()`."

### "Why `AppUserPrincipal`?"

> "Spring Security needs an authenticated principal. This class wraps our DevPilot user while still implementing Spring Security's OAuth2 user contract."

### "Why `CurrentUser`?"

> "It avoids repeating `SecurityContextHolder` logic across controllers and services."

### "Why `UserResponse`?"

> "It is a DTO used to control exactly what user information leaves the backend."

### "Why encrypt the token?"

> "The GitHub access token is sensitive. The implementation encrypts it before database storage so the raw token is not persisted as plain text."

---

# 24. Terms You Must Know

## OAuth2

An authorization framework that allows an application to obtain limited access to another service without collecting that service's password.

For DevPilot:

```text
DevPilot ←→ GitHub
```

---

## OAuth2 Client

DevPilot is the OAuth2 client because it asks GitHub for authorization and receives tokens/user information.

---

## Authorization Code Flow

High-level sequence:

```text
User
 ↓
DevPilot
 ↓
GitHub
 ↓
User authorizes
 ↓
GitHub callback
 ↓
DevPilot exchanges/handles authorization
 ↓
Authenticated session
```

---

## Access Token

A credential representing the permissions granted by GitHub.

DevPilot stores it encrypted.

---

## Scope

Defines what the token is allowed to access.

Current configuration requests:

```text
read:user
repo
```

---

## Principal

The object representing the currently authenticated identity.

DevPilot uses:

```text
AppUserPrincipal
```

---

## Authentication vs Authorization

### Authentication

> "Who are you?"

GitHub login answers this.

### Authorization

> "What are you allowed to do?"

Spring Security roles and permissions handle this.

Current principal receives:

```text
ROLE_USER
```

---

## Session

The browser receives a session cookie:

```text
DEVPILOT_SESSION
```

The backend uses the session to remember that the user has authenticated.

---

# 25. Problems We Encountered During Implementation

## Problem 1 — Bean methods were accidentally inside the security method

Incorrect:

```java
return http.build();

@Bean
AuthenticationSuccessHandler ...
```

### Why it failed

Java methods cannot be declared inside another method.

### Fix

Move the bean methods outside:

```text
SecurityConfig
├── securityFilterChain()
├── oauth2SuccessHandler()
└── oauth2FailureHandler()
```

---

## Problem 2 — Wrong `@Value` import

Incorrect:

```java
import lombok.Value;
```

Correct:

```java
import org.springframework.beans.factory.annotation.Value;
```

### Why it matters

These are two completely different annotations.

The Spring annotation reads configuration values such as:

```text
${app.frontend-url}
```

---

## Problem 3 — OAuth2 handlers were nested at the wrong level

Incorrect:

```java
.userInfoEndpoint(userInfo -> userInfo
    .userService(...)
    .successHandler(...)
    .failureHandler(...)
)
```

Correct:

```java
.oauth2Login(oauth -> oauth
    .userInfoEndpoint(userInfo -> userInfo
        .userService(...)
    )
    .successHandler(...)
    .failureHandler(...)
)
```

---

## Problem 4 — `tokenScope` vs `tokenScopes`

The entity contains:

```java
private String tokenScope;
```

Therefore Lombok generates:

```java
setTokenScope(...)
```

The service originally called:

```java
setTokenScopes(...)
```

The correct call is:

```java
user.setTokenScope(scopes);
```

---

# 26. Security Things to Fix Before Production

These are important because the current implementation is still development-oriented.

## 1. Never commit GitHub credentials

Use:

```properties
spring.security.oauth2.client.registration.github.client-id=${GITHUB_CLIENT_ID}
spring.security.oauth2.client.registration.github.client-secret=${GITHUB_CLIENT_SECRET}
```

and set the values through environment variables.

If a real secret has already been pushed, rotate it.

---

## 2. Replace development encryption defaults

The current local defaults are convenient for development.

Production should use strong secret values supplied through the environment or a secret manager.

---

## 3. Be careful with OAuth scopes

The current project asks for:

```text
read:user
repo
```

The `repo` scope can provide broad repository access.

Only request permissions DevPilot actually needs.

---

## 4. Review the CSRF strategy before production

The current configuration disables CSRF:

```java
.csrf(csrf -> csrf.disable())
```

That can be acceptable for some architectures, but because DevPilot is using browser sessions, the production CSRF strategy should be reviewed rather than copied blindly.

---

## 5. Consider stronger production token/secret storage

The current design encrypts the token before database storage.

For production, also consider:

```text
Secret manager
KMS
Vault
Cloud secret management
```

depending on deployment architecture.

---

# 27. Interview Questions You Should Be Able to Answer

### Q1. Why use GitHub OAuth2 instead of storing passwords?

Because DevPilot can rely on GitHub to authenticate the user instead of creating and protecting its own password system.

### Q2. What does `GitHubOAuth2UserService` do?

It loads GitHub's OAuth2 user, extracts token/scope information, sends the user data to `UserService`, and returns a DevPilot-specific security principal.

### Q3. Why do you have both `id` and `githubId`?

`id` is DevPilot's internal primary key. `githubId` identifies the same user at GitHub.

### Q4. How do you know whether a GitHub user is already registered?

`UserRepository.findByGithubId(githubId)`.

### Q5. What happens when they log in again?

The existing user is found and updated instead of creating a duplicate.

### Q6. Where is the OAuth token stored?

In the `User` entity's `accessToken` field, but encrypted before persistence.

### Q7. How does Spring know who the current user is?

Spring Security stores the authentication, whose principal is `AppUserPrincipal`.

### Q8. How does `/api/auth/me` get the logged-in user?

`CurrentUser.require()` obtains the authenticated principal from `SecurityContextHolder`.

### Q9. Why not return the `User` entity directly?

Because the entity contains server-side/sensitive fields such as the OAuth access token.

### Q10. What happens if a user accesses `/api/**` without logging in?

`SecurityConfig` marks `/api/**` as authenticated, so Spring returns HTTP 401 through `HttpStatusEntryPoint`.

---

# 28. Mental Model to Remember

You do **not** need to memorize every Spring class.

Remember the responsibility chain:

```text
SecurityConfig
      ↓
Starts / controls authentication
      ↓
GitHubOAuth2UserService
      ↓
Understands GitHub OAuth2 data
      ↓
UserService
      ↓
Creates/updates DevPilot user
      ↓
UserRepository
      ↓
PostgreSQL
      ↓
AppUserPrincipal
      ↓
Spring Security knows who is logged in
      ↓
CurrentUser
      ↓
Controllers can access the logged-in user
```

That is the architecture.

---

# 29. Final Presentation Summary

### Problem

DevPilot needs a secure way to identify users and connect those users to their GitHub accounts.

### Solution

Use:

```text
Spring Security
+
GitHub OAuth2
+
JPA/PostgreSQL
+
Session authentication
```

### Main implementation

```text
1. Configure GitHub OAuth2
2. Configure Spring Security
3. Receive GitHub user
4. Find/create DevPilot user
5. Encrypt access token
6. Save user
7. Create AppUserPrincipal
8. Store authentication in session
9. Redirect to frontend
10. Protect /api endpoints
```

### Result

DevPilot can now establish a GitHub-authenticated user identity and use that identity throughout the backend.

---

# 30. Quick Cheat Sheet

```text
SecurityConfig
→ Security rules + OAuth2 + session + logout

GitHubOAuth2UserService
→ GitHub user → DevPilot user

UserService
→ Business logic + upsert + token encryption

UserRepository
→ Database access

User
→ Database model

AppUserPrincipal
→ Authenticated DevPilot identity

CurrentUser
→ Get current logged-in user

AuthController
→ Authentication API endpoints

UserResponse
→ Safe API response

CryptoConfig
→ Token encryption bean

CorsConfig
→ Frontend/backend cross-origin communication

application.properties
→ Runtime configuration

pom.xml
→ Security/OAuth2 dependencies
```

---

# 31. One Sentence to Memorize

> **"GitHub authenticates the user, Spring Security manages the authentication flow and session, `GitHubOAuth2UserService` maps the GitHub identity into DevPilot, `UserService` persists the encrypted user data, and `AppUserPrincipal` becomes the authenticated identity used by the rest of the application."**

