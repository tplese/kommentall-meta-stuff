# GitHub Integration Implementation Plan for TestAI (Kommentall)

**Project**: TestAI (fe_kommentall + be_kommentall)  
**Target**: Production-ready GitHub OAuth integration similar to ChatGPT/Claude  
**Date**: November 2025

---

## Executive Summary

This plan implements full GitHub repository integration with OAuth authentication, allowing users to:
- Read files from repositories
- Create/modify files in repositories
- Search across repositories
- Clone/download repositories
- Commit changes to separate branches (e.g., `kommentall/feature-name`)

**Key Architecture Decisions**:
- GitHub OAuth (not GitHub App) - simpler for MVP
- Backend-only token storage in PostgreSQL (secure)
- Device-specific sessions (no user auth yet)
- In-app WebView OAuth flow with deep linking
- Spring Security for API protection
- Rate limiting for GitHub API calls

---

## PHASE 1: Foundation & OAuth Setup

**Goal**: Establish OAuth flow, token storage, and basic GitHub connection.

**Estimated Time**: 1 conversation session

### 1.1 GitHub OAuth App Setup (Manual - Outside Code)

**Steps to perform on GitHub.com**:

1. Go to GitHub Settings → Developer settings → OAuth Apps → New OAuth App
2. Configure:
   ```
   Application name: TestAI
   Homepage URL: http://your-domain.com (or http://localhost:8080 for dev)
   Authorization callback URL: http://192.168.68.57:8080/api/github/oauth/callback
   ```
3. Note down:
   - **Client ID**: `xxxxxxxxxxxxx`
   - **Client Secret**: `yyyyyyyyyyyy`
4. Requested scopes:
   - `repo` (full control of private repos)
   - `read:user` (read user profile)
   - `read:org` (read org membership)

### 1.2 Backend: Database Schema

**File**: Create new migration or update `schema.sql`

**Tables to create**:

```sql
-- GitHub connections per device
CREATE TABLE github_connections (
    id BIGSERIAL PRIMARY KEY,
    device_id VARCHAR(255) UNIQUE NOT NULL,  -- Device-specific identifier
    access_token TEXT NOT NULL,               -- GitHub OAuth token
    token_type VARCHAR(50) NOT NULL,          -- Usually "Bearer"
    scope TEXT,                               -- Granted scopes
    username VARCHAR(255),                    -- GitHub username
    avatar_url TEXT,                          -- User's GitHub avatar
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_used_at TIMESTAMP,
    is_active BOOLEAN DEFAULT true
);

-- Selected repositories per device
CREATE TABLE github_repositories (
    id BIGSERIAL PRIMARY KEY,
    device_id VARCHAR(255) NOT NULL,
    repo_full_name VARCHAR(512) NOT NULL,    -- e.g., "owner/repo-name"
    repo_id BIGINT NOT NULL,                 -- GitHub's repo ID
    is_private BOOLEAN DEFAULT false,
    default_branch VARCHAR(255) DEFAULT 'main',
    selected_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (device_id) REFERENCES github_connections(device_id) ON DELETE CASCADE,
    UNIQUE(device_id, repo_full_name)
);

-- Rate limiting tracking
CREATE TABLE github_rate_limits (
    id BIGSERIAL PRIMARY KEY,
    device_id VARCHAR(255) NOT NULL,
    endpoint VARCHAR(255) NOT NULL,          -- Which API endpoint
    request_count INTEGER DEFAULT 0,
    window_start TIMESTAMP NOT NULL,
    window_end TIMESTAMP NOT NULL,
    FOREIGN KEY (device_id) REFERENCES github_connections(device_id) ON DELETE CASCADE
);

-- Create indexes
CREATE INDEX idx_github_connections_device ON github_connections(device_id);
CREATE INDEX idx_github_repos_device ON github_repositories(device_id);
CREATE INDEX idx_rate_limits_device_endpoint ON github_rate_limits(device_id, endpoint);
```

### 1.3 Backend: Maven Dependencies

**File**: `pom.xml`

**Add these dependencies**:

```xml
<!-- GitHub API Client -->
<dependency>
    <groupId>org.kohsuke</groupId>
    <artifactId>github-api</artifactId>
    <version>1.321</version>
</dependency>

<!-- Spring Security (for API protection) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT for device session tokens -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>

<!-- HTTP Client (for OAuth) -->
<!-- Already included in Spring Boot, but explicitly declare -->
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
</dependency>
```

### 1.4 Backend: Configuration

**File**: `src/main/resources/application.yaml`

**Add GitHub configuration**:

```yaml
github:
  oauth:
    client-id: ${GITHUB_CLIENT_ID}
    client-secret: ${GITHUB_CLIENT_SECRET}
    authorization-uri: https://github.com/login/oauth/authorize
    token-uri: https://github.com/login/oauth/access_token
    user-info-uri: https://api.github.com/user
    redirect-uri: http://192.168.68.57:8080/api/github/oauth/callback
    scopes:
      - repo
      - read:user
      - read:org
  api:
    base-url: https://api.github.com
    rate-limit:
      max-requests-per-hour: 4500  # Leave buffer (GitHub allows 5000)
      
# Device session configuration
device:
  session:
    jwt-secret: ${DEVICE_SESSION_SECRET:your-secret-key-min-256-bits-change-in-production}
    expiration-hours: 720  # 30 days
```

**Environment Variables** (add to your system or Docker):
```bash
GITHUB_CLIENT_ID=your_github_client_id
GITHUB_CLIENT_SECRET=your_github_client_secret
DEVICE_SESSION_SECRET=generate-secure-random-string-here
```

### 1.5 Backend: Entity Classes

**File**: `src/main/java/com/kommentall/model/GitHubConnection.java`

```java
package com.kommentall.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Entity
@Table(name = "github_connections")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class GitHubConnection {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String deviceId;
    
    @Column(nullable = false, columnDefinition = "TEXT")
    private String accessToken;
    
    @Column(nullable = false, length = 50)
    private String tokenType = "Bearer";
    
    @Column(columnDefinition = "TEXT")
    private String scope;
    
    private String username;
    
    @Column(columnDefinition = "TEXT")
    private String avatarUrl;
    
    private LocalDateTime createdAt = LocalDateTime.now();
    
    private LocalDateTime updatedAt = LocalDateTime.now();
    
    private LocalDateTime lastUsedAt;
    
    private Boolean isActive = true;
}
```

**File**: `src/main/java/com/kommentall/model/GitHubRepository.java`

```java
package com.kommentall.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Entity
@Table(name = "github_repositories")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class GitHubRepository {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String deviceId;
    
    @Column(nullable = false, length = 512)
    private String repoFullName;
    
    @Column(nullable = false)
    private Long repoId;
    
    private Boolean isPrivate = false;
    
    private String defaultBranch = "main";
    
    private LocalDateTime selectedAt = LocalDateTime.now();
}
```

**File**: `src/main/java/com/kommentall/model/GitHubRateLimit.java`

```java
package com.kommentall.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Entity
@Table(name = "github_rate_limits")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class GitHubRateLimit {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String deviceId;
    
    @Column(nullable = false)
    private String endpoint;
    
    private Integer requestCount = 0;
    
    @Column(nullable = false)
    private LocalDateTime windowStart;
    
    @Column(nullable = false)
    private LocalDateTime windowEnd;
}
```

### 1.6 Backend: Repository Interfaces

**File**: `src/main/java/com/kommentall/repository/GitHubConnectionRepository.java`

```java
package com.kommentall.repository;

import com.kommentall.model.GitHubConnection;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface GitHubConnectionRepository extends JpaRepository<GitHubConnection, Long> {
    Optional<GitHubConnection> findByDeviceId(String deviceId);
    boolean existsByDeviceId(String deviceId);
    void deleteByDeviceId(String deviceId);
}
```

