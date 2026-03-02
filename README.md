# n8n Docker Compose - Production-Ready Queue Mode

Production-ready n8n v2.0+ self-hosted deployment with queue mode, horizontal scaling, and external task runners.

[![n8n version](https://img.shields.io/badge/n8n-2.0.3-blue)](https://n8n.io)
[![Docker](https://img.shields.io/badge/docker-compose-blue)](https://docs.docker.com/compose/)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

## Why This Setup?

- **Queue Mode** - Horizontal scaling with Redis-backed job queue
- **External Task Runners** - Isolated JavaScript/Python code execution
- **Multi-Environment** - Local, staging, production from one repo
- **One Command Setup** - Auto-generated secrets, ready in seconds
- **Custom npm Packages** - optional custom runner build with extra npm/Python packages
- **Production Hardened** - Resource limits, health checks, security defaults

## Quick Start

```bash
# Clone
git clone https://github.com/YOUR_USERNAME/n8n-docker-compose.git
cd n8n-docker-compose

# Local development (full queue mode)
./n8n init local
./n8n local up -d
# Open http://localhost:5678

# Production
./n8n init
nano .env  # Set N8N_HOST, WEBHOOK_URL
./n8n up -d
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              QUEUE MODE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────────────────────┐ │
│  │  PostgreSQL │      │    Redis    │      │         n8n-main            │ │
│  │   (data)    │◄────►│   (queue)   │◄────►│  (UI, API, webhooks)        │ │
│  └─────────────┘      └──────┬──────┘      │                             │ │
│                              │             │  ┌─────────────────────────┐ │ │
│                              │             │  │   n8n-main-runner       │ │ │
│                              │             │  │   (JS/Python sidecar)   │ │ │
│                              │             │  └─────────────────────────┘ │ │
│                              │             └─────────────────────────────┘ │
│                              │                                             │
│                              ▼                                             │
│                    ┌─────────────────────────────────────────────────────┐ │
│                    │              n8n-worker (scalable)                  │ │
│                    │              executes queued jobs                   │ │
│                    │                                                     │ │
│                    │  ┌─────────────────────────────────────────────────┐│ │
│                    │  │   n8n-worker-runner (JS/Python sidecar)         ││ │
│                    │  │   Code node execution isolated from worker      ││ │
│                    │  └─────────────────────────────────────────────────┘│ │
│                    └─────────────────────────────────────────────────────┘ │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**How it works:**
1. **n8n-main** receives triggers (webhooks, schedules, manual runs)
2. Jobs are queued to **Redis**
3. **n8n-worker** picks up and executes workflows
4. **Code nodes** run in isolated **task runner** sidecars (JS & Python)
5. Results stored in **PostgreSQL**

## Commands

```bash
# Initialize environment
./n8n init local              # Local (.env.local)
./n8n init staging            # Staging (.env.staging)
./n8n init                    # Production (.env)

# Start/Stop
./n8n local up -d             # Start local
./n8n staging up -d           # Start staging
./n8n up -d                   # Start production
./n8n <env> down              # Stop
./n8n <env> ps                # Status
./n8n <env> logs -f           # Follow logs

# Scale workers
./n8n up -d --scale n8n-worker=3

# Simple dev mode (no queue, single instance)
./n8n dev up -d
```

## Environment Isolation

Run multiple environments on the same host:

| Environment | Port | Config | Containers |
|-------------|------|--------|------------|
| Local | 5678 | `.env.local` | n8n-local-* |
| Staging | 5679 | `.env.staging` | n8n-staging-* |
| Production | 5678 | `.env` | n8n-prod-* |

```bash
./n8n init local && ./n8n init staging
./n8n local up -d             # http://localhost:5678
./n8n staging up -d           # http://localhost:5679
```

## Configuration

Secrets are auto-generated as files in `./secrets/` by `./n8n init`. Edit your `.env` file for production:

| Variable | Description |
|----------|-------------|
| `N8N_HOST` | Your domain (e.g., `n8n.example.com`) |
| `WEBHOOK_URL` | Public webhook URL (e.g., `https://n8n.example.com/`) |
| `N8N_PROTOCOL` | `http` or `https` |

**Optional tuning:**

| Variable | Default | Description |
|----------|---------|-------------|
| `WORKER_CONCURRENCY` | 10 | Parallel executions per worker |
| `N8N_MEMORY_LIMIT` | 2G | Memory limit for n8n containers |
| `RUNNER_MEMORY_LIMIT` | 1G | Memory limit for runner containers |
| `EXECUTIONS_DATA_MAX_AGE` | 336 | Hours to retain execution history |

## VPS / Cloud Deployment

```bash
# 1. Clone and initialize
git clone https://github.com/YOUR_USERNAME/n8n-docker-compose.git n8n
cd n8n && ./n8n init

# 2. Configure
nano .env
# N8N_HOST=n8n.yourdomain.com
# WEBHOOK_URL=https://n8n.yourdomain.com/
# N8N_PROTOCOL=https

# 3. Start with scaling
./n8n up -d --scale n8n-worker=2

# 4. Setup reverse proxy (Caddy/nginx)
```

**Caddy (automatic HTTPS):**
```
n8n.yourdomain.com {
    reverse_proxy localhost:5678
}
```

**nginx:**
```nginx
server {
    listen 443 ssl;
    server_name n8n.yourdomain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Custom npm Packages

By default the official `n8nio/runners` image is used. To add npm packages, enable the custom build:

**1. Uncomment the build block in `compose.yml`** (and comment out the plain `image:` line):

```yaml
x-runner-image: &runner-image
  build:
    context: .
    dockerfile: docker/Dockerfile.runners
    args:
      RUNNERS_VERSION: ${RUNNERS_VERSION:-latest}
  image: n8n-runners-custom:${N8N_VERSION:-2.0.3}
  # image: n8nio/runners:${RUNNERS_VERSION:-latest}   # comment out when building
```

**2. Edit `docker/Dockerfile.runners`** to add your packages:

```dockerfile
RUN cd /opt/runners/task-runner-javascript && \
    pnpm add xlsx pdf-lib uuid date-fns your-package
```

**3. Add to `n8n-task-runners.json` in `env-overrides`:**

```json
"NODE_FUNCTION_ALLOW_EXTERNAL": "xlsx,pdf-lib,uuid,date-fns,your-package"
```

**4. Rebuild:**

```bash
./n8n local down && ./n8n local up -d --build
```

**Usage in Code node:**

```javascript
const XLSX = require('xlsx');
const { v4: uuidv4 } = require('uuid');
const { format } = require('date-fns');

return { id: uuidv4(), date: format(new Date(), 'yyyy-MM-dd') };
```

**Python packages** — uncomment the Python block in `docker/Dockerfile.runners` and add packages to the `N8N_RUNNERS_EXTERNAL_ALLOW` entry in `n8n-task-runners.json`.

## Scaling

```bash
# Scale workers (each gets its own task runner sidecar)
./n8n up -d --scale n8n-worker=5

# Monitor
./n8n logs -f n8n-worker
```

**Guidelines:**
- 1 worker = 10 parallel executions (default `WORKER_CONCURRENCY`)
- 3 workers = 30 parallel workflow executions
- Monitor Redis queue depth for scaling decisions

## Files

```
├── compose.yml              # Queue mode (local/staging/production)
├── compose.dev.yml          # Simple dev mode (no queue)
├── docker/
│   └── Dockerfile.runners   # Custom runner with npm packages
├── secrets/                 # Docker secrets (auto-generated, git-ignored)
│   ├── postgres_password
│   ├── redis_password
│   ├── n8n_encryption_key
│   └── n8n_runners_auth_token
├── n8n-task-runners.json    # Runner config (module allowlist)
├── .env.example             # Configuration template
├── n8n                      # CLI wrapper script
└── README.md
```

## Security

This deployment follows the OWASP Docker Security Cheat Sheet and CIS Docker Benchmark.

### Docker Secrets

All sensitive values (database password, Redis password, encryption key, runner auth token) are stored as files in `./secrets/` and mounted into containers via Docker secrets at `/run/secrets/`. They are **never** exposed as environment variables — services use the `_FILE` suffix convention (e.g., `DB_POSTGRESDB_PASSWORD_FILE`).

```bash
./n8n init local   # Auto-generates secrets/postgres_password, secrets/redis_password, etc.
```

### Network Isolation

| Network | Type | Services |
|---------|------|----------|
| `frontend` | bridge (internet-facing) | n8n-main |
| `backend` | bridge, **internal** (no internet) | postgres, redis, n8n-main, n8n-worker |

Workers, database, and Redis have **no internet access**. Only n8n-main is reachable from the host (on `127.0.0.1` only).

### Container Hardening

Every container runs with:
- `read_only: true` — filesystem is read-only (writable paths via `tmpfs`)
- `cap_drop: ALL` — all Linux capabilities dropped (minimum added back per service)
- `no-new-privileges` — prevents privilege escalation
- `pids_limit: 256` — limits fork bombs
- Resource limits (CPU + memory) on all services

### Application Security

| Setting | Value | Purpose |
|---------|-------|---------|
| `N8N_BLOCK_ENV_ACCESS_IN_NODE` | `true` | Prevents Code nodes from reading env vars |
| `N8N_BLOCK_FILE_ACCESS_TO_N8N_FILES` | `true` | Blocks access to `.n8n` config files |
| `N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS` | `true` | Enforces 0600 on settings file |
| `N8N_RESTRICT_FILE_ACCESS_TO` | `/home/node/.n8n` | Restricts file system access |
| `N8N_COMMUNITY_PACKAGES_ENABLED` | `false` | Blocks untrusted community packages |
| `N8N_TEMPLATES_ENABLED` | `false` | Disables template gallery |
| `N8N_PUBLIC_API_DISABLED` | `true` | Disables REST API |
| `N8N_PUBLIC_API_SWAGGERUI_DISABLED` | `true` | Disables API playground |
| `N8N_GIT_NODE_DISABLE_BARE_REPOS` | `true` | Prevents Git node from bare repos |
| `NODE_FUNCTION_ALLOW_BUILTIN` | `crypto,path,url,util` | No `fs` module in Code nodes |
| `NODES_EXCLUDE` | executeCommand, localFileTrigger, readWriteFile | Blocks dangerous nodes |
| `N8N_SAMESITE_COOKIE` | `strict` | Strict cross-site cookie policy |

### Instance Isolation

All outbound connections to n8n's servers are disabled:
- `N8N_DIAGNOSTICS_ENABLED=false` + config endpoints cleared
- `N8N_VERSION_NOTIFICATIONS_ENABLED=false`
- `N8N_TEMPLATES_ENABLED=false`
- `EXTERNAL_FRONTEND_HOOKS_URLS` cleared

### Task Runner Hardening

Runners follow the [official hardening guide](https://docs.n8n.io/hosting/securing/hardening-task-runners/):
- Run as `nobody` user (UID 65532)
- Read-only filesystem with tmpfs at `/tmp`
- `N8N_RUNNERS_INSECURE_MODE=false`
- `N8N_BLOCK_RUNNER_ENV_ACCESS=true` (Python)
- External mode (isolated sidecar containers)

### Port Binding

Ports are bound to `127.0.0.1` only — not accessible from the network. Use a reverse proxy (Caddy/nginx) for external access with TLS.

### Backups

- **Never commit** `.env`, `.env.local`, `.env.staging`, or `secrets/` files
- **Backup `secrets/` directory** — losing `n8n_encryption_key` means losing all stored credentials
- **Same encryption key** across all instances in an environment
- Backup PostgreSQL data regularly: `docker compose exec postgres pg_dump -U n8n n8n > backup.sql`

## Troubleshooting

```bash
# All healthy?
./n8n local ps

# View logs
./n8n local logs -f
./n8n local logs -f n8n-main
./n8n local logs -f n8n-worker

# Verify queue mode
./n8n local logs | grep -E "(Enqueued|Worker started|Worker finished)"

# Verify task runners
./n8n local logs | grep "Registered runner"

# Restart
./n8n local down && ./n8n local up -d
```

## Requirements

- Docker Engine 24.0+
- Docker Compose v2.20+ (for `start_interval` healthcheck support)
- 2GB RAM minimum (4GB+ for production)

## AWS / Fargate

This setup can be adapted for AWS:
- Use RDS for PostgreSQL
- Use ElastiCache for Redis
- Deploy n8n-main and n8n-worker as separate ECS services
- Use ALB for load balancing and SSL termination
- Store secrets in AWS Secrets Manager

## License

MIT
