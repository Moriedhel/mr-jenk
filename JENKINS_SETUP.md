# Jenkins CI/CD Setup and Operations

This guide reproduces the local Jenkins environment used to build, test, deploy, roll back, and audit Buy-01. Run commands from the repository root. Never commit passwords, tokens, `.env` files, or Jenkins backups.

## Architecture

```text
GitHub private repository
        |
        | SCM polling every two minutes
        v
Jenkins controller :8080
        |
        | TLS-protected Docker API
        v
Docker-in-Docker staging daemon
        |
        +-- Buy-01 frontend :4200
        +-- Buy-01 gateway  :8081
```

Jenkins currently uses its built-in executor (`agent any`). This is suitable for the local coursework environment; an externally exposed or shared installation should use dedicated agents and should not run untrusted builds on the controller.

## Prerequisites

- Docker Desktop or Docker Engine with Compose
- Git
- Access to the private GitHub repository
- A Gmail account with 2-Step Verification and an App Password for SMTP
- Ports 8080, 8081, 4200, and optionally 50000 available

## Start and unlock Jenkins

The controller image is pinned in `jenkins/Dockerfile`. `jenkins-compose.yaml` provides persistent controller, certificate, and nested-Docker volumes.

```powershell
docker compose -f jenkins-compose.yaml up -d --build
docker compose -f jenkins-compose.yaml ps
```

Open `http://localhost:8080`. On a first installation, retrieve the one-time unlock password locally:

```powershell
docker exec jenkins sh -c "cat /var/jenkins_home/secrets/initialAdminPassword"
```

Do not save that output in the repository. Complete the setup wizard and create a named administrator.

## Required plugins and tools

Install plugins from **Manage Jenkins → Plugins**:

- Pipeline and Git
- Credentials Binding
- JUnit
- NodeJS
- Email Extension
- Matrix Authorization Strategy

Under **Manage Jenkins → Tools**, configure a NodeJS installation named exactly `nodejs-22-lts` and select Node.js 22 LTS. Java 21, Git, Docker CLI, and Docker Compose are supplied by the controller image.

Verify Docker connectivity:

```powershell
docker exec jenkins docker version
docker exec jenkins docker compose version
```

## Credentials

Create credentials under **Manage Jenkins → Credentials → System → Global credentials**. Values must never be placed in this document or the repository.

| ID or use | Type | Scope and purpose |
|---|---|---|
| `github-mr-jenk-read` | Username/password or supported GitHub token credential | Read-only checkout of the private repository |
| `buy01-jwt-secret` | Secret text | Staging JWT signing secret, injected only during deployment |
| Gmail SMTP | SMTP username and Google App Password | Jenkins email transport; stored in Jenkins system configuration |

Rotate a credential by replacing it in Jenkins while retaining the ID expected by the job, then run a non-production verification build. Revoke the old GitHub token or Google App Password at its provider. Rotate the JWT secret only during a controlled staging redeployment because all token-issuing and token-validating services must use the same value. Never record old or new values in tickets, logs, screenshots, or Git.

## Job configuration

Create a Pipeline job named `MrJecks` and select **Pipeline script from SCM**:

- SCM: Git
- Repository: `https://github.com/SManousis/mr-jenk`
- Credential: `github-mr-jenk-read`
- Branch: `*/main`
- Script path: `Jenkinsfile`

The repository is private and Jenkins is bound to localhost, so GitHub cannot call a webhook. The `Jenkinsfile` uses `pollSCM('H/2 * * * *')` and starts a build only when `main` changes.

## Pipeline parameters

| Parameter | Safe default | Purpose |
|---|---:|---|
| `DEPLOY_ENV` | `staging` | Select staging or no deployment environment |
| `SKIP_DEPLOY` | `true` | Prevent deployment during ordinary CI runs |
| `FORCE_BUILD_FAILURE` | `false` | Audit-only controlled early failure |
| `FORCE_TEST_FAILURE` | `false` | Audit-only failing JUnit test used to verify report publication and deployment blocking |
| `FORCE_DEPLOYMENT_FAILURE` | `false` | Audit-only unhealthy release used to test rollback |

Keep all failure parameters disabled outside an intentional audit test. A developer can start builds but only an administrator can modify the job, Jenkinsfile source, or credentials.

## Build, test, and artifacts

The pipeline runs the five Maven service builds and the Angular build/tests in parallel. It publishes Maven JUnit XML and archives service JARs plus `frontend/dist/**`, with fingerprints. Builds are serialized and the most recent 20 are retained.

A normal CI run uses:

```text
DEPLOY_ENV=none
SKIP_DEPLOY=true
FORCE_BUILD_FAILURE=false
FORCE_TEST_FAILURE=false
FORCE_DEPLOYMENT_FAILURE=false
```

## Staging deployment and verification

To deploy through **Build with Parameters**:

