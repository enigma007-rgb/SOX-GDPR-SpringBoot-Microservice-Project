╔══════════════════════════════════════════════════════════════════════════════╗
║         COMPLETE DRY RUN — SOX & GDPR COMPLIANT EMPLOYEE-MS                ║
║         Tracing every layer: HTTP → Filter → Controller →                  ║
║         Service → Repository → DB → Response                               ║
╚══════════════════════════════════════════════════════════════════════════════╝

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 STARTUP SEQUENCE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[1] Spring Boot starts → loads application.yml
    DB_URL     = jdbc:mysql://localhost:3306/employeedb
    DB_USERNAME = root
    DB_PASSWORD = ****
    ENCRYPTION_KEY = "MySecretKey12345MySecretKey12345"  ← 32 bytes ✅
    JWT_SECRET = "ThisMustBeA256BitSecretKey..."

[2] EncryptionService constructor fires
    → keyBytes = "MySecretKey12345MySecretKey12345".getBytes()
    → keyBytes.length == 32 ✅
    → PASSES — app continues booting
    (If key was 16 bytes → throws IllegalStateException → app REFUSES to start)

[3] AuditConfig fires → @EnableJpaAuditing wired
    → auditorProvider bean registered
    → Spring will auto-populate @CreatedBy/@LastModifiedBy from SecurityContext

[4] Flyway fires BEFORE Hibernate
    → Checks schema_version table in DB
    → Detects V1, V2, V3 not yet applied
    → Runs V1__create_employees_table.sql
       CREATE TABLE employees (id, first_name_enc, last_name_enc,
                               email_enc, department, position,
                               joining_date, is_active, erased_at,
                               consent_given, created_at, created_by ...)
    → Runs V2__create_audit_and_compliance_tables.sql
       CREATE TABLE audit_log (...)
       CREATE TABLE erasure_records (...)
    → Runs V3__create_app_users.sql
       CREATE TABLE app_users (...)
       INSERT admin user (BCrypt hashed password)
    → Flyway complete ✅

[5] Hibernate validates schema (ddl-auto=validate)
    → Employee.java field names map to column names ✅
    → Envers detects @Audited on Employee
    → Creates employees_aud + revinfo tables automatically
    → Schema validation PASSES ✅

[6] HikariCP pool initialises
    → minimum-idle=3 connections opened to MySQL
    → maximum-pool-size=10

