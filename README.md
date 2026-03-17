# ClinomX — OpenMRS Distribution

ClinomX is an OpenMRS distribution that adds FHIR-based questionnaire and questionnaire response management to the OpenMRS platform. It is composed of a backend OpenMRS module (OMOD) and a set of frontend microfrontends built on the OpenMRS 3 (O3) SPA framework.

---

## Repository structure

```
.
├── backend/                  # OpenMRS module (Maven multi-module project)
│   ├── api/                  # Core business logic, services, DAO, and domain model
│   ├── omod/                 # Web layer — REST resources and module packaging
│   ├── modules/              # Pre-built OMODs mounted into the runtime container
│   └── Dockerfile            # Image that bakes all OMODs into an OpenMRS instance
│
├── frontend/                 # OpenMRS O3 microfrontend (Yarn monorepo)
│   ├── packages/
│   │   ├── esm-clinomix-app/ # ClinomX microfrontend (questionnaire UI)
│   │   └── esm-home-app/     # OpenMRS home page (required host for ClinomX extensions)
│   ├── Dockerfile            # Multi-stage build — bundles the SPA and serves it via nginx
│   └── nginx.conf            # nginx config — serves the SPA and proxies /openmrs/ to backend
│
├── docker-compose.yml        # Backend only (DB + build + runtime)
└── docker-compose.frontend.yml  # Full stack (DB + backend + frontend)
```

---

## Backend

### What it contains

The backend is a standard OpenMRS module structured as a Maven multi-module project:

| Module | Purpose |
|--------|---------|
| `api` | Domain model (`QuestionnaireRecord`, `QuestionnaireResponseRecord`), service interfaces and implementations, DAO (native SQL over OpenMRS Hibernate session), FHIR R4 parse/serialize via HAPI FHIR |
| `omod` | REST resources (`/ws/rest/v1/clinomx/questionnaire`, `/ws/rest/v1/clinomx/questionnaireresponse`), module packaging |

FHIR resources are stored as JSON blobs in two custom tables (`clinom_x_questionnaire`, `clinom_x_questionnaire_response`) created by Liquibase changesets. HAPI FHIR is used as a library only — there is no external FHIR server.

### Pre-built modules

`backend/modules/` contains the OMODs that are mounted directly into the OpenMRS container at runtime:

| File | Purpose |
|------|---------|
| `clinom-x-omod-*.omod` | The ClinomX module (built by Maven) |
| `webservices.rest-*.omod` | OpenMRS REST API |
| `fhir2-*.omod` | FHIR2 module (provides the HAPI FHIR libraries at runtime) |
| `legacyui-*.omod` | Legacy UI (required by OpenMRS core) |

After running `mvn package`, the ClinomX OMOD is automatically copied to this folder by the Maven build.

### Building manually

Requires Java 17+ and Maven 3.9+.

```bash
cd backend
mvn package -pl omod -am -DskipTests
```

The OMOD is written to `backend/modules/clinom-x-omod-*.omod`.

---

## Frontend

### What it contains

The frontend is a Yarn monorepo containing OpenMRS O3 microfrontends. Only two packages are built and served:

| Package | Purpose |
|---------|---------|
| `esm-clinomix-app` | ClinomX questionnaire UI — registers a dashboard and dashboard link into the OpenMRS home page |
| `esm-home-app` | OpenMRS home page — provides the `homepage-dashboard-slot` that ClinomX plugs into |

The build produces a static SPA served by nginx. The OpenMRS app shell (`@openmrs/esm-app-shell`) is used pre-built from npm — no `openmrs build` step is needed.

### Building manually

Requires Node.js 20+ and Yarn 4.

```bash
cd frontend
yarn workspaces focus @openmrs/esm-clinomix-app @openmrs/esm-home-app
yarn workspace @openmrs/esm-clinomix-app build
yarn workspace @openmrs/esm-home-app build
```

---

## Running with Docker Compose

### Prerequisites

- Docker Desktop with at least 6 GB of memory allocated (frontend build is memory-intensive)
- Docker Compose v2

### Backend only

Starts the database, compiles the Maven project inside a container, and launches OpenMRS.

```bash
docker compose up
```

| Service | Description | Port |
|---------|-------------|------|
| `backend-build` | Runs `mvn package`, copies the OMOD to `backend/modules/`, then exits | — |
| `backend` | OpenMRS 2.7.4 with the ClinomX OMOD | `8080` |
| `db` | MariaDB 10.11 | — |

OpenMRS is available at: `http://localhost:8080/openmrs`

Default credentials: `admin` / `Admin123`

> The `backend-build` service uses a persistent `maven-cache` Docker volume so Maven dependencies are not re-downloaded on every run.

### Full stack (backend + frontend)

Builds the frontend Docker image and adds nginx to serve the SPA.

```bash
docker compose -f docker-compose.frontend.yml up
```

| Service | Description | Port |
|---------|-------------|------|
| `backend-build` | Maven build (exits after completion) | — |
| `backend` | OpenMRS 2.7.4 | `8080` |
| `db` | MariaDB 10.11 | — |
| `frontend` | nginx — serves the SPA and proxies `/openmrs/` to the backend | `80` |

The application is available at: `http://localhost/openmrs/spa/home`

> The frontend Docker image must be rebuilt whenever the frontend source changes:
> ```bash
> docker compose -f docker-compose.frontend.yml build frontend
> docker compose -f docker-compose.frontend.yml up
> ```

### Updating the backend module without rebuilding the image

Because `backend/modules/` is bind-mounted into the container, you can update the running ClinomX module without rebuilding the Docker image:

```bash
# 1. Rebuild the OMOD (copies it to backend/modules/ automatically)
cd backend && mvn package -pl omod -am -DskipTests

# 2. Restart only the backend container
docker compose restart backend
```

### Optional: HAPI FHIR server

A standalone HAPI FHIR R4 server is available for development or testing purposes. It is not required by ClinomX (which stores FHIR resources internally in OpenMRS).

```bash
docker compose -f docker-compose.hapi.yml up
```

| Service | Description | Port |
|---------|-------------|------|
| `hapi-fhir` | HAPI FHIR R4 server | `8082` |
| `hapi-postgres` | PostgreSQL database for HAPI | — |

HAPI FHIR is available at: `http://localhost:8082/fhir`