**File**: `src/main/java/com/kommentall/repository/GitHubRepositoryRepository.java`

```java
package com.kommentall.repository;

import com.kommentall.model.GitHubRepository;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface GitHubRepositoryRepository extends JpaRepository<GitHubRepository, Long> {
    List<GitHubRepository> findByDeviceId(String deviceId);
    void deleteByDeviceIdAndRepoFullName(String deviceId, String repoFullName);
}
```

**File**: `src/main/java/com/kommentall/repository/GitHubRateLimitRepository.java`

```java
package com.kommentall.repository;

import com.kommentall.model.GitHubRateLimit;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.Optional;

@Repository
public interface GitHubRateLimitRepository extends JpaRepository<GitHubRateLimit, Long> {
    Optional<GitHubRateLimit> findByDeviceIdAndEndpointAndWindowEndAfter(
        String deviceId, 
        String endpoint, 
        LocalDateTime now
    );
}
```

### 1.7 Backend: Configuration Properties

**File**: `src/main/java/com/kommentall/config/GitHubProperties.java`

```java
package com.kommentall.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

import java.util.List;

@Component
@ConfigurationProperties(prefix = "github")
@Data
public class GitHubProperties {
    private OAuth oauth = new OAuth();
    private Api api = new Api();
    
    @Data
    public static class OAuth {
        private String clientId;
        private String clientSecret;
        private String authorizationUri;
        private String tokenUri;
        private String userInfoUri;
        private String redirectUri;
        private List<String> scopes;
    }
    
    @Data
    public static class Api {
        private String baseUrl;
        private RateLimit rateLimit = new RateLimit();
        
        @Data
        public static class RateLimit {
            private int maxRequestsPerHour;
        }
    }
}
```

**File**: `src/main/java/com/kommentall/config/DeviceSessionProperties.java`

```java
package com.kommentall.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "device.session")
@Data
public class DeviceSessionProperties {
    private String jwtSecret;
    private int expirationHours;
}
```

### 1.8 Backend: Security Configuration

**File**: `src/main/java/com/kommentall/config/SecurityConfig.java`

```java
package com.kommentall.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.Arrays;
import java.util.List;

@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())  // Disable for now (enable with proper token later)
            .sessionManagement(session -> 
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers(
                    "/api/github/oauth/**",
                    "/api/github/device/register",
                    "/error"
                ).permitAll()
                // All other GitHub endpoints require device session
                .requestMatchers("/api/github/**").authenticated()
                // Existing endpoints remain open for now
                .anyRequest().permitAll()
            );
        
        return http.build();
    }
    
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("*"));  // TODO: Restrict in production
        configuration.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(List.of("*"));
        configuration.setExposedHeaders(List.of("Authorization", "X-Device-Token"));
        
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }
}
```

---

## PHASE 2: OAuth Flow Implementation

**Goal**: Complete OAuth flow from Flutter → Backend → GitHub → Backend → Flutter

**Estimated Time**: 1 conversation session

### 2.1 Backend: DTOs

**File**: `src/main/java/com/kommentall/dto/GitHubOAuthRequest.java`

```java
package com.kommentall.dto;

import lombok.Data;

@Data
public class GitHubOAuthRequest {
    private String deviceId;
}
```

**File**: `src/main/java/com/kommentall/dto/GitHubOAuthResponse.java`

```java
package com.kommentall.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class GitHubOAuthResponse {
    private String authorizationUrl;
    private String state;  // CSRF protection
}
```

**File**: `src/main/java/com/kommentall/dto/GitHubConnectionStatus.java`

```java
package com.kommentall.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class GitHubConnectionStatus {
    private boolean isConnected;
    private String username;
    private String avatarUrl;
    private String connectedAt;
}
```

**File**: `src/main/java/com/kommentall/dto/DeviceRegistrationResponse.java`

```java
package com.kommentall.dto;

import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class DeviceRegistrationResponse {
    private String deviceId;
    private String deviceToken;  // JWT for API authentication
}
```

### 2.2 Backend: Device Session Service

**File**: `src/main/java/com/kommentall/service/DeviceSessionService.java`

```java
package com.kommentall.service;

import com.kommentall.config.DeviceSessionProperties;
import io.jsonwebtoken.Claims;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Date;
import java.util.UUID;

@Service
public class DeviceSessionService {
    
    private final DeviceSessionProperties properties;
    private final SecretKey signingKey;
    
    @Autowired
    public DeviceSessionService(DeviceSessionProperties properties) {
        this.properties = properties;
        // Create signing key from secret
        this.signingKey = Keys.hmacShaKeyFor(
            properties.getJwtSecret().getBytes(StandardCharsets.UTF_8)
        );
    }
    
    /**
     * Generate a new device ID
     */
    public String generateDeviceId() {
        return UUID.randomUUID().toString();
    }
    
    /**
     * Generate JWT token for device session
     */
    public String generateDeviceToken(String deviceId) {
        Instant now = Instant.now();
        Instant expiration = now.plus(properties.getExpirationHours(), ChronoUnit.HOURS);
        
        return Jwts.builder()
            .setSubject(deviceId)
            .setIssuedAt(Date.from(now))
            .setExpiration(Date.from(expiration))
            .claim("type", "device-session")
            .signWith(signingKey, SignatureAlgorithm.HS256)
            .compact();
    }
    
    /**
     * Validate and extract device ID from token
     */
    public String validateAndExtractDeviceId(String token) {
        try {
            Claims claims = Jwts.parserBuilder()
                .setSigningKey(signingKey)
                .build()
                .parseClaimsJws(token)
                .getBody();
            
            return claims.getSubject();
        } catch (Exception e) {
            throw new RuntimeException("Invalid device token", e);
        }
    }
    
    /**
     * Check if token is valid
     */
    public boolean isValidToken(String token) {
        try {
            Jwts.parserBuilder()
                .setSigningKey(signingKey)
                .build()
                .parseClaimsJws(token);
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}
```

### 2.3 Backend: GitHub OAuth Service

**File**: `src/main/java/com/kommentall/service/GitHubOAuthService.java`

