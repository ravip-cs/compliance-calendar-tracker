# Developer Fix Recommendations
## Compliance Calendar & Tracker — Security Remediation Guide

**Prepared by:** Security Reviewer  
**For:** Development Team  
**Sprint:** 3 → 4 Remediation  
**Priority:** Address P0 fixes before any staging/production deployment

---

> This document provides copy-paste ready code fixes. Each section maps to a vulnerability in `VULNERABILITY_REPORT.md`. Implement in priority order.

---

## FIX-001 — Remove `.env` from Repository (P0 — Do Today)

### Step 1: Purge from git history

```bash
# Run from project root
git rm --cached .env
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore

# Commit the removal
git add .gitignore
git commit -m "security: remove .env from tracking, add to .gitignore"

# Remove from ALL historical commits
git filter-branch --force --index-filter \
  'git rm --cached --ignore-unmatch .env' \
  --prune-empty --tag-name-filter cat -- --all

# Force push to overwrite remote history
git push origin --force --all
git push origin --force --tags
```

### Step 2: Create `.env.example` (commit this instead)

```env
# .env.example — Copy to .env and fill in real values
# NEVER commit .env

# Spring Boot
JWT_SECRET=GENERATE_WITH: openssl rand -hex 64
JWT_EXPIRATION_MS=3600000
DB_URL=jdbc:mysql://localhost:3306/compliance_db
DB_USERNAME=CHANGE_ME
DB_PASSWORD=CHANGE_ME

# Flask AI Service
OPENAI_API_KEY=sk-CHANGE_ME
INTERNAL_API_KEY=GENERATE_WITH: openssl rand -hex 32

# Flask
FLASK_ENV=production
FLASK_DEBUG=0
```

### Step 3: Rotate all secrets immediately

- **OpenAI API Key:** Go to `platform.openai.com` → API Keys → Delete old key → Create new
- **JWT Secret:** Generate new: `openssl rand -hex 64`
- **Database password:** `ALTER USER 'dbuser'@'localhost' IDENTIFIED BY 'NEW_STRONG_PASSWORD';`

---

## FIX-002 — Secure Flask AI Service (P0 — Do Today)

### File: `app.py` — Complete replacement

```python
# app.py
import os
from flask import Flask, jsonify, request, abort
from functools import wraps
from routes.health_routes import health_bp
from routes.describe_routes import describe_bp
from routes.recommend_routes import recommend_bp
from routes.report_routes import report_bp

app = Flask(__name__)

# Load internal key from environment — never hardcode
INTERNAL_API_KEY = os.environ.get("INTERNAL_API_KEY")
if not INTERNAL_API_KEY:
    raise RuntimeError("INTERNAL_API_KEY environment variable not set")


def require_internal_key(f):
    """Decorator: validates X-Internal-Key header on all AI routes."""
    @wraps(f)
    def decorated(*args, **kwargs):
        key = request.headers.get("X-Internal-Key", "")
        if not key or key != INTERNAL_API_KEY:
            return jsonify({"error": "Unauthorized"}), 401
        return f(*args, **kwargs)
    return decorated


# Register blueprints
app.register_blueprint(health_bp)           # No auth required
app.register_blueprint(describe_bp)
app.register_blueprint(recommend_bp)
app.register_blueprint(report_bp)


@app.route("/")
def home():
    return jsonify({"message": "AI Service Running"})


@app.errorhandler(Exception)
def handle_exception(e):
    # Never expose internal errors to client
    app.logger.error(f"Unhandled exception: {e}")
    return jsonify({"error": "Internal server error"}), 500


@app.errorhandler(404)
def not_found(e):
    return jsonify({"error": "Not found"}), 404


if __name__ == "__main__":
    app.run(
        port=5000,
        debug=False,         # NEVER True in production
        host="127.0.0.1",    # Bind localhost only
    )
```

### Apply `require_internal_key` to route files

```python
# routes/describe_routes.py — add decorator
from flask import Blueprint, request, jsonify
from app import require_internal_key  # import decorator

describe_bp = Blueprint("describe", __name__)

@describe_bp.route("/describe", methods=["POST"])
@require_internal_key   # ← Add this line
def describe():
    data = request.get_json()
    # ... existing logic
```

Repeat for `recommend_routes.py` and `report_routes.py`.

### Spring Boot: Send internal key when calling Flask

