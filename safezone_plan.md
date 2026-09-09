# SafeZone Plan — SonarQube Code Quality & Security Integration

**Source requirement:** [safezone.md](./safezone.md) — Set up SonarQube for code quality & security checks, integrate with GitHub (via Actions or webhooks), fail CI when issues are detected, and implement code review/approval process.

**Goal:** stand up SonarQube against the buy-01 microservices codebase, wire it into the implemented Jenkins CI/CD pipeline, and make the pipeline fail when quality/security issues are detected — with continuous monitoring and a review/approval process on top.

**CI/CD status:** Jenkins is now implemented locally in `Jenkinsfile`, `jenkins-compose.yaml`, and `scripts/deploy-staging.sh`. The existing pipeline already checks out the private repository, builds/tests the five backend services and frontend, archives artifacts, deploys to staging, performs rollback, and sends email notifications. This plan extends that pipeline with SonarQube analysis and Quality Gate enforcement. GitHub Actions remains an optional alternative, not the assumed working path.

---

## Verified Repo Baseline (2026-09-07)

### Services & Build Tooling
- **5 Spring Boot / Maven / Java 21 services:** `user-service`, `product-service`, `media-service`, `api-gateway`, `discovery-service`
  - All have `pom.xml` files
  - `user-service`, `product-service`, `media-service` have Maven wrappers (`mvnw`/`mvnw.cmd`) ✓
  - `api-gateway`, `discovery-service` do **NOT** have wrappers — **will need to add them** or install Maven globally during CI
- **Tests verified passing (offline mode `-o`):**
  - user-service: 11 tests, all passing ✓
  - product-service: 26 tests, all passing ✓
  - media-service: 24 tests, all passing ✓
  - api-gateway: has tests (none run yet in this review, but service exists)
  - discovery-service: has tests (none run yet in this review, but service exists)
- **No JaCoCo or SonarQube plugins** in any `pom.xml` yet — all 5 services are ready for Phase 2 additions with no migration needed

### Frontend
- Angular 21.2.20 with **Vitest** (not Karma/Jasmine) — confirmed in `package.json`
- Build: `npm run build` ✓
- Tests: `npm test` (Vitest via `@angular/build:unit-test` builder) — **18 tests, all passing** ✓
- Coverage output: `ng test --coverage` produces `lcov.info` in `coverage/` directory
- No `.github/workflows` anywhere in the repo (only `frontend/.github/copilot-instructions.md`, which is unrelated to CI)

### Infrastructure & Secrets
- Docker 28.1.1 and Docker Compose v2.35.1 installed ✓
- Existing [docker-compose.yml](./docker-compose.yml) runs: mongo (27018), kafka (9092), 5 services (8080-8083, 8761), frontend (4200)
- Ports 9000 (SonarQube) and 5432 (Postgres) are **free** (not in use by app stack)
- Root [.gitignore](./.gitignore) excludes `.env` — safe for storing SonarQube token
- No `.github/workflows` or Makefile exists; Jenkins is implemented in `Jenkinsfile` and `jenkins-compose.yaml`

### Implemented Jenkins baseline
- `Jenkinsfile` is the current CI/CD entry point and polls `main` every two minutes because the local controller is not publicly reachable for GitHub webhooks.
- `jenkins-compose.yaml` provides Jenkins, a TLS-protected Docker-in-Docker daemon, and persistent volumes.
- The current pipeline runs backend `clean verify`, frontend `npm ci`/tests/build, JUnit publication, artifact archiving, optional staging deployment, rollback, and email notifications.
- SonarQube scanning, coverage publication to SonarQube, and Quality Gate failure handling are **not implemented yet**.
- The current pipeline uses `agent any` and a configured NodeJS tool named `nodejs-22-lts`; SonarQube work must add the required scanner/plugin configuration without breaking those existing stages.

---

---

## Phase 0 — Baseline check & prepare
- [ ] **Confirm all 5 services build:**
  - `user-service`, `product-service`, `media-service`: run `.\mvnw.cmd -o -DskipTests package` per service (wrappers exist)
  - `api-gateway`, `discovery-service`: run `.\mvnw.cmd -o -DskipTests package` per service — **wrappers missing**, will need to either:
    - Option A: Add Maven wrappers to both (recommended for consistency) via `mvn wrapper:wrapper` or copy from another service
    - Option B: Install Maven globally and use `mvn` in CI scripts instead
  - Frontend: `npm run build`