```java
package com.kommentall.service;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.kommentall.config.GitHubProperties;
import com.kommentall.model.GitHubConnection;
import com.kommentall.repository.GitHubConnectionRepository;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.IOException;
import java.net.URI;
import java.net.URLEncoder;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.time.LocalDateTime;
import java.util.UUID;

@Service
public class GitHubOAuthService {
    
    private static final Logger log = LoggerFactory.getLogger(GitHubOAuthService.class);
    
    private final GitHubProperties properties;
    private final GitHubConnectionRepository connectionRepository;
    private final HttpClient httpClient;
    private final ObjectMapper objectMapper;
    
    @Autowired
    public GitHubOAuthService(
            GitHubProperties properties,
            GitHubConnectionRepository connectionRepository) {
        this.properties = properties;
        this.connectionRepository = connectionRepository;
        this.httpClient = HttpClient.newHttpClient();
        this.objectMapper = new ObjectMapper();
    }
    
    /**
     * Generate GitHub authorization URL with CSRF state
     */
    public String generateAuthorizationUrl(String deviceId) {
        String state = generateState(deviceId);
        String scopes = String.join(" ", properties.getOauth().getScopes());
        
        return properties.getOauth().getAuthorizationUri()
            + "?client_id=" + properties.getOauth().getClientId()
            + "&redirect_uri=" + URLEncoder.encode(
                properties.getOauth().getRedirectUri(), 
                StandardCharsets.UTF_8)
            + "&scope=" + URLEncoder.encode(scopes, StandardCharsets.UTF_8)
            + "&state=" + state;
    }
    
    /**
     * Generate CSRF state token
     */
    private String generateState(String deviceId) {
        return UUID.randomUUID().toString() + ":" + deviceId;
    }
    
    /**
     * Validate state and extract device ID
     */
    public String validateStateAndExtractDeviceId(String state) {
        if (state == null || !state.contains(":")) {
            throw new IllegalArgumentException("Invalid state parameter");
        }
        return state.split(":")[1];
    }
    
    /**
     * Exchange authorization code for access token
     */
    @Transactional
    public GitHubConnection exchangeCodeForToken(String code, String deviceId) 
            throws IOException, InterruptedException {
        
        log.info("Exchanging code for token - deviceId: {}", deviceId);
        
        // Exchange code for token
        String accessToken = requestAccessToken(code);
        
        // Get user info
        JsonNode userInfo = getUserInfo(accessToken);
        
        // Save or update connection
        GitHubConnection connection = connectionRepository
            .findByDeviceId(deviceId)
            .orElse(new GitHubConnection());
        
        connection.setDeviceId(deviceId);
        connection.setAccessToken(accessToken);
        connection.setTokenType("Bearer");
        connection.setScope(String.join(",", properties.getOauth().getScopes()));
        connection.setUsername(userInfo.get("login").asText());
        connection.setAvatarUrl(userInfo.get("avatar_url").asText());
        connection.setUpdatedAt(LocalDateTime.now());
        connection.setLastUsedAt(LocalDateTime.now());
        connection.setIsActive(true);
        
        if (connection.getId() == null) {
            connection.setCreatedAt(LocalDateTime.now());
        }
        
        return connectionRepository.save(connection);
    }
    
    /**
     * Request access token from GitHub
     */
    private String requestAccessToken(String code) 
            throws IOException, InterruptedException {
        
        String requestBody = "client_id=" + properties.getOauth().getClientId()
            + "&client_secret=" + properties.getOauth().getClientSecret()
            + "&code=" + code;
        
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(properties.getOauth().getTokenUri()))
            .header("Accept", "application/json")
            .header("Content-Type", "application/x-www-form-urlencoded")
            .POST(HttpRequest.BodyPublishers.ofString(requestBody))
            .build();
        
        HttpResponse<String> response = httpClient.send(request, 
            HttpResponse.BodyHandlers.ofString());
        
        if (response.statusCode() != 200) {
            log.error("Failed to exchange code for token: {}", response.body());
            throw new IOException("Failed to exchange code for token: " 
                + response.statusCode());
        }
        
        JsonNode json = objectMapper.readTree(response.body());
        
        if (json.has("error")) {
            throw new IOException("GitHub OAuth error: " 
                + json.get("error_description").asText());
        }
        
        return json.get("access_token").asText();
    }
    
    /**
     * Get user info from GitHub API
     */
    private JsonNode getUserInfo(String accessToken) 
            throws IOException, InterruptedException {
        
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create(properties.getOauth().getUserInfoUri()))
            .header("Authorization", "Bearer " + accessToken)
            .header("Accept", "application/vnd.github+json")
            .GET()
            .build();
        
        HttpResponse<String> response = httpClient.send(request,
            HttpResponse.BodyHandlers.ofString());
        
        if (response.statusCode() != 200) {
            throw new IOException("Failed to get user info: " 
                + response.statusCode());
        }
        
        return objectMapper.readTree(response.body());
    }
    
    /**
     * Get connection status for device
     */
    public GitHubConnection getConnection(String deviceId) {
        return connectionRepository.findByDeviceId(deviceId)
            .orElse(null);
    }
    
    /**
     * Disconnect GitHub for device
     */
    @Transactional
    public void disconnect(String deviceId) {
        connectionRepository.deleteByDeviceId(deviceId);
    }
}
```

### 2.4 Backend: OAuth Controller

**File**: `src/main/java/com/kommentall/controller/GitHubOAuthController.java`

```java
package com.kommentall.controller;

import com.kommentall.dto.DeviceRegistrationResponse;
import com.kommentall.dto.GitHubConnectionStatus;
import com.kommentall.dto.GitHubOAuthRequest;
import com.kommentall.dto.GitHubOAuthResponse;
import com.kommentall.model.GitHubConnection;
import com.kommentall.service.DeviceSessionService;
import com.kommentall.service.GitHubOAuthService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestController
@RequestMapping("/api/github")
public class GitHubOAuthController {
    
    private static final Logger log = LoggerFactory.getLogger(GitHubOAuthController.class);
    
    private final GitHubOAuthService oauthService;
    private final DeviceSessionService sessionService;
    
    @Autowired
    public GitHubOAuthController(
            GitHubOAuthService oauthService,
            DeviceSessionService sessionService) {
        this.oauthService = oauthService;
        this.sessionService = sessionService;
    }
    
    /**
     * Register a new device and get device token
     */
    @PostMapping("/device/register")
    public ResponseEntity<DeviceRegistrationResponse> registerDevice() {
        try {
            String deviceId = sessionService.generateDeviceId();
            String deviceToken = sessionService.generateDeviceToken(deviceId);
            
            log.info("Registered new device: {}", deviceId);
            
            return ResponseEntity.ok(
                new DeviceRegistrationResponse(deviceId, deviceToken)
            );
        } catch (Exception e) {
            log.error("Error registering device", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .build();
        }
    }
    
    /**
     * Step 1: Get authorization URL to redirect user to GitHub
     */
    @PostMapping("/oauth/authorize")
    public ResponseEntity<GitHubOAuthResponse> getAuthorizationUrl(
            @RequestBody GitHubOAuthRequest request) {
        try {
            String authUrl = oauthService.generateAuthorizationUrl(request.getDeviceId());
            String state = authUrl.substring(authUrl.indexOf("state=") + 6);
            
            return ResponseEntity.ok(
                new GitHubOAuthResponse(authUrl, state)
            );
        } catch (Exception e) {
            log.error("Error generating authorization URL", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .build();
        }
    }
    
    /**
     * Step 2: OAuth callback from GitHub
     * This endpoint is called by GitHub after user authorizes
     */
    @GetMapping("/oauth/callback")
    public ResponseEntity<?> handleCallback(
            @RequestParam String code,
            @RequestParam String state) {
        
        try {
            // Validate state and extract device ID
            String deviceId = oauthService.validateStateAndExtractDeviceId(state);
            
            // Exchange code for token
            GitHubConnection connection = oauthService.exchangeCodeForToken(code, deviceId);
            
            log.info("Successfully connected GitHub for device: {}", deviceId);
            
            // Return HTML that closes WebView and notifies Flutter
            String html = """
                <!DOCTYPE html>
                <html>
                <head>
                    <title>GitHub Connected</title>
                    <style>
                        body {
                            font-family: Arial, sans-serif;
                            display: flex;
                            justify-content: center;
                            align-items: center;
                            height: 100vh;
                            margin: 0;
                            background-color: #0d1117;
                            color: #c9d1d9;
                        }
                        .container {
                            text-align: center;
                            padding: 40px;
                        }
                        .success-icon {
                            font-size: 64px;
                            color: #3fb950;
                            margin-bottom: 20px;
                        }
                        h1 {
                            color: #3fb950;
                            margin-bottom: 10px;
                        }
                        p {
                            color: #8b949e;
                            margin-bottom: 20px;
                        }
                    </style>
                    <script>
                        // Signal to Flutter that OAuth is complete
                        if (window.flutter_inappwebview) {
                            window.flutter_inappwebview.callHandler('oauthSuccess', {
                                username: '%s',
                                avatarUrl: '%s'
                            });
                        }
                        // Auto-close after 2 seconds
                        setTimeout(function() {
                            window.close();
                        }, 2000);
                    </script>
                </head>
                <body>
                    <div class="container">
                        <div class="success-icon">✓</div>
                        <h1>Successfully Connected!</h1>
                        <p>Your GitHub account has been connected.</p>
                        <p>You can close this window.</p>
                    </div>
                </body>
                </html>
                """.formatted(
                    connection.getUsername(),
                    connection.getAvatarUrl()
                );
            
            return ResponseEntity.ok()
                .header("Content-Type", "text/html")
                .body(html);
                
        } catch (Exception e) {
            log.error("Error in OAuth callback", e);
            
            String errorHtml = """
                <!DOCTYPE html>
                <html>
                <head>
                    <title>Connection Failed</title>
                    <style>
                        body {
                            font-family: Arial, sans-serif;
                            display: flex;
                            justify-content: center;
                            align-items: center;
                            height: 100vh;
                            margin: 0;
                            background-color: #0d1117;
                            color: #c9d1d9;
                        }
                        .container {
                            text-align: center;
                            padding: 40px;
                        }
                        .error-icon {
                            font-size: 64px;
                            color: #f85149;
                            margin-bottom: 20px;
                        }
                        h1 {
                            color: #f85149;
                            margin-bottom: 10px;
                        }
                        p {
                            color: #8b949e;
                        }
                    </style>
                    <script>
                        if (window.flutter_inappwebview) {
                            window.flutter_inappwebview.callHandler('oauthError', {
                                error: '%s'
                            });
                        }
                        setTimeout(function() {
                            window.close();
                        }, 3000);
                    </script>
                </head>
                <body>
                    <div class="container">
                        <div class="error-icon">✗</div>
                        <h1>Connection Failed</h1>
                        <p>%s</p>
                        <p>Please try again.</p>
                    </div>
                </body>
                </html>
                """.formatted(
                    e.getMessage(),
                    e.getMessage()
                );
            
            return ResponseEntity.ok()
                .header("Content-Type", "text/html")
                .body(errorHtml);
        }
    }
    
    /**
     * Get connection status
     */
    @GetMapping("/connection/status")
    public ResponseEntity<GitHubConnectionStatus> getConnectionStatus(
            @RequestHeader("X-Device-Token") String deviceToken) {
        try {
            String deviceId = sessionService.validateAndExtractDeviceId(deviceToken);
            GitHubConnection connection = oauthService.getConnection(deviceId);
            
            if (connection == null || !connection.getIsActive()) {
                return ResponseEntity.ok(
                    new GitHubConnectionStatus(false, null, null, null)
                );
            }
            
            return ResponseEntity.ok(new GitHubConnectionStatus(
                true,
                connection.getUsername(),
                connection.getAvatarUrl(),
                connection.getCreatedAt().toString()
            ));
            
        } catch (Exception e) {
            log.error("Error getting connection status", e);
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .build();
        }
    }
    
    /**
     * Disconnect GitHub
     */
    @DeleteMapping("/connection")
    public ResponseEntity<?> disconnect(
            @RequestHeader("X-Device-Token") String deviceToken) {
        try {
            String deviceId = sessionService.validateAndExtractDeviceId(deviceToken);
            oauthService.disconnect(deviceId);
            
            return ResponseEntity.ok(Map.of("message", "Disconnected successfully"));
            
        } catch (Exception e) {
            log.error("Error disconnecting GitHub", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", e.getMessage()));
        }
    }
}
```

