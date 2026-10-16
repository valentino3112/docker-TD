# docker-TD — DevOps project

A small three-tier web application (PostgreSQL database, Spring Boot API, Apache reverse proxy), containerized with Docker, tested and published by GitHub Actions, and deployed to a remote server with Ansible.

Live: http://valentin.goncalves.takima.school

---

## Architecture

```
Browser ──► :80 httpd (reverse proxy) ──► simple-api:8080 (Spring Boot) ──► database:5432 (PostgreSQL)
            └───────────────────────── app-network (Docker) ─────────────────────────┘
```

- **httpd** is the only container exposed to the outside (port 80). It forwards every request to the API.
- **simple-api** is the backend, reached by the proxy through its container name.
- **database** is not exposed at all: only containers on `app-network` can reach it.

Containers find each other **by name** on the Docker network, so container names must match the configuration files (`httpd.conf`, `application.yml`).

---

## Repository structure

```
.
├── .github/workflows/main.yml   # CI/CD pipeline
├── ansible/                     # Deployment to the remote server
│   ├── inventories/setup.yml    # Target host, connection settings, variables
│   ├── playbook.yml             # Calls the roles in order
│   └── roles/
│       ├── docker/              # Installs Docker + Docker SDK for Python
│       ├── network/             # Creates the app-network Docker network
│       ├── database/            # Runs the PostgreSQL container
│       ├── app/                 # Runs the Spring Boot container
│       └── proxy/               # Runs the httpd container (+ restart handler)
├── backend-api/
│   ├── hello-world/             # First Java/Docker exercise
│   └── simpleapi/               # Spring Boot API (Maven project)
├── database/                    # PostgreSQL image + init SQL scripts
├── http-server/                 # Apache image + httpd.conf (reverse proxy)
├── flask-app/                   # Python/Flask exercise from the Docker TD
└── docker-compose.yml           # Runs the whole stack locally
```

---

## Running locally

Create a `.env` file at the root (it is read by `docker-compose.yml`):

```env
POSTGRES_DB=db
POSTGRES_USER=usr
POSTGRES_PASSWORD=pwd
```

Then:

```bash
docker compose up -d --build
curl http://localhost/departments
```

API endpoints include `/departments` and `/students`.

---

## CI/CD pipeline

Defined in `.github/workflows/main.yml`, triggered on pushes to `main` and `develop` and on pull requests. Three jobs run in sequence; each one only starts if the previous one succeeded (`needs:`).

| Job | What it does | When |
|---|---|---|
| `test-backend` | Builds the API and runs unit + integration tests with `mvn clean verify` (JDK 21) | Every run |
| `build-and-push-docker-image` | Builds the three images; pushes them to Docker Hub | Builds every run, pushes only on `main` |
| `deploy` | Runs the Ansible playbook against the server | Only on push to `main` |

Images published on Docker Hub:

- `valentino3112/tp-database:latest`
- `valentino3112/tp-backend:latest`
- `valentino3112/tp-httpd:latest`

### Required secrets

Set in **Settings → Secrets and variables → Actions → Repository secrets**:

| Secret | Content |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (Read & Write), not the account password |
| `SSH_PRIVATE_KEY` | Private key used to SSH into the server, including the BEGIN/END lines |

---

## Deployment with Ansible

### Inventory

`ansible/inventories/setup.yml` declares the server in a `prod` group and holds the connection settings (`ansible_user: admin`, path to the SSH key) and the application variables (Docker network name, database credentials). In CI, the key path is overridden with `-e ansible_ssh_private_key_file=...` since the runner has its own copy of the key.

### Roles

| Role | Purpose |
|---|---|
| `docker` | Installs prerequisites, adds Docker's APT repository (`deb822_repository`), installs `docker-ce`, creates a Python virtualenv in `/opt/docker_venv` with the Docker SDK, ensures the service is running |
| `network` | Creates the `app-network` Docker network |
| `database` | Starts the `database` container with `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` |
| `app` | Starts the `simple-api` container; overrides the datasource from `application.yml` with `SPRING_DATASOURCE_URL/USERNAME/PASSWORD` env variables; notifies the proxy restart handler when the container changes |
| `proxy` | Starts the `httpd` container and publishes port 80; its handler restarts httpd so it resolves the new backend address after a redeploy |

Tasks using the `community.docker` modules set `ansible_python_interpreter: /opt/docker_venv/bin/python3`, because the Docker SDK lives in that virtualenv rather than in the system Python.

### Running it manually

```bash
cd ansible
ansible-galaxy collection install community.docker
ansible-playbook -i inventories/setup.yml playbook.yml --syntax-check
ansible-playbook -i inventories/setup.yml playbook.yml
```

### Useful ad hoc commands

```bash
ansible all -i inventories/setup.yml -m ping
ansible all -i inventories/setup.yml -m setup -a "filter=ansible_distribution*"
ansible all -i inventories/setup.yml -m command -a "docker ps -a" --become
ansible all -i inventories/setup.yml -m command -a "docker logs --tail 30 simple-api" --become
```

Structure of an ad hoc command: `ansible <hosts> -i <inventory> -m <module> -a "<arguments>" [--become]`.

---
