# Buy-01 Completion Plan

> **For agentic workers:** REQUIRED SUB-SKILL: use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Implement one batch at a time, run the stated checks, show `git status --short`, and stop so the user can review and commit manually. Do not make Git commits on the user's behalf.

**Goal:** Finish and prove the Buy-01 marketplace against the official 01-edu subject and audit questions.

**Architecture:** Angular is served by Nginx. Browser API traffic goes through one Spring Cloud Gateway, which discovers User, Product, and Media services through Eureka. Each service still validates JWTs and enforces its own authorization; MongoDB stores domain data and media metadata, while image bytes remain in the media volume.

**Tech stack:** Java 21, Spring Boot 4.1, Spring Cloud 2025.1, Spring Security JWT, MongoDB, Kafka, Eureka, Angular 21, Angular Material, Nginx, Docker Compose.

**Spec:** the official subject, database diagram, and audit below.

**Official specifications:**

- [01-edu Buy-01 subject](https://github.com/01-edu/public/blob/master/subjects/java/projects/buy-01/README.md)
- [01-edu database design](https://github.com/01-edu/public/blob/master/subjects/java/projects/buy-01/Database-Design.png)
- [01-edu audit questions](https://github.com/01-edu/public/blob/master/subjects/java/projects/buy-01/audit/README.md)

**Audit snapshot:** reconciled after committed documentation updates (`c1d18f3`, `bd65608`) and commit `1381924`, plus the subsequent Batch 8 working-tree fixes. Automated and base-Compose runtime evidence are recorded below. The official questions were closed on 2026-09-05 using a healthy live stack, source/test evidence, and the user's confirmed browser visual retest; its limited interactive-evidence caveat is stated with the checked audit items.

## Status legend

- `[x]` means the implementation exists and was confirmed by source inspection, an automated test, or recorded runtime evidence. Where a box is checked with a caveat, the caveat is stated inline rather than left implicit.
- `[ ]` means work remains, and the reason is stated inline. This includes implemented-looking behavior that has not yet passed an automated or end-to-end check, and behavior that can only be proven outside this repository.
- The official subject and audit links above are the source of truth. `AUDIT_REPORT.md` and the service `TODO.txt` files are older snapshots and must not override current code evidence.

## Verification snapshot

Two independent bodies of evidence back the checkboxes below.

**Automated (clean state, 2026-09-04):** `bash scripts/verify.sh` from a state where all five Maven modules had been cleaned, exit code 0. The same command was re-run while this section was being reconciled; it failed twice for the spec-teardown reason described below, and passes now that the cause is fixed. The per-module test totals below were read from this machine's own Surefire XML rather than copied forward.

**Runtime (full-stack Docker audit, 2026-09-04):** 172 evidence rows, **172 PASS / 0 FAIL**, spread over seven files — API through Gateway and Nginx (59), JWT tampering (8), headless browser page loads (5), production TLS overlay (18), container health (44), volume persistence (10), and secret/path leakage (28). Every row is emitted by a harness that computed its own verdict from an exact string comparison; none is hand-transcribed. Two Compose stacks were used, both built from clean with empty named volumes: the base `docker-compose.yml` and the base plus `docker-compose.prod.yml` plus a local TLS overlay.

Runtime citations below name an evidence file and a row, for example `api-evidence.tsv:client-create-product`. Those files, their harnesses, and the per-task reports live under `.superpowers/sdd/PLAN/`, which is untracked, so the citations are reproducible for whoever ran the audit but do not ship with the repository.

- [x] `JWT_SECRET=local-audit-secret-at-least-32-characters docker compose config --quiet` passes (`scripts/verify.sh:44-45`). The two overlay merges the script does not cover were validated by hand for this revision and both exit 0: base + `docker-compose.prod.yml`, and base + prod + the local TLS overlay, each with `JWT_SECRET` and `SERVER_NAME` supplied. Both are needed — the base file's `JWT_SECRET` and the overlay's `SERVER_NAME` each fail interpolation with a non-zero exit if unset, which is how the command documented in the README was found to be missing `JWT_SECRET` and corrected.
- [x] `npm run build` produces the Angular browser bundle with no budget failure against the configured budgets (`frontend/angular.json:60-71`).
- [x] API Gateway tests pass: 16 tests, 0 failures, 0 errors.
- [x] Discovery Service tests pass: 1 test, 0 failures, 0 errors.
- [x] Product Service compiles and its full suite passes: 48 tests, 0 failures, 0 errors.
- [x] Media Service compiles and its full suite passes: 30 tests, 0 failures, 0 errors.
- [x] User Service tests pass independently without a developer-managed MongoDB: 31 tests, 0 failures, 0 errors.
- [x] Backend total across the five Maven modules: 126 tests, 0 failures, 0 errors, 0 skipped (Gateway 16, Discovery 1, User 31, Product 48, Media 30).
- [x] `npm test -- --watch=false` passes: 19 spec files, 54 tests, 0 failures.
- [x] `npm audit --omit=dev` reports 0 vulnerabilities.
- [x] Product and Media have focused service tests for media ownership, validation, storage, and lifecycle behavior.
- [x] Focused User avatar service tests pass: 10 avatar cases in `UserServiceTest` covering owned assignment, replacement, removal, foreign/invalid rejection, CLIENT denial, and persistence-failure compensation.
- [x] `docker compose up --build` has been run successfully for the complete stack — `health-evidence.tsv:compose-up-build-exit` is `0` for both the base and the production-overlay stack, each with `named-volumes-created-empty=2` proving the run started from clean rather than onto existing data.
- [x] Official browser/Postman audit closure is supported by live probes, current source/test evidence, and the historical machine-verified `api-evidence.tsv` (59/59) and `jwt-evidence.tsv` (8/8) records summarized in this plan. On 2026-09-05, the live stack had eight healthy containers and returned 200 from the public frontend, Nginx/Gateway products route, and direct development Gateway products route; an anonymous protected-route request returned 401. The browser conclusion is qualified by the user's confirmed visual retest; this closure did not independently repeat every browser scenario or create new audit users, products, or media.

**A real failure was found in the verification gate during this reconciliation, and fixed.**
Both `scripts/verify.sh` runs at the start of this pass exited **1**, with Angular spec files
failing to load under `EnvironmentTeardownError: Cannot load '/chunk-…js' … after the
environment was torn down` — three catalog files in the first run, one in the second, each
reporting `0 test`. No assertion ever failed; the tests inside the unloaded files simply never
ran, so the runs reported 46 and 49 passing instead of 50. The runner's own callstack named the
cause: `src/app/layout/header/header.spec.ts` was the only spec importing `AppModule`,
`AppModule` imports `AppRoutingModule`, whose `''` route is
`loadChildren: () => import('./catalog/catalog-module')`, and the root router starts that lazy
import during its initial navigation without the spec awaiting it. Left pending, it resolved
after Vitest had torn the environment down, and the error was attributed to whichever suites
were loading at that moment. The race looked intermittent — six standalone `npm test` runs
passed, two of them with `.angular/cache` and the Vite cache deleted first — but it is
deterministic in the sequence the gate itself uses: `npm ci` immediately followed by `npm test`
reproduced it **3 out of 3** times, because the cold dependency tree widens the window. The fix
awaits the router's initial navigation in that spec's `beforeEach`, before the fixture is
created, so the lazy chunk resolves inside the test lifecycle while the component's first
`detectChanges` remains its initial render. Verified green on the same `npm ci` plus `npm test`
sequence at that time and by a full `scripts/verify.sh` run; current totals are recorded above.

## Official audit checklist

### 1. Initial setup and access

- [x] Separate User, Product, and Media Spring Boot services exist.
- [x] MongoDB, Kafka, Eureka, API Gateway, all three services, and Angular/Nginx are declared in `docker-compose.yml`.
- [x] MongoDB data and media files use named Docker volumes.
- [x] Docker Compose configuration parses with a valid `JWT_SECRET`.
- [x] Every Spring component exposes `/actuator/health`.
- [x] Build and start the entire application with one documented command — `docker compose up --build -d --wait` (`README.md`, "Start the complete Docker stack"), observed at exit code 0 with all eight containers healthy (`health-evidence.tsv:compose-up-build-exit`, `:all-containers-healthy` = `8-of-8`) in both the base and production-overlay stacks.
- [x] Confirm every container becomes healthy using only commands available inside its image — `health-evidence.tsv` carries a `*-health-state` row (`healthy`) and a separate `*-in-image-probe` row (`rc=0`) for each of the eight containers in each of the two stacks. Every probe uses only binaries present in that image: `mongosh`, `kafka-topics.sh`, busybox `wget` for the Java actuators, and busybox `wget`+`grep` for Nginx. The `frontend` health check that makes this satisfiable was missing and was added during this batch's runtime audit (`docker-compose.yml`, `frontend.healthcheck`; asserted present by `health-evidence.tsv:frontend-healthcheck-declared`, whose row text records the pre-fix RED state). The probe form matters: an earlier attempt passed in development but went unhealthy under the production overlay, because busybox `wget` followed the port-80 301 into a certificate check against the self-signed cert. The shipped probe asserts only the status line and accepts `200` or `301`.
- [x] Confirm the Angular site and all public/protected API routes are reachable through the intended public entry points — Angular renders through Nginx (`headless-evidence.tsv` 5/5) and over TLS (`tls-evidence.tsv:tls-https-serves-spa`, `:tls-spa-deeplink-fallback`). Public and protected API routes were exercised through the documented development Gateway entry point (`api-evidence.tsv`, 59 rows covering auth, `/me`, products, and media), through Nginx `/api` (`:public-products-nginx`, `:login-seller-a-nginx`, `:image-render-nosniff-via-nginx`), and through Nginx over HTTPS (`tls-evidence.tsv:tls-api-reaches-gateway`, `:tls-api-unauth-error-contract`). No route was reached by bypassing Nginx or the Gateway.
- [x] Remove normal public host-port exposure for internal microservices, or clearly separate local-debug and production Compose configurations.

### 2. User and Product operations

- [x] Users can register as either `CLIENT` or `SELLER`.
- [x] Users can log in with email/password and receive a JWT.
- [x] Authenticated users can read and update their own profile through `GET /me` and `PUT /me`.
- [x] Duplicate email and username checks exist, backed by MongoDB unique indexes.
- [x] Public users can list products and view one product.
- [x] Sellers can list their own products.
- [x] Sellers can create, update, and delete products.
- [x] Product writes derive the seller ID from the JWT rather than accepting it from the request body.
- [x] Product update/delete queries include both product ID and authenticated seller ID.
- [x] Add automated proof that a client cannot create/update/delete a product — `ProductControllerTest.deniesAClientFromCreatingAProduct` (403 with the standard body) and `ApiGatewayApplicationTests.clientRoleCannotWriteProducts`, confirmed at runtime by `api-evidence.tsv:client-create-product` (403). Update and delete are gated by the same single `ensureSeller(jwt)` guard the create path uses (`ProductController.java:75-80`, invoked by all three write methods) and by per-method `hasRole("SELLER")` matchers for `POST`, `PUT`, and `DELETE` (`api-gateway/.../security/SecurityConfig.java:57-59`); those two methods have no CLIENT-token test of their own.
- [x] Add automated proof that Seller B cannot update/delete Seller A's product — `ProductServiceTest.doesNotUpdateAProductOwnedByAnotherSeller` and `.doesNotDeleteAProductOwnedByAnotherSeller`, confirmed at runtime by `api-evidence.tsv:seller-b-update-product-blocked` and `:seller-b-delete-product-blocked` (both 404, existence-masked by design).
- [x] Reject whitespace-only product update values instead of silently ignoring them — commit `1381924` rejects whitespace-only optional Product update values at validation time, with Product tests covering the 400 response rather than a silent no-op.
- [x] Resolve the frontend-only `category` field. The official model does not require it, so remove it from the Angular interface/form/display unless the backend deliberately adopts it — removed. `grep -rni category frontend/src` now returns nothing; the last two references were dead SCSS rules (`.card-category` in `product-card.scss`, `.category` in `product-detail.scss`), deleted here.
- [x] Rename the product media collection from `imageUrls` to `imageIds`, and enforce it across Java DTOs, MongoDB, Kafka events, Angular, tests, and documentation. Legacy request/event readers remain temporarily compatible during rollout.

### 3. Authentication and role validation

- [x] Passwords are checked through Spring Security's `PasswordEncoder`.
- [x] JWT signature and expiry validation is configured in Gateway, User, Product, and Media services.
- [x] JWT role claims map to Spring authorities.
- [x] Angular has an authentication guard, seller role guard, and bearer-token interceptor.
- [x] Product and Media services retain downstream role checks even though the Gateway also checks roles.
- [x] Route all Angular development and production API requests through the Gateway.
- [x] Add an `aud` claim and validate the expected issuer and audience in every JWT decoder — the claim is minted at `user-service/.../security/JwtService.java:35,42`, and all four decoders combine `JwtValidators.createDefaultWithIssuer(...)` with an explicit audience predicate: `user-service/.../security/SecurityConfig.java:127-133`, `product-service/.../security/SecurityConfig.java:100-106`, `media-service/.../security/SecurityConfig.java:103-109`, `api-gateway/.../security/SecurityConfig.java:88-94`. Issuer and audience are `@NotBlank`-required configuration in every module, so a service cannot start with them unset.
- [x] Externalize exact CORS origins for every environment and avoid wildcard-style downstream production CORS — every module reads `CORS_ALLOWED_ORIGINS` (default `http://localhost:4200`) and binds it to a list, and the production overlay pins all four services to `https://${SERVER_NAME}` (`docker-compose.prod.yml:24-36`). No `setAllowedOriginPattern`, and no `"*"` origin, appears anywhere in the codebase. The three internal services still allow all request *headers* (`setAllowedHeaders(List.of("*"))`); origins, methods, and exposed headers are exact, and those services publish no host port in production.
- [x] Add tests for missing, malformed, expired, wrong-signature, wrong-issuer, and wrong-audience tokens — all six at the Gateway boundary: `ApiGatewayApplicationTests.protectedRouteReturnsStructuredUnauthorizedResponse` (missing), `.rejectsMalformedExpiredAndWrongSignatureTokens` (three cases), `.rejectsTokenWithWrongIssuer`, `.rejectsTokenWithWrongAudience`. All six assert 401; the missing-token case additionally pins the full error body and the JSON content type, and the wrong-issuer and wrong-audience cases pin a non-empty `correlationId`. Confirmed at runtime by `jwt-evidence.tsv` (8/8), whose `local-key-positive-control` row signs unmodified claims with the same local key and requires 200, so the negative rows cannot pass merely because the key was wrong.
- [x] Add end-to-end role checks for CLIENT, Seller A, and Seller B through the public Gateway route — `api-evidence.tsv` carries a login row per persona (`login-client`, `login-seller-a`, `login-seller-b`, plus `login-seller-a-nginx`) and role outcomes for each: `client-create-product` 403, `client-upload` 403, `client-avatar-reference-blocked` 403, Seller A's full CRUD/upload/avatar sequence, and `seller-b-*` denials on Seller A's product (404), media metadata (403), media delete (403), and cross-seller media attachment (403).

### 4. Media upload and product association

- [x] Sellers can upload through `POST /media/images`.
- [x] Images can be retrieved publicly through `GET /media/images/{id}`.
- [x] The owning seller can delete media through `DELETE /media/images/{id}`.
- [x] Media metadata is stored in MongoDB and file bytes are stored outside MongoDB.
- [x] The service rejects empty files, files larger than 2 MiB, and declared MIME types outside JPEG/PNG/WEBP.
- [x] The Angular media page checks declared type and size before uploading.
- [x] Products persist a list of media references, and the seller UI supports upload, preview, attachment, and removal.
- [x] Inspect magic bytes and decode uploaded content instead of trusting client-supplied `Content-Type`.
- [x] Derive the persisted/served MIME type and storage extension from validated content.
- [x] Generate storage names without incorporating the original filename and prove the resolved path stays below the configured storage root.
- [x] Let an exact 2 MiB file pass while allowing multipart overhead (`max-file-size: 2MB`, request limit slightly larger, service byte check authoritative).
- [x] Delete the stored file if MongoDB metadata persistence fails.
- [x] Return an intentional response for missing/corrupt storage instead of a generic 500.
- [x] Correct the returned media URL to the real Gateway-visible `/media/images/{id}` route.
- [x] Add an authenticated media metadata endpoint that exposes the media ID, owner ID, validated type, and size without exposing its filesystem path.
- [x] Before Product Service accepts any media ID, verify that it exists, is an image, and belongs to the authenticated seller.
- [x] Prevent duplicate media IDs and define replacement/removal as replacing the product's complete ordered media-ID list.
- [x] Add an orphan cleanup/compensation strategy when upload succeeds but product attachment fails.
- [x] Document the database-design mapping: `sellerId` corresponds to the diagram's product `userId`, `stock` to `quantity`, `storageKey` to `imagePath`, and `Product.imageIds` is the canonical product-media relation.

### 5. Frontend interaction and implementation

- [x] Sign-in and sign-up pages use reactive forms.
- [x] Sign-up allows CLIENT/SELLER role selection.
- [x] A public product-list page loads and displays products.
- [x] A product-detail page displays product information and images.
- [x] The seller dashboard lists owned products and offers create/edit/delete/media actions.
- [x] A dedicated per-product media-management page exists.
- [x] Angular uses feature modules, services, guards, an interceptor, and Angular Material components.
- [x] Add a protected profile page and navigation entry.
- [x] Allow a seller to upload, display, replace, and remove their avatar.
- [x] Add `GET /me` and `PUT /me` calls to the Angular auth/profile service.
- [x] Replace the missing `assets/placeholder.png` reference with a real asset or a code/CSS fallback.
- [x] Align Angular and backend field constraints (`username`, product name, and product description lengths).
- [x] Make Product Form subscriptions lifecycle-safe and prevent duplicate submits.
- [x] Show safe backend validation, conflict, forbidden, upload, and network messages instead of generic failures or silent redirects.
- [x] Limit bearer-token attachment to the configured API origin/path.
- [x] Fix the Angular template warning and either reduce the initial bundle below its configured budget or set a justified budget.
- [x] Update vulnerable transitive frontend dependencies. Current audit reports zero vulnerabilities.

### 6. Security

- [x] Registration hashes passwords with BCrypt before persistence.
- [x] API response DTOs do not contain the password field.
- [x] Real JWT secrets are required through environment variables and `.env` is ignored.
- [x] Expected application errors do not intentionally include stack traces in responses.
- [x] Product ownership is checked inside Product Service.
- [x] Media deletion ownership is checked inside Media Service.
- [x] Replace User's unrestricted `avatarUrl` input with an owned, validated media ID/reference.
- [x] Exclude the password hash from Lombok-generated `toString()` output — `@ToString.Exclude` on `User.password` (`user-service/.../model/User.java:33-35`). Corroborated at runtime: `secrets-evidence.tsv:logs-bcrypt-hash-absent` and `:responses-bcrypt-hash-absent` are both 0 across 2,571 log lines and every swept response body.
- [x] Enforce media ownership when associating media with a product or avatar.
- [x] Put Nginx/Gateway in the only external API path so Gateway security cannot be bypassed.
- [x] Add production HTTPS termination, HTTP-to-HTTPS redirect, HSTS, and certificate-renewal documentation. Local HTTP must be explicitly documented as development-only — configuration at `frontend/nginx.prod.conf:1-16` (redirect block, TLS 1.2/1.3 only, Let's Encrypt paths) and `docker-compose.prod.yml:11-19`; renewal procedure and its dry run at `README.md`, "Production HTTPS and certificate renewal"; development-only statement at `README.md`, "Architecture and public routing". Runtime behavior proven by `tls-evidence.tsv`: `tls-http-redirect-root` and `:tls-http-redirect-deep-path` (301 preserving path and query), `:tls-header-hsts`, `:tls-protocol-floor` (TLS 1.0/1.1 rejected, 1.2/1.3 accepted). **Caveat, stated deliberately:** all TLS runtime evidence was produced against a temporary, locally generated self-signed certificate on a throwaway host name (`tls-evidence.tsv:tls-certificate-is-local-self-signed` records `CN=audit.local`). No certificate was ever requested from a real CA and no deployment host or DNS name was involved, so the configuration and the documentation are proven while issuance and renewal against a real CA remain unverified — see the corresponding unchecked note in the Definition of done.
- [x] Add a restrictive production Content-Security-Policy and other appropriate Nginx security headers — `frontend/nginx.prod.conf:35-39`, all five verified present on both the SPA and `/api` responses (`tls-evidence.tsv:tls-header-csp`, `:tls-header-x-content-type-options`, `:tls-header-x-frame-options`, `:tls-header-referrer-policy`, `:tls-header-hsts`, `:tls-api-security-headers` = `5-of-5`). Version disclosure is closed on both server blocks (`:tls-redirect-server-tokens-off`, `:tls-https-server-tokens-off`, each observing a bare `nginx`).
- [x] Confirm secrets, JWTs, passwords, and storage paths are absent from logs and API responses — `secrets-evidence.tsv` 28/28. The signing secret and a freshly minted live token were both grepped for **by value** across every service log in both stacks (`logs-jwt-secret-absent`, `logs-live-token-absent`), and 9 un-redacted live response bodies plus the upload and both auth responses were swept for the secret, JWT-shaped strings, the password literal, BCrypt hashes, `mongodb://` URIs, storage paths, `storageKey`, and stack frames. `upload-response-storedfilename-has-no-path` asserts the only body that names stored bytes carries a bare basename with no separator. The two auth bodies are required to contain a token (`auth-response-returns-token` = 2), which is what stops the token-absence rows elsewhere from being vacuous.

### 7. Error handling and edge cases

- [x] Duplicate registration maps to HTTP 409.
- [x] Invalid credentials map to HTTP 401.
- [x] Missing/invalid authentication maps to JSON HTTP 401 responses.
- [x] Insufficient roles map to JSON HTTP 403 responses.
- [x] Missing Product/Media records map to HTTP 404.
- [x] Bean validation and malformed JSON map to HTTP 400.
- [x] Each core service has centralized exception handling.
- [x] Standardize one error body across Gateway, User, Product, and Media (`timestamp`, `status`, `error`, `message`, optional `details`, `correlationId`) — one private builder per service emits exactly those keys in that order (`user-service/.../exception/GlobalExceptionHandler.java:129-137`, `product-service/.../exception/GlobalExceptionHandler.java:71-81`, `media-service/.../exception/GlobalExceptionHandler.java:77-85`, which omits only the optional `details`), and the Gateway writes the same shape reactively (`api-gateway/.../web/GatewayErrorResponseWriter.java:17-26`). Confirmed at runtime: `secrets-evidence.tsv:error-response-fields` observes exactly `correlationId,error,message,status,timestamp`, `api-evidence.tsv:invalid-registration` carries the optional `details` map, and `tls-evidence.tsv:tls-api-unauth-error-contract` shows the Gateway's own 401 body with a caller-supplied correlation ID echoed back.
- [x] Preserve and display field-level validation details in Angular — commit `1381924` renders backend validation `details` at the affected form fields; the later Batch 8 working-tree change also repairs the Angular TS4111 indexed access involved in that rendering. Scoped independent reviews passed.
- [x] Handle Media Service unavailability during Product and User ownership validation with an intentional HTTP 503 response.
- [x] Test duplicate registration, invalid product fields, malformed JSON, empty uploads, false MIME declarations, corrupted images, exact/over-limit sizes, cross-seller access, missing files, and unavailable dependencies — every item has automated coverage, runtime coverage, or both: duplicate registration (`UserServiceTest.registerRejectsAnExistingEmail…`/`…Username…`, `AuthControllerTest.registerMapsExistingIdentityToConflict`; `api-evidence.tsv:duplicate-email`, `:duplicate-username`, both 409); invalid product fields (`ProductControllerTest` parameterized create and update boundaries); malformed JSON (`api-evidence.tsv:invalid-role-registration` returns 400 `"Malformed request body"` from the unreadable-body handler); empty uploads (`ImageContentValidatorTest.rejectsEmptyExecutableArchiveAndCorruptImageBodies`); false MIME (`.rejectsDeclaredMimeTypeThatDoesNotMatchDetectedContent`; `api-evidence.tsv:false-declared-mime`); corrupted images (`.rejectsCorruptAndTrailingPolyglotContent`; `:corrupt-image`, `:trailing-payload`); exact and over-limit sizes (`.acceptsAValidImageAtTheExactTwoMiBBoundary`, `.rejectsAnImageOneByteOverTheLimit`; `:exact-two-mib-png` 201, `:over-two-mib-png` 400); cross-seller access (`MediaServiceTest.preventsASellerFromDeletingAnotherSellersImage`, `ProductServiceTest` two-seller cases; the `seller-b-*` rows); missing files (`MediaServiceTest.reportsMissingStorageContentAsMediaNotFound`, `.reportsCorruptStorageContentAsMediaNotFound`); unavailable dependencies (`ProductControllerTest.returnsSafeRetryableResponseWhenMediaValidationIsUnavailable` asserting 503 and the safe message, plus three `MediaOwnershipClientTest` cases).
- [x] Propagate the Gateway correlation ID through all services and include it in logs/errors — commit `1381924` adds the missing Gateway and Media console correlation patterns; the later Batch 8 working-tree change adds reactive Gateway MDC propagation. Direct proof is the embedded HTTP Gateway test, which logs caller `gateway-client-proof`, plus concurrent cleanup/isolation tests. This is not a claim about a live Compose container log.

### 8. Code quality and testing

- [x] Controllers, services, repositories, Mongo documents, DTO validation, and security configurations use the expected Spring annotations.
- [x] Persistence entities are not returned from User/Product JSON APIs where sensitive or internal fields would leak.
- [x] Generated build output and editor lock files are ignored and no longer tracked.
- [x] Gateway route/security tests pass.
- [x] Discovery startup/configuration test passes.
- [x] Replace infrastructure-dependent context-only backend tests with reliable unit/slice tests or explicit Testcontainers integration tests — the three context tests now use `ApplicationContextRunner` with mocked repositories, encoders, and clients and start no external infrastructure (`user-service/.../Buy01ApplicationTests.java`, `product-service/.../ProductServiceApplicationTests.java`, `media-service/.../MediaServiceApplicationTests.java`). The clean-state run needed no MongoDB, Kafka, or Eureka.
- [x] Add User registration/login/profile validation and security tests — 31 tests in `user-service`: `UserServiceTest` (17, including identity normalization, duplicate email/username rejection before encoding or saving, wrong-password without token generation, unknown-user profile, and the 10 avatar cases), `AuthControllerTest` (5 HTTP-boundary cases), `UserControllerTest` (3, including JWT-subject scoping and bearer forwarding), plus the client and context tests.
- [x] Add Product CRUD, validation, role, and two-seller ownership tests — 48 tests in `product-service`, including `ProductServiceTest` coverage for create/read/update/delete, image-ID validation, two-seller denials, and duplicate-key translation; `ProductControllerTest` coverage for role denial, 503 mapping, legacy `imageUrls`, and invalid requests; plus client, migration, persistence-invariant, and event/DTO compatibility tests.
- [x] Add Media content, size-boundary, storage compensation, role, and ownership tests — 30 tests in `media-service`: `ImageContentValidatorTest` (8, including the exact-2 MiB boundary, one byte over, renamed text, empty/executable/archive bodies, MIME mismatch, and polyglot/trailing content), `MediaServiceTest` (7, including storage cleanup on metadata failure, cleanup-failure preservation, missing and corrupt stored bytes, and cross-seller deletion denial), `MediaControllerTest` (8 role/ownership/header cases), `LocalFileStorageServiceTest` (path containment), and the Kafka consumer/event tests.
- [x] Repair the four non-compiling Angular specs — `role-guard.spec.ts`, `auth.spec.ts`, `media.spec.ts`, and `product.spec.ts` all compile and pass; current root verification reports 19 spec files and 54 tests with 0 failures.
- [x] Add behavioral Angular tests for auth services, guards, interceptor, catalog pages, seller CRUD, media management, error feedback, and profile/avatar — 19 spec files covering auth service (`shared/services/auth.spec.ts`), both guards (`auth-guard.spec.ts`, `role-guard.spec.ts`), the interceptor (`auth-token-interceptor.spec.ts`), catalog (`home`, `product-list`, `product-detail`, `product-card`), seller CRUD (`dashboard.spec.ts`, `product-form.spec.ts`), media management (`product-media.spec.ts`, `media.spec.ts`), error feedback (`login.spec.ts`, `product-form.spec.ts`), and profile/avatar (`profile.spec.ts`).
- [x] Add a repeatable root-level verification command or CI workflow — `scripts/verify.sh`, documented in `README.md` under "Automated verification and manual audit", runs the five Maven suites, `npm ci`, the Angular suite, the production build, Compose validation, and the production dependency audit, and exits non-zero on any failure. There is no `.github/workflows` directory; this item's "or CI workflow" alternative is unused.
- [x] Close the official audit after all automated checks pass. The 2026-09-05 closure assessed all nine official questions as Yes or qualified Yes from live probes, current source/test evidence, historical runtime records, and the user's confirmed browser visual retest; it was not an independent complete three-persona manual UI audit. Current automated verification passes 126 backend tests (Gateway 16, Discovery 1, User 31, Product 48, Media 30), 54 Angular tests over 19 files, the production build, Compose configuration validation, and `npm audit --omit=dev` with zero vulnerabilities. No new destructive fixture flows were run during the closure.

## Completed foundation outside the minimum subject

- [x] Eureka Discovery Service exists and its test passes.
- [x] User, Product, Media, and Gateway register as Eureka clients when enabled.
- [x] API Gateway uses discovery-backed `lb://` routes.
- [x] Gateway has route authorization, JSON 401/403 responses, CORS configuration, correlation IDs, and an upload request limit.
- [x] Product and Media Kafka publishers/consumers handle product/image lifecycle events.
- [x] Repository cleanup removed tracked build artifacts and editor lock files.

## Ordered implementation batches

Only mark a batch complete after its acceptance checks pass. After every batch, show `git status --short`, summarize changed files, suggest the listed Conventional Commit message, and ask the user to review and commit manually.

### Batch 1: Route all browser traffic through the Gateway

- [x] **Batch complete**

**Files:**

- Modify: `frontend/src/environments/environment.ts`
- Modify: `frontend/src/environments/environment.prod.ts`
- Modify: `frontend/src/app/shared/services/auth.ts`
- Modify: `frontend/src/app/shared/services/product.ts`
- Modify: `frontend/src/app/shared/services/media.ts`
- Modify: `frontend/nginx.conf`
- Modify: `docker-compose.yml`
- Test: frontend service specs and Gateway route tests

**Interface:** use one `apiBaseUrl`; services append `/auth`, `/me`, `/products`, and `/media/images`. In Docker, Nginx maps `/api/**` to `http://api-gateway:8080/**`.

- [x] Write/update URL tests showing development calls `http://localhost:8080` and production calls same-origin `/api`.
- [x] Replace the three environment service origins with one Gateway base URL.
- [x] Replace the three Nginx service locations with one `/api/` Gateway proxy and preserve correlation/forwarding headers.
- [x] Make the frontend depend on healthy `api-gateway`, not directly on all three services.
- [x] Remove or isolate internal service host ports for production-style startup.
- [x] Run Gateway tests, Angular tests, Angular production build, and Compose config validation.
- [x] Suggested commit: `fix(routing): send frontend API traffic through gateway`

### Batch 2: Make image validation authoritative

- [x] **Batch complete**

**Files:**

- Modify: `media-service/pom.xml`
- Modify: `media-service/src/main/resources/application.yml`
- Modify: `media-service/src/main/java/com/example/mediaservice/service/MediaService.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/service/LocalFileStorageService.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/controller/MediaController.java`
- Create: focused Media service/controller/storage tests under `media-service/src/test/java`

**Interface:** allow only decodable JPEG, PNG, and WEBP content up to exactly `2 * 1024 * 1024` bytes; derive content type server-side.

- [x] Write failing tests for empty, one-byte-over-limit, renamed text, executable/archive signatures, MIME mismatch, corrupt JPEG/PNG/WEBP, traversal filenames, and valid boundary files.
- [x] Detect content from bytes and decode it with a maintained image reader rather than trusting request metadata.
- [x] Generate storage keys entirely server-side and enforce normalized paths below the storage root.
- [x] Increase only the multipart request limit enough for framing overhead while retaining the exact file byte limit.
- [x] Add file cleanup when metadata persistence fails and intentional handling for missing/corrupt stored bytes.
- [x] Sanitize response filenames and derive response headers from validated metadata.
- [x] Correct the public media URL to the Gateway-visible `/api/media/images/{id}` path.
- [x] Run all Media tests and package the service.
- [x] Suggested commit: `fix(media): validate image content and storage lifecycle`

### Batch 3: Enforce media ownership during product association

- [x] **Batch complete** — implementation integrated in `d2e12b4`; before a mixed-version Product rollout, close the remaining legacy-writer (`imageUrls`) exclusivity gap and rerun Docker package checks against this commit.

**Files:**

- Create: `media-service/src/main/java/com/example/mediaservice/dto/MediaMetadataResponse.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/controller/MediaController.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/service/MediaService.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/security/SecurityConfig.java`
- Create: `product-service/src/main/java/com/example/productservice/client/MediaOwnershipClient.java`
- Create: `product-service/src/main/java/com/example/productservice/config/RestClientConfig.java`
- Modify: `product-service/pom.xml`
- Modify: `product-service/src/main/java/com/example/productservice/config/AppProperties.java`
- Modify: `product-service/src/main/resources/application.yml`
- Modify: `product-service/src/main/java/com/example/productservice/controller/ProductController.java`
- Modify: `product-service/src/main/java/com/example/productservice/service/ProductService.java`
- Modify: `product-service/src/main/java/com/example/productservice/model/Product.java`
- Modify: `product-service/src/main/java/com/example/productservice/dto/CreateProductRequest.java`
- Modify: `product-service/src/main/java/com/example/productservice/dto/UpdateProductRequest.java`
- Modify: `product-service/src/main/java/com/example/productservice/dto/ProductResponse.java`
- Modify: `product-service/src/main/java/com/example/productservice/repository/ProductRepository.java`
- Modify: `product-service/src/main/java/com/example/productservice/kafka/ProductEvent.java`
- Modify: `product-service/src/main/java/com/example/productservice/kafka/ImageEventConsumer.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/kafka/ProductDeletedEvent.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/kafka/ProductEventConsumer.java`
- Modify: `frontend/src/app/shared/services/product.ts` and all components/templates that consume `imageUrls` if the field is renamed
- Create: `media-service/src/test/java/com/example/mediaservice/controller/MediaControllerTest.java`
- Create: `product-service/src/test/java/com/example/productservice/service/ProductServiceTest.java`
- Create: `product-service/src/test/java/com/example/productservice/client/MediaOwnershipClientTest.java`

**Interface:** authenticated `GET /media/images/{id}/metadata` returns `{id, sellerId, contentType, sizeBytes}`. Product create/update passes the bearer token to a discovery-backed Media client and accepts only owned validated images.

- [x] Add Media service coverage for owned metadata access and cross-seller denial.
- [x] Implement the minimal authenticated metadata DTO/endpoint without returning `storageKey`.
- [x] Add focused Product service tests for accepted, duplicate, invalid/foreign, and unavailable-media cases.
- [x] Implement a configurable internal Media client and validate all changed image references before saving Product.
- [x] Map Media unavailability to an intentional retryable HTTP 503 response instead of a generic stack trace.
- [x] Choose one field name/representation for media references and update model, DTO, events, frontend, tests, and documentation together.
- [x] Define idempotent removal and orphan cleanup behavior.
- [x] Run Product and Media tests plus Docker package checks. Product (26 tests), Media (24 tests), and frontend (29 tests) pass; Docker package checks remain to be rerun against the integrated commit.
- [x] Commit created: `d2e12b4 feat(products): integrate media ownership workflow`

### Batch 4: Complete the seller avatar backend flow

- [x] **Batch complete**

**Files:**

- Modify: `user-service/pom.xml`
- Modify: `user-service/src/main/java/com/example/userservice/config/AppProperties.java`
- Create: `user-service/src/main/java/com/example/userservice/config/RestClientConfig.java`
- Create: `user-service/src/main/java/com/example/userservice/client/MediaOwnershipClient.java`
- Modify: `user-service/src/main/java/com/example/userservice/model/User.java`
- Modify: `user-service/src/main/java/com/example/userservice/dto/UpdateProfileRequest.java`
- Modify: `user-service/src/main/java/com/example/userservice/dto/UserProfileResponse.java`
- Modify: `user-service/src/main/java/com/example/userservice/controller/UserController.java`
- Modify: `user-service/src/main/java/com/example/userservice/service/UserService.java`
- Modify: `user-service/src/main/resources/application.yml`
- Reuse: `GET /media/images/{id}/metadata` from Batch 3
- Create: `user-service/src/test/java/com/example/userservice/service/UserServiceTest.java`
- Create: `user-service/src/test/java/com/example/userservice/controller/UserControllerTest.java`

**Interface:** User stores an `avatarMediaId`, never an arbitrary URL. Seller avatar updates validate ownership through Media Service; clients cannot set seller media as an avatar.

- [x] Add tests for owned avatar assignment, foreign/invalid media rejection, CLIENT restrictions, replacement, removal, and persistence-failure cleanup.
- [x] Replace unrestricted `avatarUrl` input with a validated media ID and explicit `removeAvatar` command.
- [x] Preserve self-only profile access using the JWT subject and forward its bearer token for media validation.
- [x] Compensate for a newly uploaded avatar if profile persistence fails; replacement/removal only detach the previous reference so media shared with products is never deleted.
- [x] Exclude `password` from Lombok `toString()` and retain password-free profile response DTOs.
- [x] Run User tests without requiring a developer-managed MongoDB instance.
- [x] Suggested commit: `feat(users): add owned seller avatar references`

### Batch 5: Complete frontend profile, contracts, and feedback

- [x] **Batch complete**

**Files:**

- Create: `frontend/src/app/profile/profile-module.ts`
- Create: `frontend/src/app/profile/profile-routing-module.ts`
- Create: `frontend/src/app/profile/pages/profile/profile.ts`
- Create: `frontend/src/app/profile/pages/profile/profile.html`
- Create: `frontend/src/app/profile/pages/profile/profile.scss`
- Create: `frontend/src/app/profile/pages/profile/profile.spec.ts`
- Create: `frontend/src/app/shared/services/profile.ts`
- Create: `frontend/src/app/shared/services/profile.spec.ts`
- Modify: `frontend/src/app/app-routing-module.ts`
- Modify: `frontend/src/app/layout/header/header.ts`
- Modify: `frontend/src/app/layout/header/header.html`
- Modify: `frontend/src/app/shared/services/auth.ts`
- Modify: `frontend/src/app/shared/services/media.ts`
- Modify: `frontend/src/app/shared/services/media.spec.ts`
- Modify: `frontend/src/app/shared/services/product.ts`
- Modify: `frontend/src/app/seller/pages/product-form/product-form.ts`
- Modify: `frontend/src/app/seller/pages/product-form/product-form.html`
- Modify: `frontend/src/app/shared/interceptors/auth-token-interceptor.ts`
- Modify: `frontend/src/app/shared/interceptors/auth-token-interceptor.spec.ts`
- Modify: catalog/seller image components to use a real asset or code/CSS fallback

- [x] Add a protected profile route and header entry.
- [x] Load `GET /me`; support username update plus seller avatar upload/replace/remove.
- [x] Render the avatar from the Media endpoint without persisting image bytes or passwords in browser storage.
- [x] Remove the unsupported category field unless it was deliberately added to the backend.
- [x] Align all frontend validators with backend limits and reject whitespace-only form values.
- [x] Make submit flows resistant to duplicate clicks and lifecycle-safe.
- [x] Scope bearer-token attachment to the API and display useful 401/403/409/validation/upload/network feedback.
- [x] Supply a reliable missing-image fallback.
- [x] Run Angular tests, production build, and dependency audit.
- [x] Suggested commit: `feat(frontend): add seller profile and align API contracts`

### Batch 6: Standardize JWT validation, errors, and production security

- [x] **Batch complete**

**Files:**

- Modify: `user-service/src/main/java/com/example/userservice/security/JwtService.java`
- Modify: `user-service/src/main/java/com/example/userservice/security/SecurityConfig.java`
- Modify: `product-service/src/main/java/com/example/productservice/security/SecurityConfig.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/security/SecurityConfig.java`
- Modify: `api-gateway/src/main/java/com/example/apigateway/security/SecurityConfig.java`
- Modify: `user-service/src/main/java/com/example/userservice/config/AppProperties.java`
- Modify: `product-service/src/main/java/com/example/productservice/config/AppProperties.java`
- Modify: `media-service/src/main/java/com/example/mediaservice/config/AppProperties.java`
- Modify: `api-gateway/src/main/java/com/example/apigateway/config/GatewayProperties.java`
- Modify: each module's `application.yml`
- Modify: each service's `GlobalExceptionHandler.java`
- Create: Product and Media servlet correlation-ID filters matching the Gateway contract
- Modify: `frontend/nginx.conf`
- Create: `frontend/nginx.prod.conf`
- Create: `docker-compose.prod.yml`
- Modify: `.env.example`
- Modify/create: security tests in Gateway, User, Product, and Media test packages

- [x] Generate JWT issuer and audience claims and validate both everywhere.
- [x] Externalize exact allowed origins and use restrictive production CORS.
- [x] Standardize the JSON error contract and propagate/include correlation IDs.
- [x] Add Nginx CSP, nosniff, frame, referrer, and production HSTS headers.
- [x] Add a production HTTPS termination configuration with HTTP redirect and no committed certificates/private keys.
- [x] Document Let's Encrypt certificate issuance/renewal and state that local HTTP is development-only.
- [x] Add security regression tests for missing, malformed, expired, wrong-signature, wrong-issuer, wrong-audience, and forbidden-role cases at the Gateway public boundary.
- [x] Suggested commit: `feat(security): enforce token claims and production transport security`

### Batch 7: Repair and expand automated tests

- [x] **Batch complete**

**Files:**

- Modify: `user-service/src/test/java/com/example/userservice/Buy01ApplicationTests.java`
- Modify: `product-service/src/test/java/com/example/productservice/ProductServiceApplicationTests.java`
- Create: `media-service/src/test/java/com/example/mediaservice/MediaServiceApplicationTests.java`
- Modify/create: focused controller/service/security tests in each backend module
- Modify: `frontend/src/app/shared/guards/role-guard.spec.ts`
- Modify: `frontend/src/app/shared/services/auth.spec.ts`
- Modify: `frontend/src/app/shared/services/media.spec.ts`
- Modify: `frontend/src/app/shared/services/product.spec.ts`
- Modify/create: behavioral specs beside each Angular feature under test
- Create: `scripts/verify.sh`

- [x] Make backend tests self-contained through mocks/slices or Testcontainers with explicit profiles.
- [x] Cover User registration/login/profile conflicts and validation.
- [x] Cover Product validation, CRUD, CLIENT denial, and two-seller ownership.
- [x] Cover Media content/size boundaries, storage compensation, and two-seller ownership.
- [x] Retain and extend Gateway/Discovery tests.
- [x] Fix incorrect Angular imports/injection in the four currently failing specs.
- [x] Add behavioral tests for services, guards, interceptor, catalog, seller CRUD/media, and profile/avatar.
- [x] Make one documented command run every backend test, Angular test, production build, Compose validation, and dependency audit.
- [x] Suggested commit: `test: cover audit-critical user product and media flows`

### Batch 8: Documentation and final audit

- [x] **Batch complete** — automated checks, the base Docker rebuild, documentation reconciliation, and the 2026-09-05 official audit closure are complete. The browser conclusion is supported by the user's confirmed visual retest together with source/test and live-stack evidence; no new destructive fixture flows were needed.

**Files:**

- Modify: `README.md` — health-check inventory, the health-versus-routing-readiness startup window (startup section and Troubleshooting), the official-diagram field mapping, the exclusive-`imageIds` save-time backstop, `server_tokens off;` on both production server blocks, the self-signed-certificate limits of the TLS evidence, and the Compose-overlay gap in `scripts/verify.sh`
- Modify: `media-service/README.md` — `nosniff` is set by Spring Security inside the service and duplicated by Nginx, not supplied by Nginx alone
- Modify: `frontend/src/app/catalog/components/product-card/product-card.scss` and `frontend/src/app/catalog/pages/product-detail/product-detail.scss` — delete the last two dead `category` selectors
- Modify: `frontend/src/app/layout/header/header.spec.ts` — await the root router's initial navigation so its lazy `catalog-module` import cannot resolve after Vitest tears the environment down, which was failing `scripts/verify.sh`
- Modify: `docker-compose.yml` — declare the missing `frontend` health check (landed during this batch's runtime audit)
- Modify: `frontend/nginx.prod.conf` — `server_tokens off;` on the port-80 redirect block (landed during this batch's runtime audit)
- Update: this `PLAN.md` only after checks actually pass

- [x] Document prerequisites, environment variables, local development, Docker startup, architecture, routes, health checks, Eureka inspection, storage, HTTPS, and troubleshooting — all eleven subjects have their own `README.md` section. The health-check section now lists the probe declared for each of the eight services, and Troubleshooting now covers the startup window in which every container reports `healthy` while `/api/**` still answers `503`, including the conditional poll to wait on instead of a fixed sleep.
- [x] Correct README contracts, including email login, Gateway paths, avatar media IDs, and media URLs — login is documented as email-only, Gateway paths are documented unprefixed with the `/api` Nginx rule stated once, `avatarMediaId` is documented as an owned Media ID that must be rendered through `GET /media/images/{id}` rather than treated as a URL, and the upload response's `url` is documented as the browser-origin-relative `/api/media/images/{id}` with the direct-Gateway and source-mode variants called out.
- [x] Document the intentional database mapping and Kafka consistency/orphan limitations — the official-diagram field mapping (`userId`→`sellerId`, `quantity`→`stock`, `imagePath`→`storageKey`, and `imageIds[]` as the canonical relation) is now a table in the README's database section rather than living only in this plan, and the Kafka section lists all five topics with their consumers and eight explicit consistency/orphan limitations.
- [x] Run all automated checks from a clean dependency/build state — current root verification passes 126 backend tests (Gateway 16, Discovery 1, User 31, Product 48, Media 30), 54 Angular tests over 19 files, a successful production build, Compose configuration validation, and `npm audit --omit=dev` with zero vulnerabilities. Scoped independent reviews also passed for the subsequent Batch 8 working-tree fixes.
- [x] Run `docker compose up --build` and wait for every required service to become healthy — the current base Docker rebuild reached eight healthy containers; Nginx root and Nginx/Gateway `/api/products` each returned 200. This runtime smoke evidence is limited to those observations and does not establish the separate interactive audit or a live-container Gateway MDC log proof.
- [x] Register CLIENT, Seller A, and Seller B; test duplicate and invalid registrations — `api-evidence.tsv:register-client`, `:register-seller-a`, `:register-seller-b` (201 each), `:duplicate-email`, `:duplicate-username` (409), `:invalid-registration` (400 with per-field `details`), `:invalid-role-registration` (400 malformed body).
- [x] Browse product list/detail without authentication — `api-evidence.tsv:public-products-gateway`, `:public-products-nginx`, `:public-product-detail` with no `Authorization` header, plus `headless-evidence.tsv:products` and `:product-detail` rendering in a real browser engine.
- [x] Prove CLIENT cannot write products or upload media — `api-evidence.tsv:client-create-product` (403), `:client-upload` (403), `:client-avatar-reference-blocked` (403).
- [x] Prove Seller A can CRUD products, upload valid images, manage associations, and manage an avatar — `api-evidence.tsv:seller-a-jpeg`/`-png`/`-webp` (201), `:seller-a-create-product` (201), `:public-product-detail`, `:seller-a-my-products`, `:seller-a-update-product` (200), `:seller-a-delete-product` (204), and the avatar sequence `:avatar-assign` → `:avatar-replacement-upload` → `:avatar-replace` → `:avatar-remove`.
- [x] Prove Seller B cannot mutate Seller A's products, media, or avatar references — `api-evidence.tsv:seller-b-update-product-blocked` and `:seller-b-delete-product-blocked` (404, existence-masked), `:seller-b-media-metadata-blocked` and `:seller-b-media-delete-blocked` (403), `:seller-b-avatar-reference-blocked` (400), `:seller-b-cross-media-product` (403 — ownership is decided before the association check, which is why this is 403 and not 400).
- [x] Test valid JPEG/PNG/WEBP, exact 2 MiB, over 2 MiB, false MIME, corrupt image, text, executable, and archive uploads — `api-evidence.tsv` rows `seller-a-jpeg`, `seller-a-png`, `seller-a-webp`, `exact-two-mib-png` (201) and `over-two-mib-png`, `false-declared-mime`, `corrupt-image`, `trailing-payload`, `text-file`, `executable-file`, `archive-file` (400 each).
- [x] Verify image rendering, response content headers, volume persistence, deletion cleanup, and Kafka-driven reference cleanup — `api-evidence.tsv:image-render-headers` (`200+2097152+image/png+nosniff`, with the cache and inline-disposition headers recorded), `:image-render-nosniff-via-gateway` and `:image-render-nosniff-via-nginx` attributing the header to Spring Security rather than to Nginx alone, `persistence-evidence.tsv` 10/10 across a `restart` that provably reused the same container IDs (bytes identical by SHA-256, product, product list, referenced image, and BCrypt-backed login all surviving), `:direct-media-delete` (204) with `:kafka-image-deleted-reference-repair` = `REPAIRED`, and `:seller-a-delete-product` (204) with `:kafka-product-deleted-media-cleanup` = `DELETED`.
- [x] Verify malformed/expired/wrong-signature/wrong-issuer/wrong-audience JWT behavior through Gateway and downstream defenses — `jwt-evidence.tsv` 8/8 through the public route, anchored by `local-key-positive-control`. The downstream half is configuration evidence rather than a separate probe: User, Product, and Media each build the same issuer-plus-audience validator over the shared secret (cited in section 3), and they publish no host port in the production overlay, so no external caller can reach them without passing the Gateway first.
- [x] Verify production HTTPS redirect/security headers using the documented deployment configuration — `tls-evidence.tsv` 18/18 against `docker-compose.yml` + `docker-compose.prod.yml` + `frontend/nginx.prod.conf`, including the two verification commands the README documents (`:tls-readme-verification-commands`). Certificate material was a temporary local self-signed pair; see the caveat in section 6.
- [x] Re-run the official 01-edu audit questions and mark checkboxes only from observed results. The 2026-09-05 closure observed eight healthy containers, public frontend/products routes returning 200, direct-Gateway products returning 200, and anonymous protected access returning 401; it reconciled those probes with source/tests and the historical runtime records summarized above. The user also confirmed the browser visual issues were fixed. The closure did not create new audit data, so CRUD/media conclusions retain their recorded-evidence caveat.
- [x] Suggested commit: `docs: add verified setup and audit instructions` — documentation reconciliation is already committed as `c1d18f3 docs: correct setup, contracts, and audit procedure` and `bd65608 docs(plan): reconcile the checklist with audit evidence`.

## Definition of done

- [x] Every checkbox in the official audit checklist is checked with code/test/runtime evidence. The 2026-09-05 closure answered all nine questions Yes or qualified Yes from current live probes, current source/test evidence, historical runtime records, and the user's confirmed browser visual retest. The whitespace-only update rejection, Angular field-level validation display, Gateway/Media correlation logging gaps, stylesheet CSP conflict, and Profile loading resilience gap are resolved.
- [x] Every automated backend and frontend test passes — current root verification reports 126 backend tests over five Maven modules (Gateway 16, Discovery 1, User 31, Product 48, Media 30) and 54 Angular tests over 19 files, with 0 failures, 0 errors, and 0 skipped; the production build, Compose configuration validation, and `npm audit --omit=dev` also pass with zero vulnerabilities.
- [x] Angular production build passes without unexplained warnings or budget failures — the production build succeeds against the configured budgets (`frontend/angular.json:60-71`, initial warn 700 kB / error 1 MB, deliberately set to a realistic value rather than left at the default). The build emits no template or budget warning; the only frontend warning in the whole verification is npm's notice that `@angular/animations@21.2.20` is deprecated, which is a dependency advisory rather than a build defect. The backend warnings seen in the same run are likewise enumerated and attributed rather than suppressed: a deprecated `@Valid`-on-container usage in the Gateway route configuration, Spring Cloud LoadBalancer's development-cache notice, Mockito's dynamic agent self-attachment notice, Netty's macOS native-DNS fallback, the Discovery test's missing Bean Validation provider, and one intentional orphan-cleanup `IOException` logged by a negative-path media test.
- [x] Docker Compose starts the complete stack from a clean checkout using documented steps — `docker compose up --build -d --wait` exactly as documented, run under a fresh project name with empty named volumes and rebuilt images, reaching 8 of 8 healthy and serving traffic. The one thing not literally exercised is a fresh `git clone`: the run used this working tree, which carries the uncommitted changes listed in this batch.
- [x] All external API traffic uses HTTPS in the production configuration and passes through Nginx then Gateway — in the production overlay the only published ports are the frontend's 80 and 443 (`docker-compose.prod.yml`, `ports: !reset []` on Mongo, Kafka, Discovery, and the Gateway), port 80 answers only with a 301, and `/api` requests traverse Nginx to the Gateway over TLS (`tls-evidence.tsv:tls-api-reaches-gateway`, `:tls-api-security-headers`, `:tls-api-unauth-error-contract`). Proven for the configuration using a temporary local self-signed certificate; no real CA, DNS name, or deployment host was involved, so a deployed environment must still be verified with the three commands in the README's HTTPS section.
- [x] The CLIENT/Seller A/Seller B official-audit closure is supported by the historical machine-verified `api-evidence.tsv` (59/59 covering all three personas) and `jwt-evidence.tsv` (8/8) records summarized in this plan, current source/test evidence, live probes that reconfirmed public and protected boundaries, and the user's confirmed browser visual retest; it does not claim an independently repeated complete three-persona manual UI audit. No new accounts, products, or media were created during the closure, so persona CRUD/media behavior is supported by existing evidence rather than new destructive fixture flows.
- [x] No secrets, uploaded test media, database data, build output, or dependency directories are tracked — `git ls-files` matches nothing against `\.pem$`, `\.key$`, `media-storage`, `^\.env$`, `node_modules`, `target/`, or `dist/`. `.env` is ignored at `.gitignore:18` and build/dependency output at `.gitignore:2,5,6`. The only lock file tracked is `frontend/package-lock.json`, which is intentional and required by `npm ci`. All runtime-audit evidence and task reports are untracked under `.superpowers/`.
- [x] `README.md` matches the actual commands, routes, data contracts, and limitations — reconciled against the code and configuration in this batch. The documented startup, health, log, and teardown commands were exercised during the runtime audit; `bash scripts/verify.sh` was re-run for this revision; and the two HTTPS verification commands were exercised over TLS (`tls-evidence.tsv:tls-readme-verification-commands`). Limitations are stated rather than implied: the routing-readiness startup window, the Kafka consistency and orphan boundaries, the exclusive-`imageIds` save-time backstop, the self-signed scope of the TLS evidence, and the Compose overlays that `scripts/verify.sh` does not validate.