### 2.5 Frontend: Dependencies

**File**: `pubspec.yaml`

**Add these dependencies**:

```yaml
dependencies:
  # Existing dependencies...
  
  # WebView for OAuth
  flutter_inappwebview: ^6.0.0
  
  # Secure storage for device token
  flutter_secure_storage: ^9.0.0
  
  # HTTP client (if not already present)
  http: ^1.1.0
  
  # URL launcher (already present)
  url_launcher: ^6.2.1
```

### 2.6 Frontend: Device Session Service

**File**: `lib/services/device_session_service.dart`

```dart
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:http/http.dart' as http;
import 'dart:convert';
import '../config/url_constants.dart';

/// Manages device registration and session token
class DeviceSessionService {
  static const String _deviceIdKey = 'device_id';
  static const String _deviceTokenKey = 'device_token';
  
  final FlutterSecureStorage _storage = const FlutterSecureStorage();
  
  String? _cachedDeviceId;
  String? _cachedDeviceToken;
  
  /// Get or create device ID and token
  Future<Map<String, String>> getOrCreateSession() async {
    // Try to get from cache
    if (_cachedDeviceId != null && _cachedDeviceToken != null) {
      return {
        'deviceId': _cachedDeviceId!,
        'deviceToken': _cachedDeviceToken!,
      };
    }
    
    // Try to get from storage
    String? deviceId = await _storage.read(key: _deviceIdKey);
    String? deviceToken = await _storage.read(key: _deviceTokenKey);
    
    if (deviceId != null && deviceToken != null) {
      _cachedDeviceId = deviceId;
      _cachedDeviceToken = deviceToken;
      return {
        'deviceId': deviceId,
        'deviceToken': deviceToken,
      };
    }
    
    // Register new device
    return await _registerDevice();
  }
  
  /// Register a new device with backend
  Future<Map<String, String>> _registerDevice() async {
    try {
      final response = await http.post(
        Uri.parse('$baseUrl/api/github/device/register'),
        headers: {'Content-Type': 'application/json'},
      );
      
      if (response.statusCode == 200) {
        final data = json.decode(response.body);
        final deviceId = data['deviceId'] as String;
        final deviceToken = data['deviceToken'] as String;
        
        // Store in secure storage
        await _storage.write(key: _deviceIdKey, value: deviceId);
        await _storage.write(key: _deviceTokenKey, value: deviceToken);
        
        // Cache
        _cachedDeviceId = deviceId;
        _cachedDeviceToken = deviceToken;
        
        return {
          'deviceId': deviceId,
          'deviceToken': deviceToken,
        };
      } else {
        throw Exception('Failed to register device: ${response.statusCode}');
      }
    } catch (e) {
      throw Exception('Error registering device: $e');
    }
  }
  
  /// Get device token for API calls
  Future<String?> getDeviceToken() async {
    final session = await getOrCreateSession();
    return session['deviceToken'];
  }
  
  /// Clear session (logout)
  Future<void> clearSession() async {
    await _storage.delete(key: _deviceIdKey);
    await _storage.delete(key: _deviceTokenKey);
    _cachedDeviceId = null;
    _cachedDeviceToken = null;
  }
}
```

### 2.7 Frontend: GitHub Service

**File**: `lib/services/github_service.dart`

```dart
import 'package:http/http.dart' as http;
import 'dart:convert';
import '../config/url_constants.dart';
import 'device_session_service.dart';

/// Service for GitHub integration
class GitHubService {
  final DeviceSessionService _sessionService;
  
  GitHubService(this._sessionService);
  
  /// Get GitHub authorization URL
  Future<Map<String, dynamic>> getAuthorizationUrl() async {
    try {
      final session = await _sessionService.getOrCreateSession();
      final deviceId = session['deviceId']!;
      
      final response = await http.post(
        Uri.parse('$baseUrl/api/github/oauth/authorize'),
        headers: {'Content-Type': 'application/json'},
        body: json.encode({'deviceId': deviceId}),
      );
      
      if (response.statusCode == 200) {
        return json.decode(response.body);
      } else {
        throw Exception('Failed to get authorization URL: ${response.statusCode}');
      }
    } catch (e) {
      throw Exception('Error getting authorization URL: $e');
    }
  }
  
  /// Get GitHub connection status
  Future<Map<String, dynamic>> getConnectionStatus() async {
    try {
      final deviceToken = await _sessionService.getDeviceToken();
      
      final response = await http.get(
        Uri.parse('$baseUrl/api/github/connection/status'),
        headers: {
          'Content-Type': 'application/json',
          'X-Device-Token': deviceToken!,
        },
      );
      
      if (response.statusCode == 200) {
        return json.decode(response.body);
      } else {
        throw Exception('Failed to get connection status: ${response.statusCode}');
      }
    } catch (e) {
      throw Exception('Error getting connection status: $e');
    }
  }
  
  /// Disconnect GitHub
  Future<void> disconnect() async {
    try {
      final deviceToken = await _sessionService.getDeviceToken();
      
      final response = await http.delete(
        Uri.parse('$baseUrl/api/github/connection'),
        headers: {
          'Content-Type': 'application/json',
          'X-Device-Token': deviceToken!,
        },
      );
      
      if (response.statusCode != 200) {
        throw Exception('Failed to disconnect: ${response.statusCode}');
      }
    } catch (e) {
      throw Exception('Error disconnecting GitHub: $e');
    }
  }
}
```

