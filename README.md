# DevOps Camp Pipeline

A fully automated, end-to-end CI/CD pipeline built as part of a **DevSecOps training program** for engineers supporting the **U.S. Department of Education's Federal Student Aid (FSA)** systems, under the **Accenture Federal Services (AFS)** engagement. This project served as a hands-on capstone for ops engineers learning modern DevSecOps workflows — from source control to container security scanning to production deployment.

---

## Background

This pipeline was developed as part of **DevOps Camp**, an internal DevSecOps training initiative run by **Accenture Federal Services (AFS)** for engineers supporting the **U.S. Department of Education's Federal Student Aid (FSA)** portfolio. AFS was a government IT contractor, and the engineers going through this program were ops staff — including subcontractors — responsible for managing the systems that process and deliver federal student aid to millions of Americans.

The goal of the program was to upskill those engineers on the tools and practices that define a modern, security-conscious software delivery pipeline: containerization, CI/CD automation, private artifact management, container image vulnerability scanning, and Kubernetes-based deployments.

Each participant built and maintained their own version of this pipeline as part of the curriculum, connecting a live application repository to a fully automated delivery system backed by Jenkins, Harbor, Nexus, and Kubernetes.

---

## Project Overview

This repository contains the pipeline definitions and supporting automation scripts that drive the build, scan, and deployment lifecycle for a Python web application backed by a PostgreSQL database.

The pipeline is split into two distinct flows:

| Pipeline | File | Purpose |
|---|---|---|
| Application Pipeline | `devops-camp-jenkinsfile` | Builds, scans, and deploys the app container |
| Database Pipeline | `devops-camp-db-jenkinsfile` | Conditionally builds, scans, and deploys the DB container only when the schema has changed |

---

## Architecture

```
GitHub (afs-labs-student)
        |
        v
   Jenkins Agent
   /           \
  /             \
App Pipeline    DB Pipeline
  |               |
  v               v
Docker Build    Check Harbor for DB changes (check_harbor_db.py)
  |               |
  v           (if changed)
Push to           |
Harbor         Docker Build
Registry          |
  |               v
  v            Push to Harbor Registry
Harbor           |
Security         v
Scan          Harbor Security Scan
(harbor_scanner.py)
  |               |
  v               v
Kubernetes    Kubernetes
app-deployment  db-deployment
```

### Infrastructure Components

| Component | Role |
|---|---|
| **Jenkins** | Orchestrates all pipeline stages via agents |
| **Harbor** (`harbor.dev.afsmtddso.com`) | Private container registry + vulnerability scanning (AFS MTDDSO-hosted) |
| **Nexus** (`nexus.dev.afsmtddso.com`) | Private PyPI proxy for Python dependency management |
| **Kubernetes** | Container orchestration — deploys to namespace `jmedina` |
| **GitHub** | Source of truth for application and database source code |

---

## Repository Structure

```
devops-camp-pipeline/
├── devops-camp-jenkinsfile       # CI/CD pipeline for the application
├── devops-camp-db-jenkinsfile    # CI/CD pipeline for the database
├── harbor_scanner.py             # Triggers and reports on Harbor vulnerability scans
├── check_harbor_db.py            # Determines whether a new DB image build is needed
├── app/
│   └── Dockerfile                # Container definition for the Python web app
└── db/
    └── Dockerfile                # Container definition for the PostgreSQL database
```

---

## Application Pipeline (`devops-camp-jenkinsfile`)

The application pipeline runs on every push and handles the full lifecycle of the web application container.

### Stages

1. **Application Repository**
   Clones the student application repo (`afs-labs-student`) and captures the current Git commit hash. This hash is used to tag the resulting Docker image, creating a direct traceability link between the deployed image and the source commit.