- [ ] Confirm ports `9000` (SonarQube) and `5432` (Postgres DB) are free alongside the existing app stack (done: confirmed free)

## Phase 1 — Local SonarQube via Docker
- [ ] Add a **separate** `docker-compose.sonar.yml` (kept apart from the app stack so it can run/stop independently) with:
  - `sonarqube` (Community/Developer Edition) container
  - Postgres backing DB
  - named volumes for data / extensions / logs
- [ ] Start it with `docker compose -f docker-compose.sonar.yml up -d`
- [ ] Log into `http://localhost:9000`, rotate the default admin password
- [ ] Generate a global analysis token, store it only in `.env` (already gitignored) — never commit it

## Phase 2 — Backend scanner configuration (per Spring Boot service)
- [ ] Add `sonar-maven-plugin` + `jacoco-maven-plugin` to each of the 5 `pom.xml` files so unit-test coverage is generated and picked up by Sonar
- [ ] Give each service its own `sonar.projectKey` (e.g. `buy-01-user-service`, `buy-01-product-service`, …)
- [ ] Verify a manual `mvn verify sonar:sonar -Dsonar.host.url=... -Dsonar.token=...` run per service pushes results into the SonarQube UI

## Phase 3 — Frontend scanner configuration
- [ ] Add `sonar-project.properties` (or an npm script wrapping the `sonar-scanner` CLI) for `frontend/`
- [ ] Wire it to the Vitest-based `lcov.info` coverage output (`ng test --coverage`, via the `@angular/build:unit-test` builder — **not** Karma/Jasmine) and ESLint results
- [ ] Give it project key `buy-01-frontend`

## Phase 4 — Jenkins CI/CD integration
- [ ] Add SonarQube server configuration in Jenkins and store the analysis token as a masked Jenkins credential; do not put it in `Jenkinsfile`, `.env`, or build logs.
- [ ] Add the SonarQube environment/`withSonarQubeEnv` integration to `Jenkinsfile`.
- [ ] Extend the existing backend stages to generate JaCoCo coverage and run `sonar:sonar` for all five Maven services.
- [ ] Add the frontend Sonar scanner configuration and run it after the existing frontend tests/coverage generation.
- [ ] Add a Jenkins `waitForQualityGate` stage that aborts/fails the build when the SonarQube gate is not green, before `Deploy to Staging`.
- [ ] Preserve the existing JUnit publication, artifact archiving, deployment, rollback, email notifications, and controlled-failure audit parameters.
- [ ] Configure a SonarQube webhook to the Jenkins endpoint required by `waitForQualityGate`; document the local-controller limitation and the externally reachable URL required for a shared deployment.
- [ ] Keep scanner commands in reusable scripts where practical so the same analysis steps can optionally be invoked from GitHub Actions later.

## Phase 5 — Quality Gate enforcement
- [ ] Define/assign a Quality Gate (Sonar Way baseline plus coverage-on-new-code threshold, 0 new bugs/vulnerabilities) to all buy-01 projects
- [ ] Prove the pipeline actually fails CI on an intentionally bad commit, then passes once fixed

## Phase 6 — GitHub branch protection & review process
- [ ] Make the Jenkins Quality Gate result (or the retained GitHub Actions adapter) a required status check on the default branch
- [ ] Require at least one PR approval before merge (branch protection rule / CODEOWNERS)

## Phase 7 — Continuous monitoring
- [ ] Confirm Jenkins SCM polling gives ongoing analysis for changes to `main`
- [ ] Configure a reachable GitHub webhook or a scheduled Jenkins job for pull-request analysis before relying on the Quality Gate for PR merges
- [ ] Optionally add a scheduled scan for periodic full re-scans of the default branch

## Phase 8 — Bonus
- [ ] SonarQube webhook → Slack/email (or a GitHub Actions notification step) on Quality Gate failure
- [ ] Document SonarLint IDE setup (VS Code/IntelliJ, connected mode) for real-time feedback during development