### 2.8 Frontend: Update Connections Screen

**File**: `lib/screens/connections_screen.dart`

**Replace the entire file with**:

```dart
import 'package:flutter/material.dart';
import 'package:flutter_inappwebview/flutter_inappwebview.dart';
import '../services/github_service.dart';
import '../services/device_session_service.dart';

/// Connections screen for managing external service connections
class ConnectionsScreen extends StatefulWidget {
  const ConnectionsScreen({super.key});

  @override
  State<ConnectionsScreen> createState() => _ConnectionsScreenState();
}

class _ConnectionsScreenState extends State<ConnectionsScreen> {
  late final GitHubService _githubService;
  late final DeviceSessionService _sessionService;
  
  bool _isLoading = false;
  bool _isConnected = false;
  String? _username;
  String? _avatarUrl;

  @override
  void initState() {
    super.initState();
    _sessionService = DeviceSessionService();
    _githubService = GitHubService(_sessionService);
    _checkConnectionStatus();
  }

  Future<void> _checkConnectionStatus() async {
    setState(() => _isLoading = true);
    
    try {
      final status = await _githubService.getConnectionStatus();
      setState(() {
        _isConnected = status['isConnected'] ?? false;
        _username = status['username'];
        _avatarUrl = status['avatarUrl'];
      });
    } catch (e) {
      debugPrint('Error checking connection status: $e');
      // Not connected or error - show as disconnected
      setState(() => _isConnected = false);
    } finally {
      setState(() => _isLoading = false);
    }
  }

  Future<void> _connectGitHub() async {
    setState(() => _isLoading = true);
    
    try {
      // Get authorization URL
      final authData = await _githubService.getAuthorizationUrl();
      final authUrl = authData['authorizationUrl'] as String;
      
      if (!mounted) return;
      
      // Open WebView for OAuth
      final result = await Navigator.push(
        context,
        MaterialPageRoute(
          builder: (context) => GitHubOAuthWebView(
            authorizationUrl: authUrl,
          ),
        ),
      );
      
      if (result == true) {
        // Refresh connection status
        await _checkConnectionStatus();
        
        if (mounted) {
          ScaffoldMessenger.of(context).showSnackBar(
            const SnackBar(
              content: Text('GitHub connected successfully!'),
              backgroundColor: Colors.green,
            ),
          );
        }
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text('Failed to connect: $e'),
            backgroundColor: Colors.red,
          ),
        );
      }
    } finally {
      setState(() => _isLoading = false);
    }
  }

  Future<void> _disconnectGitHub() async {
    // Show confirmation dialog
    final confirm = await showDialog<bool>(
      context: context,
      builder: (context) => AlertDialog(
        title: const Text('Disconnect GitHub?'),
        content: const Text(
          'This will remove access to your GitHub repositories. '
          'You can reconnect at any time.'
        ),
        actions: [
          TextButton(
            onPressed: () => Navigator.pop(context, false),
            child: const Text('Cancel'),
          ),
          TextButton(
            onPressed: () => Navigator.pop(context, true),
            style: TextButton.styleFrom(foregroundColor: Colors.red),
            child: const Text('Disconnect'),
          ),
        ],
      ),
    );
    
    if (confirm != true) return;
    
    setState(() => _isLoading = true);
    
    try {
      await _githubService.disconnect();
      await _checkConnectionStatus();
      
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(
            content: Text('GitHub disconnected'),
            backgroundColor: Colors.orange,
          ),
        );
      }
    } catch (e) {
      if (mounted) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(
            content: Text('Failed to disconnect: $e'),
            backgroundColor: Colors.red,
          ),
        );
      }
    } finally {
      setState(() => _isLoading = false);
    }
  }

  Widget _buildSectionHeader(String title) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 4.0),
      child: Text(
        title,
        style: TextStyle(
          color: Colors.grey[400],
          fontSize: 14,
          fontWeight: FontWeight.w600,
          letterSpacing: 0.5,
        ),
      ),
    );
  }

  Widget _buildConnectionCard({
    required IconData icon,
    required String serviceName,
    required String description,
    required bool isConnected,
    required VoidCallback onConnect,
    required VoidCallback onDisconnect,
    String? username,
    String? avatarUrl,
  }) {
    return Container(
      padding: const EdgeInsets.all(16.0),
      decoration: BoxDecoration(
        color: Colors.grey[850],
        borderRadius: BorderRadius.circular(8.0),
      ),
      child: Row(
        children: [
          // Service icon or avatar
          if (isConnected && avatarUrl != null)
            ClipRRect(
              borderRadius: BorderRadius.circular(14),
              child: Image.network(
                avatarUrl,
                width: 28,
                height: 28,
                errorBuilder: (context, error, stackTrace) => Icon(
                  icon,
                  color: Colors.grey[300],
                  size: 28,
                ),
              ),
            )
          else
            Icon(
              icon,
              color: Colors.grey[300],
              size: 28,
            ),
          
          const SizedBox(width: 16),
          
          // Service details
          Expanded(
            child: Column(
              crossAxisAlignment: CrossAxisAlignment.start,
              children: [
                Text(
                  serviceName,
                  style: const TextStyle(
                    color: Colors.white,
                    fontSize: 18,
                    fontWeight: FontWeight.w500,
                  ),
                ),
                const SizedBox(height: 4),
                Text(
                  isConnected && username != null
                      ? 'Connected as $username'
                      : description,
                  style: TextStyle(
                    color: Colors.grey[400],
                    fontSize: 14,
                  ),
                ),
              ],
            ),
          ),
          
          const SizedBox(width: 12),
          
          // Connect/Disconnect button
          ElevatedButton(
            onPressed: _isLoading 
                ? null 
                : (isConnected ? onDisconnect : onConnect),
            style: ElevatedButton.styleFrom(
              backgroundColor: isConnected 
                  ? Colors.red[700] 
                  : Colors.blue[700],
              foregroundColor: Colors.white,
              padding: const EdgeInsets.symmetric(
                horizontal: 20,
                vertical: 12,
              ),
              shape: RoundedRectangleBorder(
                borderRadius: BorderRadius.circular(8.0),
              ),
            ),
            child: _isLoading
                ? const SizedBox(
                    width: 16,
                    height: 16,
                    child: CircularProgressIndicator(
                      strokeWidth: 2,
                      valueColor: AlwaysStoppedAnimation<Color>(Colors.white),
                    ),
                  )
                : Row(
                    mainAxisSize: MainAxisSize.min,
                    children: [
                      if (isConnected) ...[
                        const Icon(Icons.link_off, size: 18),
                        const SizedBox(width: 4),
                        const Text('Disconnect'),
                      ] else ...[
                        const Icon(Icons.link, size: 18),
                        const SizedBox(width: 4),
                        const Text('Connect'),
                      ],
                    ],
                  ),
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.grey[900],
      appBar: AppBar(
        backgroundColor: Colors.grey[850],
        leading: IconButton(
          icon: const Icon(Icons.arrow_back, color: Colors.white),
          onPressed: () => Navigator.pop(context),
        ),
        title: const Text(
          'Connections',
          style: TextStyle(color: Colors.white),
        ),
      ),
      body: ListView(
        padding: const EdgeInsets.all(16.0),
        children: [
          _buildSectionHeader('VERSION CONTROL'),
          const SizedBox(height: 8),

          _buildConnectionCard(
            icon: Icons.code,
            serviceName: 'GitHub',
            description: 'Connect to GitHub repositories',
            isConnected: _isConnected,
            username: _username,
            avatarUrl: _avatarUrl,
            onConnect: _connectGitHub,
            onDisconnect: _disconnectGitHub,
          ),
        ],
      ),
    );
  }
}

/// WebView for GitHub OAuth flow
class GitHubOAuthWebView extends StatefulWidget {
  final String authorizationUrl;
  
  const GitHubOAuthWebView({
    super.key,
    required this.authorizationUrl,
  });

  @override
  State<GitHubOAuthWebView> createState() => _GitHubOAuthWebViewState();
}

class _GitHubOAuthWebViewState extends State<GitHubOAuthWebView> {
  InAppWebViewController? _webViewController;
  bool _isLoading = true;

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      backgroundColor: Colors.grey[900],
      appBar: AppBar(
        backgroundColor: Colors.grey[850],
        leading: IconButton(
          icon: const Icon(Icons.close, color: Colors.white),
          onPressed: () => Navigator.pop(context, false),
        ),
        title: const Text(
          'Connect GitHub',
          style: TextStyle(color: Colors.white),
        ),
      ),
      body: Stack(
        children: [
          InAppWebView(
            initialUrlRequest: URLRequest(
              url: WebUri(widget.authorizationUrl),
            ),
            initialSettings: InAppWebViewSettings(
              useShouldOverrideUrlLoading: true,
              mediaPlaybackRequiresUserGesture: false,
            ),
            onWebViewCreated: (controller) {
              _webViewController = controller;
              
              // Add JavaScript handlers
              controller.addJavaScriptHandler(
                handlerName: 'oauthSuccess',
                callback: (args) {
                  // OAuth successful
                  Navigator.pop(context, true);
                },
              );
              
              controller.addJavaScriptHandler(
                handlerName: 'oauthError',
                callback: (args) {
                  // OAuth failed
                  Navigator.pop(context, false);
                },
              );
            },
            onLoadStart: (controller, url) {
              setState(() => _isLoading = true);
            },
            onLoadStop: (controller, url) {
              setState(() => _isLoading = false);
            },
            onReceivedError: (controller, request, error) {
              debugPrint('WebView error: $error');
            },
          ),
          
          if (_isLoading)
            const Center(
              child: CircularProgressIndicator(),
            ),
        ],
      ),
    );
  }
}
```