```java
// AIServiceClient.java
@Service
public class AIServiceClient {

    @Value("${ai.service.url}")
    private String aiServiceUrl;

    @Value("${ai.internal.key}")
    private String internalKey;

    private final RestTemplate restTemplate;

    public String callDescribe(String taskDescription) {
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.APPLICATION_JSON);
        headers.set("X-Internal-Key", internalKey);  // ← Always include

        Map<String, String> body = Map.of("task", taskDescription);
        HttpEntity<Map<String, String>> entity = new HttpEntity<>(body, headers);

        ResponseEntity<String> response = restTemplate.postForEntity(
            aiServiceUrl + "/describe", entity, String.class
        );
        return response.getBody();
    }
}
```

```properties
# application.properties
ai.service.url=http://127.0.0.1:5000
ai.internal.key=${INTERNAL_API_KEY}
```

---

## FIX-003 — Prompt Injection Sanitization (P1)

### New file: `services/input_sanitizer.py`

```python
# services/input_sanitizer.py
import re
from typing import Optional

# Patterns that indicate prompt injection attempts
INJECTION_PATTERNS = [
    r"ignore\s+(all\s+)?previous\s+instructions",
    r"disregard\s+(all\s+)?previous",
    r"you\s+are\s+now\s+",
    r"act\s+as\s+(if\s+you\s+are|an?\s+)",
    r"pretend\s+(you\s+are|to\s+be)",
    r"system\s*:\s*",
    r"system\s+prompt",
    r"repeat\s+(everything|all|your\s+(context|prompt|instructions))",
    r"output\s+(your\s+)?(system|context|configuration|api\s+key)",
    r"reveal\s+your\s+(prompt|instructions|key)",
    r"you\s+have\s+no\s+restrictions",
    r"DAN\s+mode",
    r"jailbreak",
]

COMPILED_PATTERNS = [re.compile(p, re.IGNORECASE) for p in INJECTION_PATTERNS]

MAX_INPUT_LENGTH = 2000


def sanitize_ai_input(text: Optional[str], field_name: str = "input") -> str:
    """
    Sanitizes user input before passing to LLM.
    
    Raises ValueError if input is invalid or contains injection patterns.
    Returns sanitized string.
    """
    if not text:
        raise ValueError(f"{field_name} is required")
    
    if not isinstance(text, str):
        raise ValueError(f"{field_name} must be a string")
    
    text = text.strip()
    
    if len(text) > MAX_INPUT_LENGTH:
        raise ValueError(f"{field_name} exceeds maximum length of {MAX_INPUT_LENGTH} characters")
    
    if len(text) < 3:
        raise ValueError(f"{field_name} is too short")
    
    # Check for injection patterns
    for pattern in COMPILED_PATTERNS:
        if pattern.search(text):
            raise ValueError(f"{field_name} contains disallowed content")
    
    return text
```

### Updated route example:

```python
# routes/describe_routes.py
from flask import Blueprint, request, jsonify
from services.input_sanitizer import sanitize_ai_input

describe_bp = Blueprint("describe", __name__)

@describe_bp.route("/describe", methods=["POST"])
@require_internal_key
def describe():
    data = request.get_json(silent=True)
    if not data:
        return jsonify({"error": "Invalid JSON body"}), 400
    
    try:
        task = sanitize_ai_input(data.get("task"), field_name="task")
    except ValueError as e:
        return jsonify({"error": str(e)}), 400
    
    # Now safe to pass to LLM
    result = ai_describe_service(task)
    return jsonify({"description": result})
```

---

## FIX-004 — Rate Limiting on Login Endpoint (P2)

### Add Bucket4j dependency to `pom.xml`

```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.3.0</version>
</dependency>
```

### Create `RateLimitFilter.java`

```java
// src/main/java/com/compliance/security/RateLimitFilter.java
package com.compliance.security;

import io.github.bucket4j.*;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import javax.servlet.*;
import javax.servlet.http.*;
import java.io.IOException;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class RateLimitFilter extends OncePerRequestFilter {

    private static final int MAX_REQUESTS_PER_MINUTE = 5;
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    private Bucket createBucketForIp() {
        return Bucket.builder()
            .addLimit(Bandwidth.classic(MAX_REQUESTS_PER_MINUTE,
                Refill.greedy(MAX_REQUESTS_PER_MINUTE, Duration.ofMinutes(1))))
            .build();
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        // Only rate-limit login endpoint
        if ("/api/auth/login".equals(request.getRequestURI())
                && "POST".equals(request.getMethod())) {

            String clientIp = getClientIp(request);
            Bucket bucket = buckets.computeIfAbsent(clientIp, k -> createBucketForIp());

            if (!bucket.tryConsume(1)) {
                response.setStatus(HttpServletResponse.SC_TOO_MANY_REQUESTS); // 429
                response.setContentType("application/json");
                response.getWriter().write(
                    "{\"error\":\"Too many login attempts. Try again in 1 minute.\"}");
                return;
            }
        }

        filterChain.doFilter(request, response);
    }

    private String getClientIp(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

---

## FIX-005 — IDOR Prevention (P2)

### Update Service Layer

```java
// TaskService.java
@Service
public class TaskService {

