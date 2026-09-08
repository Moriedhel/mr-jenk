# Buy-01 CI/CD Agent Handoff

This file is the continuity guide for AI coding agents and developers working on this repository. Read it completely before changing the project.

## Start here

Read these files in order:

1. `project.md` — assignment requirements.
2. `audit-quesstions.md` — evaluator questions (filename is intentionally retained as currently committed).
3. `PLAN.md` — detailed checklist and evidence log; this is the source of truth for progress.
4. `JENKINS_SETUP.md` — reproducible Jenkins operations and migration/setup guide.
5. `Jenkinsfile` — active pipeline definition.

The canonical working repository is:

```text
https://github.com/SManousis/mr-jenk
```

The tracked branch is `main`. The original Zone01 Buy-01 repository is upstream source material, not the Jenkins SCM target.

## Project architecture

Buy-01 is an Angular 21 frontend plus five Java 21/Spring Boot Maven services:

- `api-gateway`
- `discovery-service`
- `user-service`
- `product-service`
- `media-service`
- `frontend`

MongoDB, Kafka, and Eureka support the application. Application Compose files are `docker-compose.yml` and `docker-compose.ci.yml`.

Jenkins runs through:

- `jenkins-compose.yaml`
- `jenkins/Dockerfile`
- Jenkins image based on `jenkins/jenkins:2.568.3-lts-jdk21`
- Isolated `docker:28.4.0-dind` staging daemon
- Jenkins UI at `http://localhost:8080`
- Staging frontend at `http://localhost:4200`
- Staging gateway at `http://localhost:8081`

Do not replace the isolated Docker-in-Docker deployment with the host Docker socket without an explicit design decision and security review.

## Jenkins job and tools

- Job name: `MrJecks`
- Pipeline source: private GitHub repository above
- Script path: `Jenkinsfile` at repository root, with no extension
- Branch: `*/main`
- Trigger: `pollSCM('H/2 * * * *')`; localhost cannot receive a GitHub webhook
- NodeJS tool name: `nodejs-22-lts`
- Builds use the controller's built-in executor (`agent any`)
- Build retention: most recent 20 builds
- Concurrency: disabled for this job
- Timeout: 60 minutes

Required Jenkins plugins include Pipeline, Git, Credentials Binding, JUnit, NodeJS, Email Extension, and Matrix Authorization Strategy.

## Credentials and sensitive data

Never commit, display, request, or reproduce credential values.

Known identifiers and uses:

- `github-mr-jenk-read` — read-only private GitHub checkout credential.
- `buy01-jwt-secret` — Secret Text credential injected only into staging deployment.
- Gmail SMTP — configured in Jenkins system email settings using a Google App Password.

The Gmail address, App Password, GitHub token, JWT value, and Jenkins account passwords are intentionally absent from Git. `.env`, Jenkins state, build output, and common secret files must remain ignored.

Build #23 showed:

```text
Masking supported pattern matches of $JWT_SECRET
```

A non-disclosing repository/history scan found no recognized GitHub token/private-key signatures or tracked secret files. Do not run commands that print suspected secret values; report only paths and classifications.

## Authentication and authorization

Jenkins uses its own user database, signup is disabled, CSRF protection remains enabled, and Matrix Authorization Strategy is configured.

| Identity | Access |
|---|---|
| `stamatis` | Overall/Administer |
| `developer` | Overall/Read, Job/Read, Job/Build, Job/Cancel, View/Read |
| `auditor` | Overall/Read, Job/Read, View/Read |
| anonymous | None |

Incognito tests confirmed these boundaries. Do not weaken them. Jenkins is localhost-only HTTP; use authenticated HTTPS before any external exposure.

## Pipeline parameters and safe defaults

| Parameter | Safe value |
|---|---|
| `DEPLOY_ENV` | Use `none` for CI-only verification; `staging` only for intended deployment |
| `SKIP_DEPLOY` | `true` |
| `FORCE_BUILD_FAILURE` | `false` |
| `FORCE_TEST_FAILURE` | `false` |
| `FORCE_DEPLOYMENT_FAILURE` | `false` |

All force-failure parameters are audit controls and must remain disabled by default.

## Verified audit evidence

The detailed evidence is in `PLAN.md`. Important Jenkins runs include:

| Build | Result and evidence |
|---|---|
| #7 | Successful backend/frontend CI with reports and artifacts |
| #10 | Successful automatic SCM-triggered build |
| #13 | Successful staging deployment and health/smoke verification |
| #15 | Controlled deployment failure; automatic rollback restored healthy staging |
| #17 | Successful-build email delivered |
| #20 | Controlled early build failure; failure email and compressed log delivered |
| #21 | Successful recovery build and email after #20 |
| #23 | Successful staging deployment email and JWT masking evidence |
| #25 | Controlled deployment failure; rollback succeeded, port 4200 stayed healthy, failure email/ZIP delivered |
| #30 | Successful default build proving controlled JUnit failure is disabled |
| #31 | Controlled JUnit failure published in Test Result, returned exit code 1, skipped deployment, and sent failure email/ZIP |