---

## PHASE 3: Repository Operations (Read)

**Goal**: Implement repository listing and file reading

**Estimated Time**: 1 conversation session

### 3.1 Backend: GitHub API Service (Part 1 - Read Operations)

**File**: `src/main/java/com/kommentall/service/GitHubApiService.java`

```java
package com.kommentall.service;

import com.kommentall.config.GitHubProperties;
import com.kommentall.model.GitHubConnection;
import com.kommentall.repository.GitHubConnectionRepository;
import org.kohsuke.github.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.time.LocalDateTime;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class GitHubApiService {
    
    private static final Logger log = LoggerFactory.getLogger(GitHubApiService.class);
    
    private final GitHubConnectionRepository connectionRepository;
    private final GitHubRateLimitService rateLimitService;
    private final GitHubProperties properties;
    
    @Autowired
    public GitHubApiService(
            GitHubConnectionRepository connectionRepository,
            GitHubRateLimitService rateLimitService,
            GitHubProperties properties) {
        this.connectionRepository = connectionRepository;
        this.rateLimitService = rateLimitService;
        this.properties = properties;
    }
    
    /**
     * Get GitHub client for device
     */
    private GitHub getGitHubClient(String deviceId) throws IOException {
        GitHubConnection connection = connectionRepository.findByDeviceId(deviceId)
            .orElseThrow(() -> new IllegalStateException("GitHub not connected"));
        
        if (!connection.getIsActive()) {
            throw new IllegalStateException("GitHub connection is not active");
        }
        
        // Update last used timestamp
        connection.setLastUsedAt(LocalDateTime.now());
        connectionRepository.save(connection);
        
        return new GitHubBuilder()
            .withOAuthToken(connection.getAccessToken())
            .build();
    }
    
    /**
     * List all repositories accessible to the user
     */
    public List<Map<String, Object>> listRepositories(String deviceId) 
            throws IOException {
        
        rateLimitService.checkRateLimit(deviceId, "list_repositories");
        
        GitHub github = getGitHubClient(deviceId);
        GHMyself myself = github.getMyself();
        
        List<Map<String, Object>> repositories = new ArrayList<>();
        
        // Get all repositories (user's own + collaborator repos)
        Map<String, GHRepository> allRepos = myself.getAllRepositories();
        
        for (GHRepository repo : allRepos.values()) {
            Map<String, Object> repoInfo = new HashMap<>();
            repoInfo.put("id", repo.getId());
            repoInfo.put("name", repo.getName());
            repoInfo.put("fullName", repo.getFullName());
            repoInfo.put("description", repo.getDescription());
            repoInfo.put("isPrivate", repo.isPrivate());
            repoInfo.put("defaultBranch", repo.getDefaultBranch());
            repoInfo.put("owner", repo.getOwnerName());
            repoInfo.put("url", repo.getHtmlUrl().toString());
            repoInfo.put("language", repo.getLanguage());
            repoInfo.put("updatedAt", repo.getUpdatedAt().toString());
            
            repositories.add(repoInfo);
        }
        
        // Sort by updated date (most recent first)
        repositories.sort((a, b) -> 
            ((String) b.get("updatedAt")).compareTo((String) a.get("updatedAt"))
        );
        
        rateLimitService.recordRequest(deviceId, "list_repositories");
        
        return repositories;
    }
    
    /**
     * Get repository details
     */
    public Map<String, Object> getRepository(String deviceId, String owner, String repoName) 
            throws IOException {
        
        rateLimitService.checkRateLimit(deviceId, "get_repository");
        
        GitHub github = getGitHubClient(deviceId);
        GHRepository repo = github.getRepository(owner + "/" + repoName);
        
        Map<String, Object> repoInfo = new HashMap<>();
        repoInfo.put("id", repo.getId());
        repoInfo.put("name", repo.getName());
        repoInfo.put("fullName", repo.getFullName());
        repoInfo.put("description", repo.getDescription());
        repoInfo.put("isPrivate", repo.isPrivate());
        repoInfo.put("defaultBranch", repo.getDefaultBranch());
        repoInfo.put("owner", repo.getOwnerName());
        repoInfo.put("url", repo.getHtmlUrl().toString());
        repoInfo.put("language", repo.getLanguage());
        repoInfo.put("stars", repo.getStargazersCount());
        repoInfo.put("forks", repo.getForksCount());
        repoInfo.put("openIssues", repo.getOpenIssueCount());
        repoInfo.put("createdAt", repo.getCreatedAt().toString());
        repoInfo.put("updatedAt", repo.getUpdatedAt().toString());
        
        // Get branches
        List<String> branches = repo.getBranches().keySet()
            .stream()
            .collect(Collectors.toList());
        repoInfo.put("branches", branches);
        
        rateLimitService.recordRequest(deviceId, "get_repository");
        
        return repoInfo;
    }
    
    /**
     * List files in a directory
     */
    public List<Map<String, Object>> listFiles(
            String deviceId, 
            String owner, 
            String repoName,
            String path,
            String branch) throws IOException {
        
        rateLimitService.checkRateLimit(deviceId, "list_files");
        
        GitHub github = getGitHubClient(deviceId);
        GHRepository repo = github.getRepository(owner + "/" + repoName);
        
        if (branch == null || branch.isEmpty()) {
            branch = repo.getDefaultBranch();
        }
        
        List<GHContent> contents = repo.getDirectoryContent(path, branch);
        
        List<Map<String, Object>> files = new ArrayList<>();
        
        for (GHContent content : contents) {
            Map<String, Object> fileInfo = new HashMap<>();
            fileInfo.put("name", content.getName());
            fileInfo.put("path", content.getPath());
            fileInfo.put("type", content.getType());  // "file" or "dir"
            fileInfo.put("size", content.getSize());
            fileInfo.put("sha", content.getSha());
            fileInfo.put("url", content.getHtmlUrl());
            
            files.add(fileInfo);
        }
        
        // Sort: directories first, then files, alphabetically
        files.sort((a, b) -> {
            String typeA = (String) a.get("type");
            String typeB = (String) b.get("type");
            
            if (!typeA.equals(typeB)) {
                return typeA.equals("dir") ? -1 : 1;
            }
            
            return ((String) a.get("name")).compareToIgnoreCase((String) b.get("name"));
        });
        
        rateLimitService.recordRequest(deviceId, "list_files");
        
        return files;
    }
    
    /**
     * Read file content
     */
    public Map<String, Object> readFile(
            String deviceId,
            String owner,
            String repoName,
            String path,
            String branch) throws IOException {
        
        rateLimitService.checkRateLimit(deviceId, "read_file");
        
        GitHub github = getGitHubClient(deviceId);
        GHRepository repo = github.getRepository(owner + "/" + repoName);
        
        if (branch == null || branch.isEmpty()) {
            branch = repo.getDefaultBranch();
        }
        
        GHContent content = repo.getFileContent(path, branch);
        
        if (!content.isFile()) {
            throw new IllegalArgumentException("Path is not a file: " + path);
        }
        
        Map<String, Object> fileInfo = new HashMap<>();
        fileInfo.put("name", content.getName());
        fileInfo.put("path", content.getPath());
        fileInfo.put("size", content.getSize());
        fileInfo.put("sha", content.getSha());
        fileInfo.put("content", content.getContent());  // Base64 decoded content
        fileInfo.put("encoding", content.getEncoding());
        
        rateLimitService.recordRequest(deviceId, "read_file");
        
        return fileInfo;
    }
    
    /**
     * Search code across repositories
     */
    public List<Map<String, Object>> searchCode(
            String deviceId,
            String query,
            String owner,
            String repoName) throws IOException {
        
        rateLimitService.checkRateLimit(deviceId, "search_code");
        
        GitHub github = getGitHubClient(deviceId);
        
        // Build search query
        String searchQuery = query;
        if (owner != null && repoName != null) {
            searchQuery += " repo:" + owner + "/" + repoName;
        }
        
        GHContentSearchBuilder searchBuilder = github.searchContent();
        PagedSearchIterable<GHContent> results = searchBuilder.q(searchQuery).list();
        
        List<Map<String, Object>> searchResults = new ArrayList<>();
        int maxResults = 50;  // Limit results
        
        for (GHContent content : results) {
            if (searchResults.size() >= maxResults) break;
            
            Map<String, Object> result = new HashMap<>();
            result.put("name", content.getName());
            result.put("path", content.getPath());
            result.put("repository", content.getOwner().getFullName());
            result.put("url", content.getHtmlUrl());
            result.put("sha", content.getSha());
            
            searchResults.add(result);
        }
        
        rateLimitService.recordRequest(deviceId, "search_code");
        
        return searchResults;
    }
}
```