```text
DEPLOY_ENV=staging
SKIP_DEPLOY=false
FORCE_BUILD_FAILURE=false
FORCE_TEST_FAILURE=false
FORCE_DEPLOYMENT_FAILURE=false
```

`scripts/deploy-staging.sh` builds commit-SHA-tagged images, starts the stack in the isolated daemon, waits with bounded retries, and smoke-tests the frontend. Success is recorded only after verification.

- Frontend: `http://localhost:4200`
- Gateway health: `http://localhost:8081/actuator/health`

## Automatic and manual rollback

The deployment script stores the last successful image tag in the persistent Jenkins workspace at `.jenkins-state/staging-last-successful-tag`. If readiness or the smoke test fails, it restores those existing images with `--no-build`, verifies service health, and leaves the attempted pipeline result as `FAILURE`.

For the controlled audit test, set `FORCE_DEPLOYMENT_FAILURE=true`. Confirm the console contains both the rollback-success message and `Finished: FAILURE`, then confirm port 4200 remains healthy.

If automation is unavailable, an administrator can perform a manual rollback from the Jenkins workspace:

1. Read the last successful tag from `.jenkins-state/staging-last-successful-tag`.
2. Securely supply the same staging `JWT_SECRET` used by the services.
3. From the repository workspace, run the two Compose files with project `buy01-staging`, set `IMAGE_TAG` to the recorded tag, and execute `up --detach --no-build --remove-orphans`.
4. Verify every container is healthy, request the frontend, and request the gateway health endpoint.
5. Record the restored tag and reason without recording the secret.

Do not delete the nested Docker data volume during recovery; it contains the previously built images required by `--no-build`.

## Email notifications

Under **Manage Jenkins → System**, configure both **E-mail Notification** and **Extended E-mail Notification**:

- SMTP server: `smtp.gmail.com`
- Port: `465`
- SSL: enabled
- Username: full Gmail address
- Password: 16-character Google App Password without spaces
- Default recipients: intended notification address

The pipeline sends one final-status email for success, failure, unstable, or aborted runs. Failure messages attach a compressed console log. Notification content includes job, build, commit, environment, duration, and build URL, but no credential values.

## Authentication and authorization

Use Jenkins' own user database with signup disabled and Matrix Authorization Strategy:

| Identity | Permissions |
|---|---|
| `stamatis` | Overall/Administer |
| `developer` | Overall/Read, Job/Read, Job/Build, Job/Cancel, View/Read |
| `auditor` | Overall/Read, Job/Read, View/Read |
| anonymous | None |

Keep CSRF protection enabled. This instance is localhost-only HTTP; place Jenkins behind authenticated HTTPS before exposing it outside the trusted machine.

## Restart and persistence

A normal restart preserves configuration, users, credentials, jobs, build history, deployment state, certificates, and staging images in named volumes:

```powershell
docker compose -f jenkins-compose.yaml restart
docker compose -f jenkins-compose.yaml ps
```

After restart, sign in, open `MrJecks`, confirm its history and credentials exist, and run the Docker connectivity checks above.

## Backup and restore

Treat a Jenkins-home backup as highly sensitive: it contains encrypted credentials and the keys needed to decrypt them. Store it outside the repository with restricted access. Stop Jenkins before taking a consistent volume backup. Back up all three named volumes shown in `jenkins-compose.yaml` when staging images and Docker TLS state must also be recoverable.

Before restoring, stop the Compose project, preserve the current volumes as a fallback, restore the archive into the matching named volume, start the project, and verify login, credentials, job history, Docker connectivity, and a safe CI build. Never test restore by overwriting the only working copy.

## Troubleshooting

- **No automatic build:** wait for the two-minute SCM poll; verify the job tracks `*/main` and the GitHub credential still works.
- **Jenkinsfile not found:** set Script Path to `Jenkinsfile` and verify it is at the repository root.
- **Docker unavailable:** run the two Docker verification commands and check that `jenkins-docker` is healthy.
- **Node shared-library error:** configure the supported Node.js 22 LTS tool and rebuild the controller if its required OS libraries are missing.
- **M wrapper permission denied:** invoke the wrapper with `sh ./mvnw`, as the pipeline does.
- **SMTP tries localhost:25:** configure the basic **E-mail Notification** section as well as **Extended E-mail Notification**.
- **SMTP authentication fails:** use a Google App Password without spaces, not the normal Google password or six-digit authenticator code.
- **403 no valid crumb:** reload the Jenkins form, avoid stale tabs, and keep CSRF protection enabled.
- **Long staging build:** the first nested-Docker image build can take 10–15 minutes; monitor stage output rather than starting a duplicate build.

## Audit evidence

The reproducible test matrix and verified build numbers are maintained in `PLAN.md`. Key evidence includes automatic checkout/build, retained tests and artifacts, successful staging deployment, controlled build failure and recovery, automatic rollback, notification delivery, matrix authorization tests, and masked credential injection.