## Phase 9 — Documentation & end-to-end verification
- [ ] Document run steps in README.md or this file
- [ ] Run the full loop for real: start SonarQube → push a commit → watch the Jenkins run → see the Quality Gate result on the GitHub repository/PR
- [ ] Re-confirm every `safezone.md` testing criterion is met

---

## Definition of Complete

The SafeZone project is complete when **all** of the following are true and demonstrable, directly mapping to [safezone.md](./safezone.md)'s Testing section:

### 1. Successful setup and configuration of SonarQube using Docker
- [ ] SonarQube Community/Developer Edition (+ Postgres backing DB) runs via `docker compose -f docker-compose.sonar.yml up -d` with data/extensions/logs persisted in named volumes
- [ ] Container survives restart without losing configuration or analysis results
- [ ] Default admin credentials rotated (no admin/admin)
- [ ] Global analysis token generated and stored in `.env` (gitignored), never committed to repo

### 2. Integration of SonarQube with GitHub repository and CI/CD pipeline
- [ ] Jenkins checks out the private GitHub repository from `main` using a managed credential.
- [ ] Jenkins successfully builds all 5 services with test coverage and SonarQube analysis (`mvn verify sonar:sonar` per service or equivalent).
- [ ] Jenkins successfully runs frontend build, tests, coverage, and SonarQube analysis.
- [ ] Jenkins uses a masked credential for `SONAR_TOKEN` and a configured SonarQube server URL.
- [ ] Jenkins waits on the SonarQube Quality Gate and **fails the build** if the gate is not green (proven by at least one observed failure case).
- [ ] At least one successful end-to-end Jenkins run is visible in build history with all projects analyzed.
- [ ] If GitHub branch protection requires a repository status check, publish Jenkins build status to GitHub through the GitHub integration/plugin or retain a GitHub Actions status-check workflow as a thin adapter.

### 3. Effective code analysis and detection of code quality and security issues
- [ ] All 6 projects (5 services + frontend) appear in SonarQube dashboard with real, non-zero analysis (lines of code, issues, coverage %)
- [ ] Test coverage is visible and non-empty for each project (JaCoCo for Java, Vitest lcov for Angular)
- [ ] Quality Gate is configured with at least: 0 new bugs, 0 new vulnerabilities, coverage threshold on new code
- [ ] A deliberately-introduced issue (e.g., hardcoded password, security smell, missing coverage) causes the Quality Gate to fail and is caught by the pipeline
- [ ] Reverting/fixing that issue makes the Quality Gate pass — both outcomes have been observed in real CI runs

### 4. Implementation of code review and approval process
- [ ] Branch protection rule on the default branch requires **both**:
  - The Jenkins Quality Gate status check (or GitHub Actions adapter) to pass
  - At least one human code review + approval (PR approval requirement)
- [ ] Enforce via GitHub Settings > Branches (not just documentation)

### 5. Bonus (optional, not blocking "complete" per safezone.md)
- [ ] Email or Slack notification configured on SonarQube to fire when Quality Gate fails
- [ ] SonarLint IDE integration documented for VS Code or IntelliJ in connected mode

### 6. Jenkins operational readiness
- [ ] Jenkins starts from `jenkins-compose.yaml`, retains its configuration and Docker-in-Docker volumes across restart, and can run the existing CI pipeline.
- [ ] SonarQube credentials and server configuration are managed in Jenkins and are masked from console output.
- [ ] The SonarQube webhook reaches Jenkins and `waitForQualityGate` returns the actual analysis result.
- [ ] All build/test/coverage/sonar-scan logic is documented and reusable outside Jenkins where practical.

---

## Optional GitHub Actions alternative
If Jenkins is unavailable or the repository later moves to hosted CI:
1. Add a `.github/workflows/sonarqube.yml` workflow for `push` and `pull_request`.
2. Invoke the documented reusable build/scan commands with `SONAR_TOKEN` and `SONAR_HOST_URL` GitHub secrets.
3. Wait for the Quality Gate and publish the workflow status as the required branch-protection check.