**(Continued in next file due to length...)**

### 3.2 Backend: Rate Limit Service

**File**: `src/main/java/com/kommentall/service/GitHubRateLimitService.java`

```java
package com.kommentall.service;

import com.kommentall.config.GitHubProperties;
import com.kommentall.model.GitHubRateLimit;
import com.kommentall.repository.GitHubRateLimitRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.Optional;

@Service
public class GitHubRateLimitService {
    
    private final GitHubRateLimitRepository rateLimitRepository;
    private final GitHubProperties properties;
    
    @Autowired
    public GitHubRateLimitService(
            GitHubRateLimitRepository rateLimitRepository,
            GitHubProperties properties) {
        this.rateLimitRepository = rateLimitRepository;
        this.properties = properties;
    }
    
    /**
     * Check if request would exceed rate limit
     */
    public void checkRateLimit(String deviceId, String endpoint) {
        Optional<GitHubRateLimit> rateLimitOpt = rateLimitRepository
            .findByDeviceIdAndEndpointAndWindowEndAfter(
                deviceId, 
                endpoint, 
                LocalDateTime.now()
            );
        
        if (rateLimitOpt.isPresent()) {
            GitHubRateLimit rateLimit = rateLimitOpt.get();
            
            if (rateLimit.getRequestCount() >= 
                    properties.getApi().getRateLimit().getMaxRequestsPerHour()) {
                throw new RuntimeException(
                    "Rate limit exceeded. Try again after " + 
                    rateLimit.getWindowEnd()
                );
            }
        }
    }
    
    /**
     * Record a request for rate limiting
     */
    @Transactional
    public void recordRequest(String deviceId, String endpoint) {
        LocalDateTime now = LocalDateTime.now();
        
        Optional<GitHubRateLimit> rateLimitOpt = rateLimitRepository
            .findByDeviceIdAndEndpointAndWindowEndAfter(
                deviceId, 
                endpoint, 
                now
            );
        
        if (rateLimitOpt.isPresent()) {
            // Increment counter in existing window
            GitHubRateLimit rateLimit = rateLimitOpt.get();
            rateLimit.setRequestCount(rateLimit.getRequestCount() + 1);
            rateLimitRepository.save(rateLimit);
        } else {
            // Create new window
            GitHubRateLimit rateLimit = new GitHubRateLimit();
            rateLimit.setDeviceId(deviceId);
            rateLimit.setEndpoint(endpoint);
            rateLimit.setRequestCount(1);
            rateLimit.setWindowStart(now);
            rateLimit.setWindowEnd(now.plusHours(1));
            rateLimitRepository.save(rateLimit);
        }
    }
}
```

### 3.3 Backend: Repository Operations Controller

**File**: `src/main/java/com/kommentall/controller/GitHubRepositoryController.java`