    @Autowired
    private TaskRepository taskRepository;

    public Task getTaskById(Long taskId, String currentUsername) {
        Task task = taskRepository.findById(taskId)
            .orElseThrow(() -> new ResourceNotFoundException("Task not found: " + taskId));

        // CRITICAL: Always verify ownership
        if (!task.getOwner().getUsername().equals(currentUsername)) {
            // Use 403, not 404 — 404 leaks that the resource exists
            throw new AccessDeniedException("Access denied to task: " + taskId);
        }

        return task;
    }

    public void deleteTask(Long taskId, String currentUsername) {
        Task task = getTaskById(taskId, currentUsername); // Reuse — already checks ownership
        taskRepository.delete(task);
    }

    public List<Task> getTasksForUser(String currentUsername) {
        // Always filter by owner — never return all tasks
        return taskRepository.findByOwnerUsername(currentUsername);
    }
}
```

### Update Controller to Pass Username

```java
// TaskController.java
@RestController
@RequestMapping("/api/tasks")
public class TaskController {

    @Autowired
    private TaskService taskService;

    @GetMapping("/{id}")
    public ResponseEntity<Task> getTask(@PathVariable Long id,
                                         Authentication auth) {
        String username = auth.getName(); // Get from JWT, not request body
        Task task = taskService.getTaskById(id, username);
        return ResponseEntity.ok(task);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteTask(@PathVariable Long id,
                                            Authentication auth) {
        String username = auth.getName();
        taskService.deleteTask(id, username);
        return ResponseEntity.noContent().build();
    }
}
```

### Repository Method

```java
// TaskRepository.java
public interface TaskRepository extends JpaRepository<Task, Long> {
    List<Task> findByOwnerUsername(String username); // Add this
}
```

---

## FIX-006 — Disable Spring Boot Actuator (P3)

```properties
# src/main/resources/application.properties

# Option A: Expose health only (recommended)
management.endpoints.web.exposure.include=health
management.endpoint.health.show-details=never

# Option B: Disable actuator web exposure completely
management.endpoints.web.exposure.exclude=*

# Move actuator to non-standard path if health check needed
management.endpoints.web.base-path=/internal/mgmt
```

---

## FIX-007 — JWT Storage in HttpOnly Cookie (P3)

### Spring Boot: Set cookie instead of returning token in body

```java
// AuthController.java
@PostMapping("/login")
public ResponseEntity<Map<String, String>> login(
        @RequestBody LoginRequest request,
        HttpServletResponse response) {

    String token = authService.authenticate(request);

    // Set as HttpOnly cookie instead of response body
    Cookie jwtCookie = new Cookie("jwt", token);
    jwtCookie.setHttpOnly(true);   // JS cannot read this
    jwtCookie.setSecure(true);     // HTTPS only
    jwtCookie.setPath("/");
    jwtCookie.setMaxAge(3600);     // 1 hour
    // jwtCookie.setAttribute("SameSite", "Strict");
    response.addCookie(jwtCookie);

    return ResponseEntity.ok(Map.of("message", "Login successful"));
}
```

### React: Use `credentials: 'include'` in fetch calls

```javascript
// api.js
export const apiCall = (endpoint, options = {}) => {
  return fetch(`http://localhost:8080${endpoint}`, {
    ...options,
    credentials: 'include',  // Send cookies automatically
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
  });
};

// Remove all localStorage.getItem('token') references
// Remove all Authorization header manual setting
```

---

## Quick Reference — Fix Priority

| Fix ID | File(s) to Edit | Time Estimate | Done? |
|--------|-----------------|---------------|-------|
| FIX-001 | `.env`, `.gitignore`, git history | 30 min | ☐ |
| FIX-002 | `app.py`, all `routes/*.py` | 2 hrs | ☐ |
| FIX-003 | New: `services/input_sanitizer.py` + routes | 2 hrs | ☐ |
| FIX-004 | `pom.xml`, new `RateLimitFilter.java` | 2 hrs | ☐ |
| FIX-005 | `TaskService.java`, `TaskController.java`, `TaskRepository.java` | 2 hrs | ☐ |
| FIX-006 | `application.properties` | 15 min | ☐ |
| FIX-007 | `AuthController.java`, `api.js` | 3 hrs | ☐ |

---

*Questions? Contact the Security Reviewer before implementing — especially for FIX-001 (git history rewrite affects all collaborators).*