2. **Application Docker Build**
   Builds the app container image using `./app/Dockerfile` against the cloned source. The image is tagged with a compound version identifier:
   ```
   <app-commit-hash>-<pipeline-commit-hash>
   ```
   This dual-hash strategy ensures both the application source version and the pipeline version are captured in every image tag. The image is pushed to Harbor.

3. **Security Scanning**
   Invokes `harbor_scanner.py` to trigger a Trivy-backed vulnerability scan of the pushed image via Harbor's API. The pipeline polls for scan completion and prints a JSON summary of findings. See the [Harbor Security Scanning](#harbor-security-scanning) section for full details.

4. **Test**
   Placeholder stage for integration and unit tests.

5. **Deploy**
   Performs a rolling update in Kubernetes by patching the `app-deployment` image reference:
   ```bash
   kubectl -n jmedina set image deployment/app-deployment app-deployment=<image>
   kubectl -n jmedina apply -f ./afs-labs-student/kubernetes/config-map.yaml
   ```

6. **Cleanup**
   Wipes the Jenkins workspace after every run to prevent stale artifacts from affecting future builds.

---

## Database Pipeline (`devops-camp-db-jenkinsfile`)

The database pipeline introduces an important optimization: it only rebuilds and redeploys the database container when the underlying schema has actually changed. This avoids unnecessary downtime and image churn for a component that changes far less frequently than the application.

### Stages

1. **Application Repository**
   Clones the application repo and runs `check_harbor_db.py` to compare the current commit hash for `database/database.sql` against the hash embedded in the latest DB image tag already in Harbor.

2. **DB Changes: true** _(conditional)_
   This entire nested block only executes if `check_harbor_db.py` returns `true` — meaning the database schema has changed since the last build.

   - **Database Docker Build** — Builds the DB image using `./db/Dockerfile` and pushes it to Harbor with the same dual-hash tagging scheme used by the app pipeline.
   - **Security Scanning** — Runs `harbor_scanner.py` against the new DB image.
   - **Deploy** — Performs a rolling image update on the `db-deployment` in Kubernetes.

3. **Cleanup**
   Same workspace wipe as the application pipeline.

---

## Harbor Security Scanning

**File:** `harbor_scanner.py`

This script acts as the bridge between Jenkins and Harbor's container scanning API (backed by [Trivy](https://trivy.dev/)). It is invoked as a pipeline stage for both the app and database images.

### How It Works

1. Queries Harbor's REST API (`/api/v2.0`) to retrieve the SHA256 digest of the most recently pushed image artifact.
2. POSTs a scan initiation request to Harbor. Returns a non-zero exit code and fails the build if the scan request is rejected.
3. Polls the scan status endpoint every 4 seconds (up to 5 attempts) until the scan completes.
4. Prints the vulnerability summary as formatted JSON to the build log.

### Usage

```bash
python harbor_scanner.py \
  -r <registry-host> \
  -p <harbor-project-name> \
  -i <image-name> \
  -c <username>:<password>
```

| Flag | Description |
|---|---|
| `-r` | Harbor registry hostname |
| `-p` | Harbor project name |
| `-i` | Image/repository name |
| `-c` | Credentials in `user:password` format |

### Example Output

```json
{
    "total": 12,
    "fixable": 8,
    "critical": 0,
    "high": 3,
    "medium": 7,
    "low": 2,
    "none": 0,
    "unknown": 0
}
```

---

## Harbor DB Change Detection

**File:** `check_harbor_db.py`

This script prevents redundant database rebuilds by comparing the current Git commit hash of the database schema file against the hash stored in the image tag of the latest DB artifact in Harbor.

### How It Works