```java
package com.kommentall.controller;

import com.kommentall.service.DeviceSessionService;
import com.kommentall.service.GitHubApiService;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;
import java.util.Map;

@RestController
@RequestMapping("/api/github/repositories")
public class GitHubRepositoryController {
    
    private static final Logger log = LoggerFactory.getLogger(GitHubRepositoryController.class);
    
    private final GitHubApiService githubApiService;
    private final DeviceSessionService sessionService;
    
    @Autowired
    public GitHubRepositoryController(
            GitHubApiService githubApiService,
            DeviceSessionService sessionService) {
        this.githubApiService = githubApiService;
        this.sessionService = sessionService;
    }
    
    /**
     * List all accessible repositories
     */
    @GetMapping
    public ResponseEntity<?> listRepositories(
            @RequestHeader("X-Device-Token") String deviceToken) {
        try {
            String deviceId = sessionService.validateAndExtractDeviceId(deviceToken);
            List<Map<String, Object>> repos = githubApiService.listRepositories(deviceId);
            
            return ResponseEntity.ok(repos);
            
        } catch (Exception e) {
            log.error("Error listing repositories", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", e.getMessage()));
        }
    }
    
    /**
     * Get repository details
     */
    @GetMapping("/{owner}/{repoName}")
    public ResponseEntity<?> getRepository(
            @RequestHeader("X-Device-Token") String deviceToken,
            @PathVariable String owner,
            @PathVariable String repoName) {
        try {
            String deviceId = sessionService.validateAndExtractDeviceId(deviceToken);
            Map<String, Object> repo = githubApiService.getRepository(
                deviceId, owner, repoName
            );
            
            return ResponseEntity.ok(repo);
            
        } catch (Exception e) {
            log.error("Error getting repository", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", e.getMessage()));
        }
    }
    
    /**
     * List files in directory
     */
    @GetMapping("/{owner}/{repoName}/files")
    public ResponseEntity<?> listFiles(
            @RequestHeader("X-Device-Token") String deviceToken,
            @PathVariable String owner,
            @PathVariable String repoName,
            @RequestParam(defaultValue = "") String path,
            @RequestParam(required = false) String branch) {
        try {
            String deviceId = sessionService.validateAndExtractDeviceId(deviceToken);
            List<Map<String, Object>> files = githubApiService.listFiles(
                deviceId, owner, repoName, path, branch
            );
            
            return ResponseEntity.ok(files);
            
        } catch (Exception e) {
            log.error("Error listing files", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", e.getMessage()));
        }
    }
    
    /**
     * Read file content
     */
    @GetMapping("/{owner}/{repoName}/file")
    public ResponseEntity<?> readFile(
            @RequestHeader("X-Device-Token") String deviceToken,
            @PathVariable String owner,
            @PathVariable String repoName,
            @RequestParam String path,
            @RequestParam(required = false) String branch) {
        try {
            String deviceId = sessionService.validateAndExtractDeviceId(deviceToken);
            Map<String, Object> file = githubApiService.readFile(
                deviceId, owner, repoName, path, branch
            );
            
            return ResponseEntity.ok(file);
            
        } catch (Exception e) {
            log.error("Error reading file", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", e.getMessage()));
        }
    }
    
    /**
     * Search code
     */
    @GetMapping("/search")
    public ResponseEntity<?> searchCode(
            @RequestHeader("X-Device-Token") String deviceToken,
            @RequestParam String query,
            @RequestParam(required = false) String owner,
            @RequestParam(required = false) String repoName) {
        try {
            String deviceId = sessionService.validateAndExtractDeviceId(deviceToken);
            List<Map<String, Object>> results = githubApiService.searchCode(
                deviceId, query, owner, repoName
            );
            
            return ResponseEntity.ok(results);
            
        } catch (Exception e) {
            log.error("Error searching code", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(Map.of("error", e.getMessage()));
        }
    }
}
```

---

## PHASE 4: Repository Operations (Write & Commit)

**Goal**: Implement file creation, modification, deletion, and committing to branches

**Estimated Time**: 1 conversation session

**(Implementation details for Phase 4 will be provided in the next conversation)**

---

## PHASE 5: Repository Selection & Management

**Goal**: Allow users to select specific repositories and manage their selections

**Estimated Time**: 1 conversation session

**(Implementation details for Phase 5 will be provided in the next conversation)**

---

## PHASE 6: UI Enhancement & Repository Browser

**Goal**: Create comprehensive UI for browsing repositories, files, and committing changes

**Estimated Time**: 1 conversation session

**(Implementation details for Phase 6 will be provided in the next conversation)**

---

## Future Enhancements (Post Phase 6)

### User Authentication System
- Spring Security with JWT
- User registration and login
- Password encryption
- OAuth integration with existing GitHub connection
- Multi-device support per user

### Advanced Features
- Pull requests creation
- Issue browsing and creation
- Branch management UI
- Merge conflict resolution
- Repository cloning to local storage
- Webhook integration for notifications
- Collaborative features

### Performance Optimizations
- Repository content caching
- Lazy loading for large file trees
- Background sync service
- Optimistic UI updates

---

## Testing Strategy

### Phase 1-2 Testing
- Device registration flow
- OAuth flow end-to-end
- Token storage and retrieval
- Connection status checks
- WebView OAuth in Flutter

### Phase 3 Testing
- Repository listing
- File browsing
- File reading
- Search functionality
- Rate limiting

### Phase 4-5 Testing
- File creation and modification
- Branch creation
- Commit operations
- Repository selection
- Error handling

### Phase 6 Testing
- Full UI flow
- Cross-platform compatibility
- Performance under load
- Security audit

---

## Security Considerations

### Critical Security Measures
1. **Token Storage**: Never expose GitHub tokens to frontend
2. **HTTPS Only**: Enforce HTTPS in production
3. **CSRF Protection**: State parameter in OAuth
4. **Rate Limiting**: Prevent abuse
5. **Input Validation**: Sanitize all inputs
6. **SQL Injection**: Use parameterized queries (JPA handles this)
7. **XSS Protection**: Sanitize displayed content
8. **Device Token Expiration**: Implement token refresh

### Production Checklist
- [ ] Move secrets to HashiCorp Vault
- [ ] Enable HTTPS with valid certificates
- [ ] Set up monitoring and alerting
- [ ] Implement audit logging
- [ ] Configure CORS properly (restrict origins)
- [ ] Set up database backups
- [ ] Enable database encryption at rest
- [ ] Implement API request logging
- [ ] Set up error tracking (Sentry/similar)
- [ ] Load testing

---

## Deployment Architecture

### Development
```
Flutter App (Android Emulator)
    ↓
Backend (localhost:8080)
    ↓
PostgreSQL (localhost:5432)
    ↓
GitHub API
```

### Production
```
Flutter Apps (iOS/Android/Web)
    ↓
Load Balancer (Digital Ocean)
    ↓
Kubernetes Cluster
    ├─ Backend Pods (Spring Boot)
    └─ HashiCorp Vault
    ↓
Managed PostgreSQL (Digital Ocean)
```

---

## Monitoring & Observability

### Metrics to Track
- OAuth success/failure rates
- API response times
- Rate limit hits
- Error rates by endpoint
- Active device connections
- Database connection pool usage
- GitHub API quota usage

### Logging Strategy
- Structured JSON logging
- Request/response logging (excluding sensitive data)
- Error stack traces
- Performance metrics
- Security events

---

## Cost Estimation

### GitHub API
- Free tier: 5,000 requests/hour (authenticated)
- Cost: $0 for most users

### Digital Ocean (Production)
- Kubernetes Cluster: ~$40/month (2 nodes)
- Managed PostgreSQL: ~$15/month (1GB RAM)
- Load Balancer: ~$12/month
- **Total**: ~$67/month

### Development
- No cost (local development)

---

## Timeline

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| Phase 1 | 2-3 hours | OAuth foundation setup |
| Phase 2 | 3-4 hours | Complete OAuth flow |
| Phase 3 | 2-3 hours | Read operations |
| Phase 4 | 3-4 hours | Write operations |
| Phase 5 | 2 hours | Repository selection |
| Phase 6 | 4-5 hours | UI polish |
| **Total** | **16-21 hours** | Full feature |

---

## Support & Documentation

### User Documentation Needed
- How to connect GitHub account
- How to select repositories
- How to browse files
- How to commit changes
- Troubleshooting guide

### Developer Documentation Needed
- API endpoint documentation
- Database schema
- OAuth flow diagram
- Deployment guide
- Security guidelines

---

**END OF PLAN**

This plan should be used as a reference document for implementing the GitHub integration feature in multiple conversation sessions. Each phase is designed to be self-contained and can be implemented independently.