Do not invent build evidence or mark a checkbox merely because code exists. Mark it only after observing the corresponding Jenkins result.

## Deployment and rollback

`scripts/deploy-staging.sh` builds commit-SHA-tagged images, deploys into the isolated daemon, waits for service health, and smoke-tests the frontend. The last successful tag is stored in:

```text
.jenkins-state/staging-last-successful-tag
```

That state lives in persistent Jenkins storage and is ignored by Git. On failed health or smoke checks, the script restores the previous images with `--no-build`, verifies them, and intentionally leaves the attempted pipeline result as `FAILURE`.

Never delete `mr-jenk_jenkins-docker-data` during ordinary cleanup because it contains rollback images.

## Notifications

Email Extension sends one final-status email for successful, failed, unstable, or aborted runs. Failure email includes a compressed console log. Global default recipients are configured in Jenkins, not Git.

SMTP uses `smtp.gmail.com`, SSL port 465, the full Gmail address as username, and a Google App Password without spaces. Never use or expose the normal Google password.

## Current work status

Completed and evidenced:

- Private GitHub SCM checkout
- Automatic SCM polling trigger
- Parallel backend/frontend CI
- JUnit publication and artifact archival
- Successful staging deployment
- Health and smoke verification
- Automatic rollback
- Success, failure, recovery, deployment, and rollback-related email delivery
- Matrix authorization and anonymous-access restrictions
- Credential injection/masking and repository secret scan
- Controlled build-failure and JUnit-failure audits
- Jenkins setup and operations documentation
- Reconciled core audit checklist and evidence table in `PLAN.md`

### True remaining required work

1. Run `git status` and preserve all existing user changes.
2. Review the installed Jenkins plugin list for necessity and maintenance warnings; do not upgrade during the audit unless a specific issue requires it.
3. On the second laptop, clone the repository and run the documented local build/test commands from a clean checkout.
4. Follow `JENKINS_SETUP.md` as a clean-room Jenkins setup, then run a safe CI-only build.
5. Record the new build number and any documentation corrections in `PLAN.md`.
6. Mark the clean-room Definition of Done and Phase 10 items only after that independent walkthrough succeeds.

The core assignment and every non-bonus audit question already have implementation and evidence. The only required unfinished work is the plugin review and clean-room reproducibility walkthrough. A custom build display name is deliberately not required because logs and emails already capture job, build, commit, environment, duration, and URL.

### Optional work

- Distributed/multiple Jenkins agents are bonus-only and are not implemented.
- Code coverage, static analysis, external registry publishing, production deployment, and public webhook/HTTPS exposure are out of the current assignment scope.
- The localhost job already runs backend and frontend stages in parallel on the built-in executor.

Avoid another 10–15 minute staging deployment unless it closes a specific unverified requirement.

## Continuing on another laptop

A Git clone transfers source code and documentation, but it does **not** transfer Jenkins users, credentials, job history, build evidence, Gmail SMTP configuration, named Docker volumes, staging images, or `.jenkins-state`.

On the new laptop, choose one of these approaches:

### Rebuild from documentation

1. Clone the GitHub repository.
2. Follow `JENKINS_SETUP.md` from the beginning.
3. Recreate credentials using the same IDs but new or securely transferred values.
4. Recreate the `MrJecks` job and authorization matrix.
5. Run a safe CI-only build before any staging deployment.

This is the preferred clean-room reproducibility test.

### Migrate existing Jenkins state

Back up and restore the named volumes described in `JENKINS_SETUP.md`. Jenkins-home backups are highly sensitive because they contain encrypted credentials and the decryption keys. Transfer them only through encrypted storage, keep a fallback copy, stop Jenkins for a consistent backup, and never commit an archive to this repository.

If historical Jenkins URLs/screenshots are needed for the audit, retain the original laptop until the evidence has been exported or the volumes have been verified on the new machine.

## Working rules

- Preserve user changes and inspect `git status` before edits.
- Use `apply_patch` for manual file edits.
- Keep secrets out of source, logs, documentation, screenshots, and chat.
- Keep infrastructure changes reproducible and version controlled.
- Use immutable commit-SHA image tags for staging releases.
- Keep controlled failure parameters false by default.
- Update `PLAN.md` after verified work, including exact Jenkins build numbers.
- Validate changes proportionally (`git diff --check`, relevant tests, Compose config, then Jenkins).
- Do not force-push, rewrite history, delete volumes, or reset the working tree without explicit user authorization.