1. Accepts the current commit hash of `database/database.sql` (extracted by the pipeline via `git log --follow`).
2. Queries Harbor to retrieve the tag of the latest existing DB image.
3. Parses the `<db-hash>-<pipeline-hash>` tag format to extract the stored DB hash.
4. Compares the two hashes:
   - If they differ → prints `true` (rebuild needed)
   - If they match → prints `false` (skip build)
   - If any error occurs (image doesn't exist yet, API failure) → prints `true` (default to rebuild)

### Usage

```bash
python check_harbor_db.py \
  -r <registry-host> \
  -p <harbor-project-name> \
  -i <image-name> \
  -c <username>:<password> \
  -h <database-commit-hash>
```

| Flag | Description |
|---|---|
| `-r` | Harbor registry hostname |
| `-p` | Harbor project name |
| `-i` | Image/repository name |
| `-c` | Credentials in `user:password` format |
| `-h` | Git commit hash of the database schema file |

---

## Dockerfiles

### Application (`app/Dockerfile`)

```dockerfile
FROM python:3.9
COPY / ./app/
WORKDIR app
RUN pip install --no-cache-dir \
    --index-url https://nexus.dev.afsmtddso.com/repository/jmedina-nexus-repo/simple \
    -r requirements.txt
ENTRYPOINT ["python"]
CMD ["server.py"]
```

- Based on the official `python:3.9` image.
- Pulls Python dependencies from a private **Nexus** repository rather than the public PyPI index — a common enterprise pattern for dependency governance and air-gap compatibility.
- Runs `server.py` as the application entrypoint.

### Database (`db/Dockerfile`)

```dockerfile
FROM postgres:9.6
COPY database/database.sql /docker-entrypoint-initdb.d/
```

- Based on the official `postgres:9.6` image.
- Seeds the database on first container startup by placing `database.sql` into PostgreSQL's `docker-entrypoint-initdb.d/` directory, which is automatically executed when the container initializes for the first time.

---

## Image Tagging Strategy

Both pipelines use a compound tag format:

```
<image-name>:<source-commit-hash>-<pipeline-commit-hash>
```

For example:
```
harbor.dev.afsmtddso.com/jmedina-harbor-project/app:a1b2c3d-e4f5g6h
```

This provides two layers of traceability:
- **Source commit hash** — links the image directly back to the application code that was built.
- **Pipeline commit hash** — identifies which version of the pipeline definition produced the build, useful for auditing and debugging CI changes over time.

For the database pipeline specifically, the source commit hash is scoped to the last commit that touched `database/database.sql`, not the overall repo head. This is what enables `check_harbor_db.py` to make accurate change detection decisions.

---

## Jenkins Configuration

### Required Credentials

| Credential ID | Type | Used For |
|---|---|---|
| `jmedina-harbor-auth` | Username + Password | Harbor registry authentication (push + API) |

### Required Agent Label

All pipeline stages run on agents labeled `jenkins-agent`.

### Environment Variables

| Variable | Value |
|---|---|
| `HARBOR_REGISTRY` | `harbor.dev.afsmtddso.com` |
| `HARBOR_PROJECT` | `jmedina-harbor-project` |
| `APP_IMAGE_NAME` | `app` |
| `DB_IMAGE_NAME` | `db` |

---

## DevOps Camp Training Context

This pipeline was part of a hands-on DevSecOps curriculum where engineers were guided through building each component incrementally:

| Module | Skills Covered |
|---|---|
| Containerization | Writing Dockerfiles, building and tagging images |
| Private Registries | Pushing to and pulling from Harbor, managing projects |
| CI/CD Automation | Writing Jenkins declarative pipelines, agents, stages |
| Artifact Management | Nexus as a private PyPI proxy |
| Security Scanning | Integrating Trivy via Harbor API into CI gates |
| Kubernetes Deployment | `kubectl set image`, rolling updates, config maps |
| Pipeline Scripting | Python automation scripts for registry interaction and conditional logic |

The separation of the application and database pipelines — and the smart change-detection logic in the DB pipeline — reflects a real-world concern that participants were expected to understand and solve as part of their training: **don't rebuild and redeploy things that haven't changed**.

---

## License

This project was developed as part of an internal government training program. All code is intended for educational use.
