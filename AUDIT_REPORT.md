# Workspace Audit Report

## Executive summary

The workspace has a substantial implementation: three Spring Boot services, MongoDB persistence, JWT authentication, seller ownership checks, filesystem-backed media, Kafka cleanup events, an Angular Material frontend, and Docker Compose. Core controllers and UI flows are connected coherently.

It does not yet satisfy the required architecture or delivery bar:

- No API Gateway exists. Nginx only proxies frontend requests and performs no authentication.
- No service-discovery implementation exists.
- Avatar upload/update is not implemented end to end.
- Media validation trusts the client-provided MIME type and accepts disguised non-images.
- Product image IDs are not checked against the authenticated seller.
- The frontend test suite has definite TypeScript compilation failures.
- Backend tests are only context-load skeletons; Media Service has none.
- HTTPS, production CORS, robust deployment documentation, and failure recovery are incomplete.

`docker compose config --quiet` parsed successfully with a temporary audit-only secret. Builds/tests were not run because they create build artifacts, and no files were changed during the audit.

## Checklist results

### 1. Architecture

| Area | Requirement | Status | Evidence | Gap / next action |
|---|---|---:|---|---|
| Architecture | Three independently structured Spring Boot services | PASS | Entry points: [`Buy01Application.java`](user-service/src/main/java/com/example/userservice/Buy01Application.java#L10), [`ProductServiceApplication.java`](product-service/src/main/java/com/example/productservice/ProductServiceApplication.java#L9), [`MediaServiceApplication.java`](media-service/src/main/java/com/example/mediaservice/MediaServiceApplication.java#L9); separate POMs and Dockerfiles | — |
| Architecture | API Gateway | MISSING | Compose defines Mongo, Kafka, the three services, and frontend only: [`docker-compose.yml`](docker-compose.yml#L1). Nginx proxies paths but is not a Spring/API gateway: [`nginx.conf`](frontend/nginx.conf#L12) | Add a gateway service, route definitions, JWT enforcement, CORS, correlation IDs, and deployment wiring. |
| Architecture | Service discovery | MISSING | No Eureka/Consul/discovery module, dependency, registration config, or Compose service exists | Add a discovery server and clients, or document an approved equivalent if fixed orchestration DNS is intentionally accepted. |
| Architecture | MongoDB persistence | PASS | Mongo repositories exist in all services; URIs are configured in each `application.yml` and Compose. `spring.mongodb.uri` is the correct Boot 4 property according to [Spring Boot's property reference](https://docs.spring.io/spring-boot/appendix/application-properties/index.html). | — |
| Architecture | Images outside database | PASS | Mongo stores only metadata and `storageKey`: [`MediaAsset.java`](media-service/src/main/java/com/example/mediaservice/model/MediaAsset.java#L15). Bytes go through filesystem storage: [`LocalFileStorageService.java`](media-service/src/main/java/com/example/mediaservice/service/LocalFileStorageService.java#L24) | — |
| Architecture | Every service exposes `/actuator/health` | PASS | Actuator dependencies and exposure exist in all three POM/YAML files; routes are explicitly permitted in [`user SecurityConfig`](user-service/src/main/java/com/example/userservice/security/SecurityConfig.java#L64), [`product SecurityConfig`](product-service/src/main/java/com/example/productservice/security/SecurityConfig.java#L44), and [`media SecurityConfig`](media-service/src/main/java/com/example/mediaservice/security/SecurityConfig.java#L46) | — |
| Architecture | Docker Compose/runnable deployment | PARTIAL | Compose parses and has health checks, dependency ordering, and persistent Mongo/media volumes: [`docker-compose.yml`](docker-compose.yml#L5) | Runtime was not started. Gateway/discovery are absent; services are directly exposed. Add full-stack smoke testing. |
| Architecture | Kafka events | PARTIAL *(optional)* | Product events: [`ProductEventProducer.java`](product-service/src/main/java/com/example/productservice/kafka/ProductEventProducer.java#L21); image events: [`ImageEventProducer.java`](media-service/src/main/java/com/example/mediaservice/kafka/ImageEventProducer.java#L22); cleanup consumers exist | No Kafka tests, DLQ, retry policy, or outbox. A successful DB write can permanently lose its cleanup event. |

### 2. User Service and authentication

| Area | Requirement | Status | Evidence | Gap / next action |
|---|---|---:|---|---|
| User | `POST /auth/register`, CLIENT/SELLER, validation, duplicate prevention | PASS | Route: [`AuthController.java`](user-service/src/main/java/com/example/userservice/controller/AuthController.java#L25); constraints: [`RegisterRequest.java`](user-service/src/main/java/com/example/userservice/dto/RegisterRequest.java#L9); roles: [`UserRole.java`](user-service/src/main/java/com/example/userservice/model/UserRole.java#L3); duplicate checks and unique indexes: [`UserService.java`](user-service/src/main/java/com/example/userservice/service/UserService.java#L44), [`User.java`](user-service/src/main/java/com/example/userservice/model/User.java#L26) | — |
| User | `POST /auth/login` returns JWT | PASS | Email/password validation and route: [`LoginRequest.java`](user-service/src/main/java/com/example/userservice/dto/LoginRequest.java#L6), [`AuthController.java`](user-service/src/main/java/com/example/userservice/controller/AuthController.java#L36); token creation: [`JwtService.java`](user-service/src/main/java/com/example/userservice/security/JwtService.java#L29) | — |
| User | `GET /me` returns only authenticated profile | PASS | Subject is taken from validated JWT and mapped to a password-free DTO: [`UserController.java`](user-service/src/main/java/com/example/userservice/controller/UserController.java#L25), [`UserProfileResponse.java`](user-service/src/main/java/com/example/userservice/dto/UserProfileResponse.java#L8) | — |
| User | `PUT /me` updates only authenticated profile | PASS | Uses `jwt.getSubject()` rather than a client-supplied ID: [`UserController.java`](user-service/src/main/java/com/example/userservice/controller/UserController.java#L36), [`UserService.java`](user-service/src/main/java/com/example/userservice/service/UserService.java#L87) | — |
| User | Seller avatar delegated to Media Service | PARTIAL | User model stores only `avatarUrl`, but `PUT /me` accepts any arbitrary string: [`UpdateProfileRequest.java`](user-service/src/main/java/com/example/userservice/dto/UpdateProfileRequest.java#L5), [`UserService.java`](user-service/src/main/java/com/example/userservice/service/UserService.java#L105). No frontend profile/avatar flow exists. | Add an authenticated avatar UI and verify the submitted media ID/URL belongs to the seller. |
| User | BCrypt hashing and salting | PASS | BCrypt cost 12 bean: [`SecurityConfig.java`](user-service/src/main/java/com/example/userservice/security/SecurityConfig.java#L142); registration hashes before save: [`UserService.java`](user-service/src/main/java/com/example/userservice/service/UserService.java#L56) | — |
| User | Password never returned/logged/persisted in frontend session | PASS | API responses exclude password; logs use IDs only: [`UserService.java`](user-service/src/main/java/com/example/userservice/service/UserService.java#L63). Frontend stores token plus username/role only: [`auth.ts`](frontend/src/app/shared/services/auth.ts#L61) | Avoid logging the Lombok-generated `User.toString()`, which would include the password hash. |
| User | JWT validation actually enforced | PASS | Stateless OAuth2 resource-server decoder is installed: [`user SecurityConfig`](user-service/src/main/java/com/example/userservice/security/SecurityConfig.java#L59); Product and Media independently validate the shared HS256 signature | Consider additionally enforcing the expected issuer and audience. |
| User | CLIENT/SELLER roles modeled and checked | PASS | Role claim generated at [`JwtService.java`](user-service/src/main/java/com/example/userservice/security/JwtService.java#L33); product write checks at [`ProductController.java`](product-service/src/main/java/com/example/productservice/controller/ProductController.java#L42); media uses `hasRole`: [`MediaController.java`](media-service/src/main/java/com/example/mediaservice/controller/MediaController.java#L37) | — |
| User | SELLER manages only own products/media | PASS | Product lookup includes `sellerId`: [`ProductService.java`](product-service/src/main/java/com/example/productservice/service/ProductService.java#L60); media delete compares JWT subject to owner: [`MediaService.java`](media-service/src/main/java/com/example/mediaservice/service/MediaService.java#L92) | Image association itself still needs ownership validation; see Product findings. |
| User | ADMIN | N/A *(optional)* | Enum contains CLIENT and SELLER only | Deliberately not implemented. |

### 3. Product Service

| Area | Requirement | Status | Evidence | Gap / next action |
|---|---|---:|---|---|
| Product | Public `GET /products` and `GET /products/{id}` | PASS | [`ProductController.java`](product-service/src/main/java/com/example/productservice/controller/ProductController.java#L25); GET rules are public: [`SecurityConfig.java`](product-service/src/main/java/com/example/productservice/security/SecurityConfig.java#L44) | — |
| Product | Seller POST/PUT/DELETE | PASS | Routes and seller checks: [`ProductController.java`](product-service/src/main/java/com/example/productservice/controller/ProductController.java#L42) | — |
| Product | Required fields, price > 0, invalid input → 400 | PASS | Constraints: [`CreateProductRequest.java`](product-service/src/main/java/com/example/productservice/dto/CreateProductRequest.java#L12), [`UpdateProductRequest.java`](product-service/src/main/java/com/example/productservice/dto/UpdateProductRequest.java#L9); handler: [`GlobalExceptionHandler.java`](product-service/src/main/java/com/example/productservice/exception/GlobalExceptionHandler.java#L20) | Update values consisting only of spaces are silently ignored rather than rejected. Tighten with a custom nonblank validator if required. |
| Product | Seller identity from authenticated token | PASS | `sellerId` is never in request DTOs; create uses `jwt.getSubject()`: [`ProductController.java`](product-service/src/main/java/com/example/productservice/controller/ProductController.java#L43), [`ProductService.java`](product-service/src/main/java/com/example/productservice/service/ProductService.java#L26) | — |
| Product | Cross-seller update/delete blocked; intentional 404/403 | PASS | `findByIdAndSellerId` and masked 404: [`ProductRepository.java`](product-service/src/main/java/com/example/productservice/repository/ProductRepository.java#L14), [`ProductService.java`](product-service/src/main/java/com/example/productservice/service/ProductService.java#L60) | Add security tests proving this behavior. |
| Product | Images stored as Media references, not binaries | PASS | Product holds `List<String> imageUrls`: [`Product.java`](product-service/src/main/java/com/example/productservice/model/Product.java#L41). Frontend treats entries as media IDs: [`product-media.ts`](frontend/src/app/seller/pages/product-media/product-media.ts#L58) | Rename to `imageIds` or consistently store full URLs to remove contract ambiguity. |
| Product | Associate and remove images during create/edit | PARTIAL | Create/update replace the reference list: [`ProductService.java`](product-service/src/main/java/com/example/productservice/service/ProductService.java#L25). UI persists IDs: [`product-media.ts`](frontend/src/app/seller/pages/product-media/product-media.ts#L145) | Product Service accepts arbitrary media IDs without confirming existence or seller ownership. Add a trusted Media Service check or signed ownership contract. |
| Product | Product calls routed through gateway | FAIL | Development calls service port directly: [`environment.ts`](frontend/src/environments/environment.ts#L1). Production calls Nginx paths: [`environment.prod.ts`](frontend/src/environments/environment.prod.ts#L1), not an API Gateway | Introduce and use the required gateway in all environments. |

### 4. Media Service

| Area | Requirement | Status | Evidence | Gap / next action |
|---|---|---:|---|---|
| Media | `POST /media/images` seller-only | PASS | Controller and method authorization: [`MediaController.java`](media-service/src/main/java/com/example/mediaservice/controller/MediaController.java#L37); route-level rule: [`SecurityConfig.java`](media-service/src/main/java/com/example/mediaservice/security/SecurityConfig.java#L50) | — |
| Media | MIME allowlist | PASS | JPEG, PNG, and WEBP allowlist: [`MediaService.java`](media-service/src/main/java/com/example/mediaservice/service/MediaService.java#L30) | — |
| Media | Maximum size 2 MB | PASS | Explicit byte check: [`MediaService.java`](media-service/src/main/java/com/example/mediaservice/service/MediaService.java#L114); multipart limit: [`application.yml`](media-service/src/main/resources/application.yml#L13) | — |
| Media | Safe filename handling | PASS | UUID prefix and character sanitization prevent traversal: [`LocalFileStorageService.java`](media-service/src/main/java/com/example/mediaservice/service/LocalFileStorageService.java#L25) | Consider dropping the original filename entirely from the storage name. |
| Media | Content sniffing / disguised non-image rejection | FAIL | Validation only checks `MultipartFile.getContentType()`: [`MediaService.java`](media-service/src/main/java/com/example/mediaservice/service/MediaService.java#L118) | Decode the image or validate magic bytes with a maintained parser; derive the stored response content type from detected content. |
| Media | `GET /media/images/{id}` serves bytes | PASS | Metadata and stream are retrieved and returned: [`MediaController.java`](media-service/src/main/java/com/example/mediaservice/controller/MediaController.java#L47) | — |
| Media | Content type and cache headers | PASS | Content type, length, one-hour cache policy, and disposition are set: [`MediaController.java`](media-service/src/main/java/com/example/mediaservice/controller/MediaController.java#L52) | Content type becomes trustworthy only after content sniffing is added. |
| Media | Optional DELETE, owning seller only | PASS *(optional)* | [`MediaController.java`](media-service/src/main/java/com/example/mediaservice/controller/MediaController.java#L63), [`MediaService.java`](media-service/src/main/java/com/example/mediaservice/service/MediaService.java#L93) | — |
| Media | Ownership stored and enforced | PASS | `sellerId` stored from JWT subject: [`MediaAsset.java`](media-service/src/main/java/com/example/mediaservice/model/MediaAsset.java#L22), [`MediaService.java`](media-service/src/main/java/com/example/mediaservice/service/MediaService.java#L49). Public retrieval is explicitly configured. | Add tests for seller A deleting seller B's media. |
| Media | Storage and orphan cleanup | PARTIAL | Persistent volume: [`docker-compose.yml`](docker-compose.yml#L99); product-deletion consumer: [`ProductEventConsumer.java`](media-service/src/main/java/com/example/mediaservice/kafka/ProductEventConsumer.java#L20) | DB-save failure after file creation and product-update failure after upload leave orphans. Add compensation/reconciliation and document retention. |
| Media | Limit at application/gateway/proxy layers | PARTIAL | Backend is 2 MB, but Nginx allows 3 MB: [`nginx.conf`](frontend/nginx.conf#L28) | Align proxy/gateway policy with the 2 MB requirement while preserving a useful 400/413 response. |

A separate URL defect exists: Compose sets `MEDIA_PUBLIC_BASE_URL=http://localhost:8083/media` in [`docker-compose.yml`](docker-compose.yml#L100), and the service appends `/{id}` in [`MediaService.java`](media-service/src/main/java/com/example/mediaservice/service/MediaService.java#L59). The resulting `/media/{id}` URL does not match `/media/images/{id}`. The current frontend happens to ignore this URL and reconstruct the correct one.

### 5. Gateway and security

| Area | Requirement | Status | Evidence | Gap / next action |
|---|---|---:|---|---|
| Gateway | Routes public product GETs | MISSING | Nginx has a generic product proxy at [`nginx.conf`](frontend/nginx.conf#L20), but no required gateway exists | Add gateway route predicates for public GETs. |
| Gateway | Protects authenticated/write routes | MISSING | Nginx contains no auth logic; authorization exists only in downstream services | Validate JWT and roles at the gateway while retaining downstream defense in depth. |
| Gateway | Validate and safely propagate auth context | PARTIAL | Nginx forwards request headers by default and downstream services validate JWTs, but there is no gateway validation or trusted identity-header strategy | Keep bearer-token propagation or use signed internal identity headers after gateway validation. |
| Security | Production CORS restricted | PARTIAL | All services allow `http://localhost:*` and all headers with credentials, e.g. [`user SecurityConfig`](user-service/src/main/java/com/example/userservice/security/SecurityConfig.java#L94) | Externalize exact allowed origins per environment; expose APIs only through the gateway. |
| Security | Secrets externalized | PASS | `${JWT_SECRET}` is required: [`application.yml`](user-service/src/main/resources/application.yml#L21); Compose fail-fast substitution: [`docker-compose.yml`](docker-compose.yml#L57) | Prefer asymmetric signing or a managed secret store for production. |
| Security | HTTPS present/documented | MISSING | Nginx listens on HTTP port 80 only: [`nginx.conf`](frontend/nginx.conf#L1); README contains no TLS deployment instructions | Add TLS termination, redirect HTTP to HTTPS, and document internal/external TLS policy. |
| Security | Auth/media rate limiting | N/A *(optional)* | No limiter found | Add at the gateway if exposed beyond a classroom/local environment. |
| Security | Does not accidentally permit all endpoints | PASS | Explicit public matchers are followed by `.anyRequest().authenticated()` in every service | — |
| Security | Errors do not expose stack traces/secrets | PASS | `include-stacktrace: never`; catch-all handlers return safe messages, e.g. [`product application.yml`](product-service/src/main/resources/application.yml#L38), [`product handler`](product-service/src/main/java/com/example/productservice/exception/GlobalExceptionHandler.java#L51) | — |

### 6. Error handling and reliability

| Area | Requirement | Status | Evidence | Gap / next action |
|---|---|---:|---|---|
| Reliability | Global exception handling | PASS | Advice classes exist in User, Product, and Media services; example: [`media GlobalExceptionHandler.java`](media-service/src/main/java/com/example/mediaservice/exception/GlobalExceptionHandler.java#L20) | — |
| Reliability | Meaningful 400/401/403/404 statuses | PASS | Validation, auth entry points, role checks, not-found exceptions, and upload-limit handlers are explicitly mapped | Add automated tests to prevent regressions. |
| Reliability | No obvious expected failures become generic 500 | PARTIAL | Storage I/O and missing files on disk become generic `IllegalStateException`/500: [`MediaService.java`](media-service/src/main/java/com/example/mediaservice/service/MediaService.java#L83) | Distinguish unavailable/corrupt storage, map expected conditions, and include correlation IDs. |
| Reliability | Useful, consistent error bodies | PARTIAL | Product/media use `error` and `message`; User validation instead puts the message in `error`: [`user GlobalExceptionHandler.java`](user-service/src/main/java/com/example/userservice/exception/GlobalExceptionHandler.java#L103). This conflicts with [`README.md`](README.md#L125). | Define one shared error schema and validation-details convention. |
| Reliability | Health checks usable | PASS | Health endpoints are exposed and Compose probes them: [`docker-compose.yml`](docker-compose.yml#L63) | Add readiness/liveness groups if deployment needs dependency-aware probes. |
| Reliability | Coherent startup/dependency configuration | PARTIAL | Compose waits for Mongo/Kafka health, but lacks gateway/discovery. Kafka admin/startup behavior was not exercised. | Add a full-stack smoke test and document startup timing/failure behavior. |
| Reliability | DB/downstream failure handling | PARTIAL | Catch-all responses are safe and Kafka send failure is logged, but there is no retry queue/outbox, circuit breaker, or compensation for file/DB divergence | Add reconciliation and reliable event publishing; define degraded behavior. |
| Reliability | Useful, sensitive-safe logging | PASS | Logs use resource IDs and roles rather than credentials; User has correlation IDs: [`CorrelationIdFilter.java`](user-service/src/main/java/com/example/userservice/config/CorrelationIdFilter.java#L23) | Add the same correlation filter to Product/Media and ensure Authorization headers are never logged. |

### 7. Angular frontend

| Area | Requirement | Status | Evidence | Gap / next action |
|---|---|---:|---|---|
| Frontend | Sign-in wired to real API | PASS | Reactive form calls `/auth/login`: [`login.ts`](frontend/src/app/auth/pages/login/login.ts#L49), [`auth.ts`](frontend/src/app/shared/services/auth.ts#L28) | Login error text says “username” although the API uses email. |
| Frontend | Sign-up with CLIENT/SELLER selection | PASS | Role toggle: [`register.html`](frontend/src/app/auth/pages/register/register.html#L27); API call: [`register.ts`](frontend/src/app/auth/pages/register/register.ts#L59) | — |
| Frontend | Avatar upload/update | MISSING | No profile route/component, `/me` service methods, or avatar UI exists | Add a protected profile page using Media Service upload followed by `PUT /me`. |
| Frontend | Public responsive product listing | PASS | Public route: [`catalog-routing-module.ts`](frontend/src/app/catalog/catalog-routing-module.ts#L7); loading/grid/error states: [`product-list.ts`](frontend/src/app/catalog/pages/product-list/product-list.ts#L23), [`product-list.html`](frontend/src/app/catalog/pages/product-list/product-list.html#L8) | `assets/placeholder.png` is referenced but absent from `frontend/public`. |
| Frontend | Product detail | PASS | Loads the API record and renders media IDs: [`product-detail.ts`](frontend/src/app/catalog/pages/product-detail/product-detail.ts#L24), [`product-detail.html`](frontend/src/app/catalog/pages/product-detail/product-detail.html#L23) | — |
| Frontend | Seller dashboard CRUD | PASS | Own-product API, edit/delete/media actions: [`dashboard.ts`](frontend/src/app/seller/pages/dashboard/dashboard.ts#L41), [`dashboard.html`](frontend/src/app/seller/pages/dashboard/dashboard.html#L47) | — |
| Frontend | Attach, preview, remove images | PASS | Upload, preview, product update, and deletion are connected: [`product-media.ts`](frontend/src/app/seller/pages/product-media/product-media.ts#L79), [`product-media.html`](frontend/src/app/seller/pages/product-media/product-media.html#L23) | Creation is two-stage: create the product first, then navigate to media management. |
| Frontend | Dedicated media-management view | PASS | `/seller/products/:id/media`: [`seller-routing-module.ts`](frontend/src/app/seller/seller-routing-module.ts#L7) | It is per-product; there is no seller-wide media library or orphan listing. |
| Frontend | File type and 2 MB checks | PASS | [`product-media.ts`](frontend/src/app/seller/pages/product-media/product-media.ts#L158) | Client checks remain bypassable as expected. |
| Frontend | Backend validation authoritative | PARTIAL | Backend repeats size/declared-type validation, but not byte-level validation | Add content sniffing server-side. |
| Frontend | Reactive Forms | PASS | Login, registration, and product form use `FormBuilder` and `ReactiveFormsModule` | — |
| Frontend | Inline required/price validation | PASS | [`product-form.ts`](frontend/src/app/seller/pages/product-form/product-form.ts#L28), [`product-form.html`](frontend/src/app/seller/pages/product-form/product-form.html#L21) | Frontend limits 100/1000 while backend allows 120/2000; align contracts. |
| Frontend | Useful upload/forbidden/server feedback | PARTIAL | Components use snackbars, but many server errors collapse to generic messages; 403 is silently redirected by the interceptor: [`auth-token-interceptor.ts`](frontend/src/app/shared/interceptors/auth-token-interceptor.ts#L17) | Preserve safe API messages and show a forbidden/session-expired notification. |
| Frontend | Routing configured | PASS | Lazy catalog/auth/seller routes and fallback: [`app-routing-module.ts`](frontend/src/app/app-routing-module.ts#L6) | — |
| Frontend | AuthGuard | PASS | [`auth-guard.ts`](frontend/src/app/shared/guards/auth-guard.ts#L16) | — |
| Frontend | RoleGuard | PASS | [`role-guard.ts`](frontend/src/app/shared/guards/role-guard.ts#L5) | — |
| Frontend | Interceptor attaches JWT | PASS | [`auth-token-interceptor.ts`](frontend/src/app/shared/interceptors/auth-token-interceptor.ts#L7) | Scope token attachment to known API origins to prevent future accidental token leakage. |
| Frontend | Interceptor handles 401/403 | PASS | 401 clears session and redirects; 403 redirects home: [`auth-token-interceptor.ts`](frontend/src/app/shared/interceptors/auth-token-interceptor.ts#L18) | Add visible feedback for 403. |
| Frontend | Secure token handling/logout | PARTIAL | Expiry is checked and logout clears state, but bearer tokens are stored in `localStorage`: [`auth.ts`](frontend/src/app/shared/services/auth.ts#L42) | Add a strict CSP and XSS hardening, or use short-lived access tokens with a protected refresh-token design. |
| Frontend | Material/responsive UI actually used | PASS | Material modules are imported: [`app-module.ts`](frontend/src/app/app-module.ts#L8); responsive grid/styles: [`styles.scss`](frontend/src/styles.scss#L159) | — |

There is also an API-model mismatch: the frontend sends and displays `category`, but the backend Product model/DTOs do not contain it. It is silently dropped during save.

### 8. Tests, documentation, and delivery

| Area | Requirement | Status | Evidence | Gap / next action |
|---|---|---:|---|---|
| Tests | Backend unit/integration/security tests | PARTIAL | Only two empty context-load tests exist: [`Buy01ApplicationTests.java`](user-service/src/test/java/com/example/userservice/Buy01ApplicationTests.java#L6), [`ProductServiceApplicationTests.java`](product-service/src/test/java/com/example/productservice/ProductServiceApplicationTests.java#L6); Media has no tests | Add isolated service tests and MockMvc/Testcontainers integration tests. |
| Tests | Authentication and role tests | MISSING | No register/login/JWT/401/403 tests found | Test invalid roles, invalid/expired signatures, login failures, and CLIENT/SELLER authorization. |
| Tests | Product/media ownership tests | MISSING | No ownership tests found | Prove seller A cannot mutate seller B's product/media. |
| Tests | Invalid product/upload tests | MISSING | No validation, oversize, content-type, or disguised-image tests found | Add boundary and malicious-payload tests. |
| Tests | Frontend component/service/guard/interceptor tests | FAIL | Several specs cannot compile: nonexistent `Auth` in [`auth.spec.ts`](frontend/src/app/shared/services/auth.spec.ts#L3), nonexistent `Media` in [`media.spec.ts`](frontend/src/app/shared/services/media.spec.ts#L3), interface `Product` injected as a runtime value in [`product.spec.ts`](frontend/src/app/shared/services/product.spec.ts#L3), and nonexistent functional `roleGuard` in [`role-guard.spec.ts`](frontend/src/app/shared/guards/role-guard.spec.ts#L4). App spec expects obsolete text: [`app.spec.ts`](frontend/src/app/app.spec.ts#L23) | Repair compilation, configure dependencies correctly, and test actual behavior rather than creation only. |
| Delivery | Build/lint/test commands defined | PARTIAL | Frontend has build/test scripts: [`package.json`](frontend/package.json#L4); Maven wrappers exist | No lint script, root task runner, or CI configuration. |
| README | Architecture overview | PARTIAL | [`README.md`](README.md#L1) lists services and Kafka flows but omits the required gateway/discovery and overstates a uniform error contract | Document actual topology and explicitly list missing components. |
| README | Prerequisites | MISSING | Root README has no consolidated Java/Node/Docker/Maven requirements | Add tested version requirements. |
| README | Environment, local startup, Compose | PARTIAL | Environment table and Compose command exist: [`README.md`](README.md#L149) | Add frontend/local commands, startup order, health verification, and troubleshooting. |
| README | Service URLs and gateway routes | PARTIAL | Direct URLs are listed, but there are no gateway routes. Login is documented with `username` although code requires `email`: [`README.md`](README.md#L52) | Correct contracts and document gateway/public URLs once implemented. |
| README | MongoDB and image-storage configuration | PASS | Mongo, storage variables, and media filesystem behavior are described: [`README.md`](README.md#L149), [`media README.md`](media-service/README.md#L40) | Document backup and orphan-retention policy. |
| README | Test commands, demo flow, known limitations | MISSING | None appear in the root README | Add exact commands, CLIENT/SELLER demo steps, known gaps, and expected responses. |
| Delivery | `.env.example` | PASS | [`.env.example`](.env.example#L1) contains a non-secret placeholder | — |
| Delivery | No committed real secrets/private keys | PASS | High-confidence tracked-file scan found only documented placeholders; `.env` is ignored: [`.gitignore`](.gitignore#L1) | Add automated secret scanning in CI. |
| Delivery | Repository hygiene | FAIL | 29 `media-service/target` build artifacts are tracked, including JAR/class files; a lock file with a developer-local absolute path is tracked: [`.LCKAppProperties.java~`](product-service/src/main/java/com/example/productservice/config/.LCKAppProperties.java~#L1) | Remove generated binaries/lock files from version control in a separate approved cleanup change. |

## End-to-end flow audit

Infrastructure was not started, so runtime-dependent flows remain `UNVERIFIED` even where the static chain is present.

| Flow | Status | Evidence | Failure point / next action |
|---|---:|---|---|
| Client registers, logs in, browses products | UNVERIFIED | Register/login frontend calls match User API; catalog calls public Product API | Start the stack and verify JWT creation, CORS/proxy behavior, Mongo persistence, and unauthenticated browsing. |
| Seller registers, logs in, reaches seller UI | UNVERIFIED | Role returned by auth controls navigation; AuthGuard and RoleGuard protect `/seller` | Verify new SELLER token is accepted by all services. |
| Seller creates valid product | UNVERIFIED | Product form → `ProductService.create` → POST `/products`; seller ID comes from JWT | Exercise through Nginx and direct API; verify Mongo document and response. |
| Seller uploads image under 2 MB and attaches it | UNVERIFIED | Media page uploads, stores returned ID, then PUTs `imageUrls` | Verify filesystem volume, Mongo metadata, image rendering, and orphan behavior if product update fails. |
| Seller edits/deletes own product | UNVERIFIED | Update/delete queries include authenticated seller ID | Test responses and Kafka cleanup. |
| Seller cannot mutate another seller's product/media | UNVERIFIED | Product lookup includes seller ID; media compares owner | Add automated two-seller tests and then verify manually. |
| CLIENT cannot write products/upload media | UNVERIFIED | Product controller checks role claim; Media uses `hasRole('SELLER')` | Verify 403 bodies through the eventual gateway. |
| Invalid product input gives clear feedback | UNVERIFIED | Backend returns 400; frontend prevents common invalid fields | Verify field-level backend details and improve frontend display of those details. |
| Oversize/non-image rejected at both layers | FAIL | Oversize and declared MIME type are checked, but arbitrary bytes labelled `image/png` pass | Add server-side decoding/magic-byte validation, then test renamed text/executable payloads. |
| Stored references retrieve and render images | UNVERIFIED | Frontend reconstructs `/media/images/{id}` correctly | Fix the Media Service's returned `url`, then verify direct and proxied rendering/cache headers. |

## Prioritized remaining work

### 1. Critical blockers

1. Add the required API Gateway and route all external API traffic through it.
2. Add service discovery and wire all services/gateway into it.
3. Repair the frontend test suite's compile errors so CI can run.
4. Fix `MEDIA_PUBLIC_BASE_URL` so returned URLs include `/media/images`.

### 2. Required missing functionality

1. Implement seller profile/avatar upload and update end to end.
2. Validate that attached media IDs exist and belong to the authenticated seller.
3. Add comprehensive backend authentication, authorization, ownership, validation, and media tests.
4. Add functional frontend guard/interceptor/service/component tests.

### 3. Security/reliability gaps

1. Inspect/decode uploaded bytes; do not trust `Content-Type`.
2. Add HTTPS deployment configuration and documentation.
3. Restrict production CORS to exact frontend origins.
4. Align proxy/gateway upload limits with 2 MB.
5. Validate JWT issuer/audience and plan asymmetric key management.
6. Add file/metadata orphan reconciliation and reliable Kafka delivery.
7. Standardize error responses and correlation IDs.
8. Harden localStorage token use with CSP/short lifetimes or a safer refresh-token design.

### 4. Frontend/UX gaps

1. Add profile/avatar UI.
2. Resolve the unsupported `category` field.
3. Add the referenced `placeholder.png` or use an embedded fallback.
4. Show backend validation and forbidden-action messages instead of generic errors.
5. Optionally add a seller-wide media library/orphan manager.

### 5. Documentation/deployment gaps

1. Rewrite the architecture section to show gateway, discovery, services, Mongo, Kafka, and storage.
2. Add prerequisites, all startup modes, health checks, and troubleshooting.
3. Correct login/media response contracts.
4. Add test/build commands, demo roles, and known limitations.
5. Stop tracking `target` binaries and editor lock files.

### 6. Optional enhancements

1. Gateway rate limiting for login/register/media upload.
2. Kafka outbox, retry topics, and dead-letter handling.
3. ADMIN moderation role.
4. Object storage such as S3/MinIO for multi-instance deployments.

## Completion score

Scoring method: each non-optional row in the checklist is one required unit; only `PASS` counts as complete. `PARTIAL`, `MISSING`, `FAIL`, and `UNVERIFIED` do not receive credit. End-to-end flows are verification scenarios and are not double-counted.

- Required requirements passed: **52 / 86**
- Required completion: **60%**
- Excluded as optional: Kafka events, ADMIN role, media DELETE, and rate limiting.

## Minimal end-to-end manual test plan

1. Start Mongo, Kafka, services, frontend, gateway, and discovery; confirm every health endpoint.
2. Register CLIENT, SELLER-A, and SELLER-B accounts; confirm duplicate/invalid registration errors.
3. Browse product list/detail with no token.
4. As CLIENT, attempt product POST/PUT/DELETE and media POST; expect 403.
5. As SELLER-A, create a product; test blank fields, zero/negative price, and negative stock.
6. Upload valid JPEG/PNG/WEBP files below 2 MB, attach IDs, reload the product, and verify image headers/rendering.
7. Upload a file over 2 MB, a text file, and text bytes falsely labelled `image/png`; all must be rejected.
8. As SELLER-B, attempt to update/delete SELLER-A's product and delete/attach SELLER-A's media.
9. Edit SELLER-A's product, remove an image, delete the product, and verify Kafka-driven cleanup.
10. Test expired, malformed, and bad-signature JWTs through both gateway and direct service URLs.

## Suggested verification commands

These commands were not executed during the audit. Build/test commands create normal `target`, `dist`, cache, or `node_modules` artifacts.

```powershell
# Static Compose validation
$env:JWT_SECRET = 'local-test-secret-at-least-32-characters'
docker compose config --quiet

# Start infrastructure for backend integration tests
docker compose up -d mongo kafka

# Backend tests
Set-Location user-service
$env:JWT_SECRET = 'local-test-secret-at-least-32-characters'
.\mvnw.cmd test

Set-Location ..\product-service
$env:JWT_SECRET = 'local-test-secret-at-least-32-characters'
$env:MONGODB_URI = 'mongodb://localhost:27018/productservice-test'
$env:KAFKA_BOOTSTRAP_SERVERS = 'localhost:9092'
.\mvnw.cmd test

Set-Location ..\media-service
$env:JWT_SECRET = 'local-test-secret-at-least-32-characters'
$env:MONGODB_URI = 'mongodb://localhost:27018/mediaservice-test'
$env:KAFKA_BOOTSTRAP_SERVERS = 'localhost:9092'
.\mvnw.cmd test

# Frontend install, tests, and production build
Set-Location ..\frontend
npm ci
npm test -- --watch=false
npm run build

# Full deployment
Set-Location ..
$env:JWT_SECRET = 'local-test-secret-at-least-32-characters'
docker compose up --build

# Health checks
Invoke-RestMethod http://localhost:8081/actuator/health
Invoke-RestMethod http://localhost:8082/actuator/health
Invoke-RestMethod http://localhost:8083/actuator/health
```

There is currently no frontend lint script or root-level aggregate verification command.