[7] SecurityConfig fires
    → JwtAuthenticationFilter registered in filter chain
    → /api/v1/auth/** → permitAll
    → /actuator/health → permitAll
    → everything else → authenticated

[8] Application ready
    → Listening on port 8080
    LOG: "Started EmloyeemsApplication in 4.2 seconds"

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 1 — LOGIN
 POST http://localhost:8080/api/v1/auth/login
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REQUEST:
  POST /api/v1/auth/login
  Content-Type: application/json
  Body: { "username": "admin", "password": "Admin@123" }

─── LAYER 1: JwtAuthenticationFilter ───────────────────────────────────────
  → Checks Authorization header → MISSING
  → URL matches /api/v1/auth/** → permitAll rule applies
  → Filter passes through without blocking ✅

─── LAYER 2: AuthController ─────────────────────────────────────────────────
  → Receives LoginRequest { username="admin", password="Admin@123" }
  → Calls AuthenticationManager.authenticate(
        new UsernamePasswordAuthenticationToken("admin", "Admin@123")
    )

─── LAYER 3: UserDetailsServiceImpl ─────────────────────────────────────────
  → SQL: SELECT * FROM app_users WHERE username = 'admin'
  → Returns: { username="admin",
               password="$2a$12$LQv3...",   ← BCrypt hash
               role="ADMIN" }
  → BCryptPasswordEncoder.matches("Admin@123", "$2a$12$LQv3...") = TRUE ✅

─── LAYER 4: JwtService.generateToken() ────────────────────────────────────
  → Creates JWT:
      Header:  { "alg": "HS256", "typ": "JWT" }
      Payload: { "sub": "admin",
                 "roles": ["ROLE_ADMIN"],
                 "iat": 1710000000,
                 "exp": 1710086400 }    ← 24 hours from now
  → Signs with HMAC-SHA256 using JWT_SECRET
  → Returns: "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.xxxx"

RESPONSE: 200 OK
  {
    "success": true,
    "message": "Login successful",
    "data": {
      "token": "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhZG1pbiJ9.xxxx",
      "role": "ADMIN"
    },
    "timestamp": "2024-03-10T10:00:00Z"
  }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 2 — CREATE EMPLOYEE (Happy Path)
 POST http://localhost:8080/api/v1/employees
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REQUEST:
  POST /api/v1/employees
  Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
  Content-Type: application/json
  Body:
  {
    "firstName":   "Alice",
    "lastName":    "Smith",
    "email":       "alice.smith@company.com",
    "department":  "Engineering",
    "position":    "Senior Developer",
    "joiningDate": "2024-03-10"
  }

─── LAYER 1: CorrelationIdFilter ────────────────────────────────────────────
  → No X-Correlation-Id in request
  → Generates: X-Correlation-Id = "a3f9-11bc-..." (UUID)
  → Stamps on request AND will add to response header

─── LAYER 2: JwtAuthenticationFilter ───────────────────────────────────────
  → Reads Authorization header: "Bearer eyJhbGciOiJIUzI1NiJ9..."
  → Calls JwtService.extractUsername("eyJ...") → "admin"
  → Calls UserDetailsService.loadUserByUsername("admin")
    → SQL: SELECT * FROM app_users WHERE username = 'admin'
  → JwtService.isTokenValid(token, userDetails) → TRUE ✅
  → Creates UsernamePasswordAuthenticationToken {
        principal = "admin",
        authorities = [ROLE_ADMIN]
    }
  → Sets into SecurityContextHolder ✅
  → Filter passes to next

─── LAYER 3: Spring Security Authorization ──────────────────────────────────
  → URL: POST /api/v1/employees
  → anyRequest().authenticated() → user IS authenticated ✅
  → Passes to controller

─── LAYER 4: EmployeeController.saveEmployee() ──────────────────────────────
  → @PreAuthorize("hasAnyRole('ADMIN', 'HR')")
  → SecurityContext has ROLE_ADMIN ✅
  → @Valid triggers validation on CreateEmployeeRequest:
      firstName = "Alice"             → @NotBlank ✅, @Size(2,50) ✅
      lastName  = "Smith"             → @NotBlank ✅, @Size(2,50) ✅
      email     = "alice.smith@..."   → @Email ✅
      department= "Engineering"       → @NotBlank ✅
      position  = "Senior Developer"  → @NotBlank ✅
      joiningDate = "2024-03-10"      → @PastOrPresent ✅
  → ALL validations pass → proceeds to service

─── LAYER 5: AuditAspect.auditMethodCall() (AOP intercepts BEFORE) ──────────
  → @Around fires because saveEmployee() has @Auditable(
        resourceType="Employee", action="CREATE")
  → Captures from SecurityContext:
      userId   = "admin"
      userRole = "[ROLE_ADMIN]"
  → Calls jp.proceed() → enters EmployeeService.saveEmployee()

─── LAYER 6: EmployeeService.saveEmployee() ─────────────────────────────────
  STEP 1: Duplicate email check
  → employeeRepository.existsByEmail("alice.smith@company.com")
  → SQL: SELECT COUNT(*) FROM employees WHERE email_enc = ?
  
  ⚠️  IMPORTANT: email is encrypted before the SQL query fires!
  EncryptedStringConverter.convertToDatabaseColumn("alice.smith@company.com")
    → iv = [random 12 bytes] e.g. [0x3a, 0x9f, ...]
    → AES-256-GCM encrypt("alice.smith@company.com", key, iv)
    → ciphertext + 16-byte auth tag appended
    → base64 encode(iv + ciphertext)
    → DB query value = "OPZh/X+Kd1...base64ciphertext=="
  
  → SQL fires: SELECT COUNT(*) FROM employees
               WHERE email_enc = 'OPZh/X+Kd1...=='
  → Result: 0 (no duplicate) ✅

  STEP 2: modelMapper.map(request, Employee.class)
  → Employee {
        firstName  = "Alice",
        lastName   = "Smith",
        email      = "alice.smith@company.com",
        department = "Engineering",
        position   = "Senior Developer",
        joiningDate = 2024-03-10,
        isActive   = true,    ← set explicitly
        consentGiven = false  ← default — consent separate step
    }

─── LAYER 7: employeeRepository.save(employee) ──────────────────────────────
  → @Transactional begins

  → Spring Auditing fires BEFORE SQL:
      @CreatedDate  → createdAt  = "2024-03-10T10:05:23Z"
      @LastModified → updatedAt  = "2024-03-10T10:05:23Z"
      @CreatedBy    → AuditorAware.getCurrentAuditor() called
                   → SecurityContextHolder.getAuthentication().getName()
                   → createdBy  = "admin"
      @LastModifiedBy → updatedBy = "admin"

  → UUID generated: id = "550e8400-e29b-41d4-a716-446655440000"

  → EncryptedStringConverter fires for each PII field:

      firstName "Alice" →
        iv = [0xA1, 0x3B, ...] (12 random bytes)
        AES-256-GCM encrypt
        base64(iv + ciphertext) = "oTsL9mNp...EncryptedAlice=="
        STORED IN DB: first_name_enc = "oTsL9mNp...EncryptedAlice=="

      lastName "Smith" →
        iv = [0x7F, 0x2C, ...] (different random IV!)
        AES-256-GCM encrypt
        STORED IN DB: last_name_enc = "fy8kXqRt...EncryptedSmith=="

      email "alice.smith@company.com" →
        iv = [0xD4, 0x91, ...]
        AES-256-GCM encrypt
        STORED IN DB: email_enc = "1Kp3Rv...EncryptedEmail=="

  → SQL INSERT fires:
      INSERT INTO employees (
          id, first_name_enc, last_name_enc, email_enc,
          department, position, joining_date, is_active,
          erased_at, consent_given, created_at, updated_at,
          created_by, updated_by
      ) VALUES (
          '550e8400-e29b-41d4-a716-446655440000',
          'oTsL9mNp...EncryptedAlice==',       ← encrypted
          'fy8kXqRt...EncryptedSmith==',        ← encrypted
          '1Kp3Rv...EncryptedEmail==',          ← encrypted
          'Engineering',                         ← plain text
          'Senior Developer',                    ← plain text
          '2024-03-10',
          TRUE,
          NULL,                                  ← not erased
          FALSE,                                 ← no consent yet
          '2024-03-10T10:05:23Z',
          '2024-03-10T10:05:23Z',
          'admin',
          'admin'
      )

  → INSERT succeeds ✅

─── LAYER 8: Hibernate Envers fires (SOX audit) ─────────────────────────────
  → Detects INSERT on @Audited Employee entity
  → Inserts revision into revinfo:
      INSERT INTO revinfo (revtstmp) VALUES (1710064523000)
      → rev = 1 (auto-generated)

  → Inserts version into employees_aud:
      INSERT INTO employees_aud (
          id, rev, revtype,
          first_name_enc, last_name_enc, email_enc,
          department, position, is_active,
          consent_given, created_by
      ) VALUES (
          '550e8400-...', 1, 0,     ← revtype 0 = INSERT
          'oTsL9mNp...EncryptedAlice==',
          'fy8kXqRt...EncryptedSmith==',
          '1Kp3Rv...EncryptedEmail==',
          'Engineering', 'Senior Developer', TRUE,
          FALSE, 'admin'
      )

  → @Transactional commits ✅

─── LAYER 9: AuditAspect finally block fires (async) ────────────────────────
  → jp.proceed() returned successfully
  → outcome = "SUCCESS"
  → writeAuditLogAsync() called on separate thread (@Async)
  → Does NOT block the response

  → Async thread writes to audit_log table:
      INSERT INTO audit_log (
          id, event_type, user_id, user_role,
          service_id, resource_type, action,
          outcome, timestamp
      ) VALUES (
          UUID(),
          'DATA_ACCESS',
          'admin',
          '[ROLE_ADMIN]',
          'emloyeems',
          'Employee',
          'CREATE',
          'SUCCESS',
          '2024-03-10T10:05:23Z'
      )

─── LAYER 10: Controller builds response ────────────────────────────────────
  → modelMapper.map(savedEmployee, EmployeeResponse.class)

  → EncryptedStringConverter.convertToEntityAttribute() fires on read:
      first_name_enc "oTsL9mNp...EncryptedAlice=="
        → base64 decode → split IV (12 bytes) + ciphertext
        → AES-256-GCM decrypt(ciphertext, key, iv)
        → "Alice" ← back to plain text for response ✅

      (same for lastName, email)

  → EmployeeResponse {
        id = "550e8400-e29b-41d4-a716-446655440000",
        firstName = "Alice",      ← decrypted ✅
        lastName  = "Smith",      ← decrypted ✅
        email     = "alice.smith@company.com",  ← decrypted ✅
        department = "Engineering",
        position   = "Senior Developer",
        joiningDate = "2024-03-10",
        createdAt  = "2024-03-10T10:05:23Z"
    }

RESPONSE: 201 CREATED
  X-Correlation-Id: a3f9-11bc-...
  {
    "success": true,
    "message": "Employee created successfully",
    "data": {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "firstName": "Alice",
      "lastName": "Smith",
      "email": "alice.smith@company.com",
      "department": "Engineering",
      "position": "Senior Developer",
      "joiningDate": "2024-03-10",
      "createdAt": "2024-03-10T10:05:23Z"
    },
    "timestamp": "2024-03-10T10:05:23Z"
  }

DATABASE STATE AFTER SCENARIO 2:
  employees:      1 row  (PII encrypted)
  employees_aud:  1 row  (SOX — revtype=0 INSERT)
  audit_log:      1 row  (action=CREATE, outcome=SUCCESS)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 3 — CREATE EMPLOYEE (Validation Failure)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REQUEST:
  POST /api/v1/employees
  Authorization: Bearer eyJ...
  Body: { "firstName": "", "email": "not-an-email", "department": "" }

─── LAYER: @Valid triggers on @RequestBody ──────────────────────────────────
  → firstName = "" → @NotBlank FAILS ❌
  → email = "not-an-email" → @Email FAILS ❌
  → lastName = null → @NotBlank FAILS ❌
  → department = "" → @NotBlank FAILS ❌
  → MethodArgumentNotValidException thrown
  → Controller method body NEVER executes
  → AuditAspect NEVER fires (no @Auditable reached)
  → No DB access at all

─── GlobalExceptionHandler.handleValidation() ───────────────────────────────
  → Collects all field errors

RESPONSE: 400 BAD REQUEST
  {
    "success": false,
    "message": "Validation failed: {firstName=must not be blank,
                lastName=must not be blank,
                email=must be a valid email address,
                department=must not be blank}",
    "timestamp": "2024-03-10T10:06:00Z"
  }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 4 — CREATE EMPLOYEE (Duplicate Email)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REQUEST:
  POST /api/v1/employees
  Body: { "email": "alice.smith@company.com", ... }  ← same email as above

─── EmployeeService.saveEmployee() ──────────────────────────────────────────
  → employeeRepository.existsByEmail("alice.smith@company.com")
  → EncryptedStringConverter encrypts email before query
  → SQL: SELECT COUNT(*) FROM employees WHERE email_enc = '1Kp3Rv...=='
  → Result: 1 (exists!) ❌
  → throws DuplicateEmailException("alice.smith@company.com")
  → AuditAspect finally block fires:
      outcome = "FAILURE"
      failureReason = "DuplicateEmailException: An employee with email..."
      audit_log: INSERT ... action=CREATE, outcome=FAILURE ✅

─── GlobalExceptionHandler.handleDuplicate() ────────────────────────────────

RESPONSE: 409 CONFLICT
  {
    "success": false,
    "message": "An employee with email 'alice.smith@company.com' already exists",
    "timestamp": "2024-03-10T10:07:00Z"
  }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 5 — GET ALL EMPLOYEES
 GET http://localhost:8080/api/v1/employees?page=0&size=20
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

─── EmployeeService.getAllEmployees(0, 20) ───────────────────────────────────
  → Pageable = page=0, size=20, sort=createdAt DESC

  → SQL:
      SELECT * FROM employees
      WHERE is_active = TRUE
      AND erased_at IS NULL          ← GDPR: erased employees invisible
      ORDER BY created_at DESC
      LIMIT 20 OFFSET 0

  → Returns 1 row (Alice)
  → EncryptedStringConverter.convertToEntityAttribute() fires on each PII field:
      first_name_enc → decrypt → "Alice"
      last_name_enc  → decrypt → "Smith"
      email_enc      → decrypt → "alice.smith@company.com"

  → Maps to Page<EmployeeResponse>

RESPONSE: 200 OK
  {
    "success": true,
    "message": "Employees retrieved",
    "data": {
      "content": [{
        "id": "550e8400-...",
        "firstName": "Alice",
        "lastName": "Smith",
        "email": "alice.smith@company.com",
        "department": "Engineering",
        "position": "Senior Developer"
      }],
      "totalElements": 1,
      "totalPages": 1,
      "size": 20,
      "number": 0
    }
  }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 6 — UPDATE EMPLOYEE
 PUT http://localhost:8080/api/v1/employees/550e8400-...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REQUEST:
  Body: { "firstName": "Alice", "lastName": "Johnson",
          "department": "Architecture", "position": "Principal Engineer" }

─── EmployeeService.updateEmployee() ────────────────────────────────────────
  → findActiveEmployee("550e8400-...")
  → SQL: SELECT * FROM employees WHERE id = '550e8400-...'
  → Employee found, isActive=true, erasedAt=null ✅

  → Sets only allowed fields:
      employee.setFirstName("Alice")       ← re-encrypts on save
      employee.setLastName("Johnson")      ← re-encrypts on save
      employee.setDepartment("Architecture")
      employee.setPosition("Principal Engineer")
      NOTE: email, id, createdAt, createdBy NOT touched ✅

  → employeeRepository.save(employee)

  → Spring Auditing:
      updatedAt = "2024-03-10T11:00:00Z"
      updatedBy = "admin"  ← from SecurityContext

  → EncryptedStringConverter re-encrypts PII on save:
      firstName "Alice" →
        NEW random IV → different ciphertext than before
        DB: first_name_enc = "Xk9pWr...newCiphertext=="
      lastName "Johnson" →
        NEW random IV → fresh ciphertext
        DB: last_name_enc = "Mn3Yz...newCiphertext=="

  → SQL UPDATE fires:
      UPDATE employees SET
        first_name_enc = 'Xk9pWr...==',
        last_name_enc  = 'Mn3Yz...==',
        department     = 'Architecture',
        position       = 'Principal Engineer',
        updated_at     = '2024-03-10T11:00:00Z',
        updated_by     = 'admin'
      WHERE id = '550e8400-...'

─── Hibernate Envers fires on UPDATE ────────────────────────────────────────
  → New revision created: rev = 2
  → INSERT INTO employees_aud (id, rev, revtype, ...)
    VALUES ('550e8400-...', 2, 1, ...)   ← revtype 1 = UPDATE
  → employees_aud now has 2 rows:
      rev=1, revtype=0 (INSERT) ← original
      rev=2, revtype=1 (UPDATE) ← this change
  → SOX: full history, who changed what, when ✅

─── AuditAspect fires (async) ───────────────────────────────────────────────
  → audit_log: action=UPDATE, outcome=SUCCESS, userId=admin ✅

RESPONSE: 200 OK
  {
    "data": {
      "firstName": "Alice",
      "lastName": "Johnson",      ← updated
      "department": "Architecture", ← updated
      "position": "Principal Engineer"
    }
  }

DATABASE STATE AFTER SCENARIO 6:
  employees:      1 row  (updated, PII re-encrypted with new IV)
  employees_aud:  2 rows (rev=1 INSERT, rev=2 UPDATE — SOX trail)
  audit_log:      3 rows (CREATE, GET, UPDATE)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 7 — RECORD CONSENT (GDPR)
 POST http://localhost:8080/api/v1/employees/550e8400-.../consent
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REQUEST:
  POST /api/v1/employees/550e8400-.../consent
        ?version=v1.0&ipAddress=192.168.1.100
  Authorization: Bearer eyJ...

─── ConsentService.recordConsent() ──────────────────────────────────────────
  → Finds employee "550e8400-..."
  → employee.setConsentGiven(true)
  → employee.setConsentGivenAt("2024-03-10T11:30:00Z")
  → employee.setConsentVersion("v1.0")
  → employeeRepository.save(employee)

  → SQL UPDATE:
      UPDATE employees SET
        consent_given    = TRUE,
        consent_given_at = '2024-03-10T11:30:00Z',
        consent_version  = 'v1.0',
        updated_by       = 'admin'
      WHERE id = '550e8400-...'

  → Envers fires: rev=3, revtype=1 (UPDATE) ← consent recorded in audit trail
  → AuditAspect: action=CREATE on ConsentRecord ✅

RESPONSE: 200 OK
  { "success": true, "message": "Consent recorded" }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 8 — DELETE EMPLOYEE (Soft Delete)
 DELETE http://localhost:8080/api/v1/employees/550e8400-...
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

─── EmployeeService.deleteEmployee() ────────────────────────────────────────
  → findActiveEmployee("550e8400-...") ✅
  → employee.setIsActive(false)   ← SOFT DELETE only
  → employeeRepository.save(employee)

  → SQL UPDATE:
      UPDATE employees SET is_active = FALSE, updated_by = 'admin'
      WHERE id = '550e8400-...'
  → NO DELETE SQL ever fires ✅

  → Envers fires: rev=4, revtype=1 (UPDATE — isActive=false recorded)
  → AuditAspect: action=DELETE, outcome=SUCCESS

RESPONSE: 204 NO CONTENT  (no body)

VERIFY — Try to GET deleted employee:
  GET /api/v1/employees/550e8400-...
  → findActiveEmployee filters: WHERE isActive=TRUE AND erasedAt IS NULL
  → isActive = FALSE → filter fails
  → throws EmployeeNotFoundException
  → 404 NOT FOUND ✅
  (Employee still EXISTS in DB — just invisible to API)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 9 — GDPR RIGHT TO ERASURE
 DELETE http://localhost:8080/api/v1/employees/550e8400-.../erase
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  NOTE: Let's create a new active employee "Bob" first for this scenario.
  Bob's id = "660e8400-e29b-41d4-a716-446655441111"

REQUEST:
  DELETE /api/v1/employees/660e8400-.../erase?requestId=DSAR-2024-001
  Authorization: Bearer eyJ... (ADMIN role required)

─── ErasureService.processErasureRequest() ──────────────────────────────────
  → findById("660e8400-...") → Employee Bob found ✅
  → erasedAt == null → not already erased → proceed ✅

  STEP 1: pseudonymizeAllPiiFields(employee) via reflection:

    Iterates all Employee.class.getDeclaredFields():

    Field: "id" (String)
      → @PersonalData? NO → skip ✅

    Field: "firstName" (String)
      → @PersonalData? YES
          classification = PII
          erasable = true  ✅
      → field.set(employee, "ERASED-a1b2c3d4")
      → firstName is now "ERASED-a1b2c3d4"

    Field: "lastName" (String)
      → @PersonalData? YES, erasable=true ✅
      → field.set(employee, "ERASED-e5f6g7h8")

    Field: "email" (String)
      → @PersonalData? YES, erasable=true ✅
      → field.set(employee, "ERASED-i9j0k1l2@erased.invalid")

    Field: "department" (String)
      → @PersonalData? NO → skip ✅

    Field: "position" (String)
      → @PersonalData? NO → skip ✅

    (If nationalId field existed with erasable=false → SKIP + log)

  STEP 2: stamp erasure
    → employee.setErasedAt("2024-03-10T14:00:00Z")
    → employee.setConsentGiven(false)
    → employee.setIsActive(false)

  STEP 3: employeeRepository.save(employee)

  → EncryptedStringConverter encrypts pseudonyms:
      "ERASED-a1b2c3d4" → AES-256-GCM → stored as ciphertext
      "ERASED-i9j0k1l2@erased.invalid" → AES-256-GCM → stored

  → SQL UPDATE:
      UPDATE employees SET
        first_name_enc = '[encrypted ERASED-a1b2c3d4]',
        last_name_enc  = '[encrypted ERASED-e5f6g7h8]',
        email_enc      = '[encrypted ERASED-i9j0k1l2@erased.invalid]',
        is_active      = FALSE,
        erased_at      = '2024-03-10T14:00:00Z',
        consent_given  = FALSE,
        updated_by     = 'admin'
      WHERE id = '660e8400-...'

  → Envers fires: rev=N, revtype=1 (UPDATE with erased values recorded)

  STEP 4: Save ErasureRecord (GDPR proof of erasure)
    → INSERT INTO erasure_records (
          id, employee_id, erased_at, erasure_type, status
      ) VALUES (
          'DSAR-2024-001',
          '660e8400-...',          ← reference only — no PII here
          '2024-03-10T14:00:00Z',
          'GDPR_REQUEST',
          'COMPLETED'
      )

  → AuditAspect fires: action=ERASURE, outcome=SUCCESS ✅

RESPONSE: 200 OK
  {
    "success": true,
    "message": "GDPR erasure completed — employee data pseudonymized",
    "data": {
      "id": "DSAR-2024-001",
      "erasedAt": "2024-03-10T14:00:00Z",
      "status": "COMPLETED"
    }
  }

VERIFY — All future reads of Bob return 404:
  GET /api/v1/employees/660e8400-...
  → filter: erasedAt IS NULL → erasedAt = "2024-03-10T14:00:00Z" → FILTERED OUT
  → 404 NOT FOUND ✅

GET /api/v1/employees (list)
  → SQL WHERE is_active=TRUE AND erased_at IS NULL
  → Bob's row excluded ✅

DATABASE STATE AFTER SCENARIO 9:
  employees:       Bob's row exists but all PII = pseudonyms, erased_at set
  employees_aud:   Full version history of Bob (every change preserved) — SOX ✅
  erasure_records: 1 row — DSAR-2024-001 — proof of erasure — GDPR ✅
  audit_log:       ERASURE event recorded ✅

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 10 — UNAUTHORIZED ACCESS (No JWT)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REQUEST:
  GET /api/v1/employees
  (No Authorization header)

─── JwtAuthenticationFilter ─────────────────────────────────────────────────
  → Authorization header MISSING
  → URL does NOT match /api/v1/auth/** or /actuator/health
  → filter calls chain.doFilter() without setting SecurityContext

─── Spring Security Authorization ───────────────────────────────────────────
  → anyRequest().authenticated() → SecurityContext has no auth → BLOCKED ❌
  → Returns 401 UNAUTHORIZED

RESPONSE: 401 UNAUTHORIZED
  { "error": "Unauthorized" }

  → Controller NEVER reached
  → Service NEVER reached
  → Database NEVER touched
  → No audit log written (no identity to log)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 SCENARIO 11 — WRONG ROLE (HR trying to delete)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

REQUEST:
  DELETE /api/v1/employees/550e8400-...
  Authorization: Bearer [HR user token]

─── JwtAuthenticationFilter ─────────────────────────────────────────────────
  → Token valid ✅
  → SecurityContext set with authorities = [ROLE_HR]

─── @PreAuthorize("hasRole('ADMIN')") on deleteEmployee() ───────────────────
  → ROLE_HR does not match ROLE_ADMIN ❌
  → AccessDeniedException thrown
  → Controller method body NEVER executes

RESPONSE: 403 FORBIDDEN
  { "error": "Forbidden" }

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 FINAL DATABASE STATE AFTER ALL SCENARIOS
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

TABLE: employees
  Row 1 (Alice):  isActive=FALSE, erasedAt=NULL    ← soft deleted, PII intact
  Row 2 (Bob):    isActive=FALSE, erasedAt=set      ← GDPR erased, PII=pseudonyms

TABLE: employees_aud (SOX immutable history)
  Alice: rev=1(INSERT), rev=2(UPDATE dept), rev=3(consent), rev=4(soft-delete)
  Bob:   rev=5(INSERT), rev=6(GDPR-erase)

TABLE: audit_log (SOX access log)
  6 rows: CREATE, READ, UPDATE, consent CREATE,
          DELETE (soft), ERASURE — all with userId=admin

TABLE: erasure_records (GDPR proof)
  1 row: DSAR-2024-001, Bob, COMPLETED

TABLE: revinfo
  6 rows: one per Envers revision, each with timestamp

WHAT API RETURNS NOW:
  GET /api/v1/employees → [] (empty — both inactive/erased)
  GET /api/v1/employees/Alice-id → 404
  GET /api/v1/employees/Bob-id   → 404
  (Both exist in DB but invisible to API ✅)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 COMPLIANCE EVIDENCE SUMMARY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

SOX AUDITOR asks:  "Show me all changes to Alice's record"
  → Query employees_aud WHERE id = 'Alice-id' ORDER BY rev
  → Returns 4 versions: created by admin, updated dept,
    consent recorded, soft-deleted — complete timeline ✅

SOX AUDITOR asks:  "Who accessed employee data on March 10?"
  → Query audit_log WHERE timestamp BETWEEN '2024-03-10' AND '2024-03-11'
  → Returns 6 rows: all by admin ✅

GDPR REGULATOR asks: "Prove Bob's data was erased"
  → Show erasure_records WHERE employee_id = 'Bob-id'
  → DSAR-2024-001, COMPLETED, 2024-03-10T14:00:00Z ✅
  → Show employees WHERE id = 'Bob-id'
  → first_name = 'ERASED-a1b2c3d4' ✅

GDPR REGULATOR asks: "Is Bob's PII in your audit logs?"
  → employees_aud for Bob shows encrypted pseudonyms only
  → audit_log has no PII — only IDs and action metadata ✅

SECURITY TEAM asks: "What's in the employees table in MySQL?"
  → first_name_enc = 'oTsL9mNp...=='  (ciphertext — unreadable) ✅
  → Anyone with DB access sees only ciphertext,
    never plain text names or emails ✅
