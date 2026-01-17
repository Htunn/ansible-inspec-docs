# Server Guide

The ansible-inspec server provides enterprise-ready features for compliance automation at scale.

## Overview

The server (introduced in v0.4.0) transforms ansible-inspec from a CLI tool into a full-featured compliance automation platform with:

- **REST API Server**: FastAPI-based async API for programmatic access
- **Flexible Storage**: File-based (no database) or PostgreSQL (enterprise features)
- **Enterprise Authentication**: Azure AD OAuth2 with RBAC (database mode)
- **VCS Integration**: Automatic Git sync for InSpec profiles (database mode)
- **Job Orchestration**: Background execution with templates and workflows
- **Monitoring**: Prometheus metrics and health checks

## Architecture Overview

```mermaid
graph TB
    subgraph Client Layer
        CLI[CLI Client]
        Browser[Web Browser]
        API_Client[API Client]
    end

    subgraph Server Layer
        API[FastAPI Server<br/>Port 8080]
        Auth[Azure AD Auth<br/>Optional]
        VCS[VCS Manager<br/>Optional]
    end

    subgraph Storage Layer
        FileStorage[File Storage<br/>/data directory]
        DBStorage[(PostgreSQL<br/>Prisma ORM)]
    end

    subgraph Execution Layer
        Queue[Job Queue]
        Workers[Job Workers]
        InSpec[InSpec CLI]
    end

    CLI --> API
    Browser --> API
    API_Client --> API

    API --> Auth
    API --> VCS
    API --> Queue

    Auth -.->|Database Mode| DBStorage
    VCS -.->|Database Mode| DBStorage
    
    API -->|File Mode| FileStorage
    API -->|Database Mode| DBStorage

    Queue --> Workers
    Workers --> InSpec
    Workers --> FileStorage
    Workers -.->|Database Mode| DBStorage

    InSpec --> Targets[Target Hosts]

    style FileStorage fill:#90EE90
    style DBStorage fill:#87CEEB
    style Auth fill:#FFE4B5
    style VCS fill:#FFE4B5
```

## Quick Start

### Simple Start (File Storage Only)

For quick testing or development without database setup:

```bash
# Start server with file-based storage
ansible-inspec start-server

# Access
# API: http://localhost:8080
# Docs: http://localhost:8080/docs
```

**Note**: This mode uses local `/data` directory for storage. Database-dependent features (authentication, VCS integration) will be disabled.

### Installation

```bash
# Install with server dependencies
pip install ansible-inspec

# Or from source
git clone https://github.com/Htunn/ansible-inspec.git
cd ansible-inspec
pip install -e ".[server]"
```

### Docker Deployment (Recommended for Production)

```bash
# 1. Clone repository
git clone https://github.com/Htunn/ansible-inspec.git
cd ansible-inspec

# 2. Configure environment
cp .env.docker .env
# Edit .env and set required variables:
# - POSTGRES_PASSWORD
# - ENCRYPTION_KEY (generate: python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
# - AUTH__JWT_SECRET (generate: python -c "import secrets; print(secrets.token_urlsafe(32))")

# 3. Start services
docker compose up -d

# 4. Initialize database
docker compose exec api prisma db push

# 5. Verify
curl http://localhost:8080/health
```

### Local Development

```bash
# 1. Start PostgreSQL
docker run -d --name ansible-inspec-postgres \
  -e POSTGRES_USER=ansible \
  -e POSTGRES_PASSWORD=ansible \
  -e POSTGRES_DB=ansible_inspec \
  -p 5432:5432 postgres:16-alpine

# 2. Configure .env
cp .env.example .env
# Edit DATABASE__URL and other settings

# 3. Initialize database
python scripts/init_prisma.py

# 4. Start server
ansible-inspec start-server

# 5. Access
# API: http://localhost:8080
# Docs: http://localhost:8080/docs
```

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                 FastAPI REST API Server                 │
│                    (Port 8080)                          │
│                                                         │
│  ┌────────────┐  ┌────────────┐  ┌──────────────┐     │
│  │ API Routes │  │   Auth     │  │ Job Executor │     │
│  │            │  │ (Azure AD) │  │              │     │
│  └────────────┘  └────────────┘  └──────────────┘     │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│            PostgreSQL Database (Prisma ORM)             │
│                                                         │
│  • Users & Authentication                              │
│  • Job Templates & Jobs                                │
│  • Workflow Templates                                  │
│  • VCS Credentials & Repositories                      │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────┐
│         ansible-inspec Core Engine                      │
│                                                         │
│  ┌──────────┐  ┌──────────┐  ┌────────────────┐       │
│  │  Runner  │  │ InSpec   │  │ Ansible        │       │
│  │          │  │ Adapter  │  │ Integration    │       │
│  └──────────┘  └──────────┘  └────────────────┘       │
└─────────────────────────────────────────────────────────┘
```

## Storage Modes

The server supports three storage backends:

### File Storage (Default)

Uses local filesystem for all data. **No database required.**

```bash
# .env or command line (default)
STORAGE_BACKEND=file

# Start server
ansible-inspec start-server --data-dir ./data
```

**Features:**
- ✅ Job templates and execution history
- ✅ Profile management
- ✅ JSON-based storage in `/data` directory
- ❌ No authentication/authorization
- ❌ No VCS integration
- ❌ No multi-user support

**Use cases:**
- Local development
- Single-user testing
- CI/CD pipelines
- Quick demos

### Database Storage

Full PostgreSQL backend with enterprise features.

```bash
# .env
STORAGE_BACKEND=database
DATABASE__URL=postgresql://user:pass@localhost:5432/dbname

# Start server (requires database setup)
ansible-inspec start-server
```

**Features:**
- ✅ All file storage features
- ✅ Azure AD authentication & RBAC
- ✅ VCS integration with Git
- ✅ Multi-user collaboration
- ✅ Audit logging
- ✅ Scalable performance

**Use cases:**
- Production deployments
- Team collaboration
- Enterprise environments

### Hybrid Storage

Dual-write for migration validation (advanced).

See [Database Setup Guide](database-setup.md) for migration details.

## Server Workflows

### Job Execution Flow

```mermaid
sequenceDiagram
    participant Client
    participant API as REST API
    participant Storage as Storage Backend
    participant Queue as Job Queue
    participant Worker as Job Worker
    participant InSpec
    participant Target as Target Hosts

    Client->>API: POST /job-templates/{id}/launch
    API->>Storage: Load template
    Storage-->>API: Template config
    API->>Queue: Queue job
    API-->>Client: 202 Accepted {job_id}

    Queue->>Worker: Pick up job
    Worker->>Storage: Update status: running
    Worker->>InSpec: Execute profile
    InSpec->>Target: Run compliance checks
    Target-->>InSpec: Check results
    InSpec-->>Worker: Aggregated results
    Worker->>Storage: Save results & status
    Worker->>Storage: Update status: completed

    Client->>API: GET /jobs/{job_id}
    API->>Storage: Fetch job
    Storage-->>API: Job data + results
    API-->>Client: 200 OK {status, results}
```

### Authentication Flow (Database Mode)

```mermaid
sequenceDiagram
    participant User
    participant Browser
    participant API as FastAPI Server
    participant AzureAD as Azure AD
    participant DB as PostgreSQL

    User->>Browser: Access protected endpoint
    Browser->>API: GET /api/v1/jobs
    API-->>Browser: 401 Unauthorized
    Browser->>API: GET /api/v1/auth/login
    API-->>Browser: Redirect to Azure AD
    Browser->>AzureAD: Login
    AzureAD-->>Browser: Auth code
    Browser->>API: GET /api/v1/auth/callback?code=...
    API->>AzureAD: Exchange code for token
    AzureAD-->>API: Access token + user info
    API->>DB: Create/update user
    DB-->>API: User record
    API->>API: Generate JWT
    API-->>Browser: Set JWT cookie
    Browser->>API: GET /api/v1/jobs (with JWT)
    API->>API: Validate JWT
    API->>DB: Fetch jobs
    DB-->>API: Job list
    API-->>Browser: 200 OK {jobs}
```

### VCS Sync Flow (Database Mode)

```mermaid
sequenceDiagram
    participant Scheduler
    participant VCS as VCS Manager
    participant Git as Git Repository
    participant DB as PostgreSQL
    participant Storage as File System

    Scheduler->>VCS: Poll repositories (every 15min)
    VCS->>DB: Fetch active repos
    DB-->>VCS: Repository configs
    
    loop For each repository
        VCS->>Git: git fetch origin
        Git-->>VCS: Latest commits
        VCS->>VCS: Check for changes
        
        alt New commits detected
            VCS->>Git: git pull
            Git-->>VCS: Updated files
            VCS->>Storage: Extract profiles
            VCS->>DB: Update sync status
            VCS->>DB: Create job templates
            DB-->>VCS: Templates created
        else No changes
            VCS->>DB: Update last_sync timestamp
        end
    end
```

### Storage Decision Flow

```mermaid
flowchart TD
    A[ansible-inspec start-server] --> B{DATABASE__URL set?}
    B -->|No| C[File Storage Mode]
    B -->|Yes| D{Can connect to DB?}
    
    D -->|Yes| E[Database Storage Mode]
    D -->|No| F[Log Error & Fallback]
    F --> C
    
    C --> G[Initialize File Backend]
    G --> H[Create /data directories]
    H --> I[Server Ready - Basic Features]
    
    E --> J[Initialize Prisma Client]
    J --> K[Run Migrations]
    K --> L[Create Default Admin User]
    L --> M{AUTH__ENABLED?}
    
    M -->|Yes| N[Initialize Azure AD]
    M -->|No| O{VCS__ENABLED?}
    
    N --> O
    O -->|Yes| P[Initialize VCS Manager]
    O -->|No| Q[Server Ready - All Features]
    P --> Q
    
    I --> R[Available: Jobs, Templates, Profiles]
    Q --> S[Available: Jobs, Templates, Profiles,<br/>Auth, VCS, Multi-user]
```

## Key Features

### 1. Job Templates

Reusable templates for compliance checks:

```bash
# Create a template via API
curl -X POST http://localhost:8080/api/v1/job-templates \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Linux Baseline Check",
    "description": "DevSec Linux Baseline compliance scan",
    "profilePath": "dev-sec/linux-baseline",
    "supermarket": true,
    "reporter": "cli json",
    "inventory": "/path/to/inventory.yml"
  }'

# Launch a job from template
curl -X POST http://localhost:8080/api/v1/job-templates/{id}/launch
```

### 2. Workflow Orchestration

Chain multiple compliance checks:

```json
{
  "name": "Full Security Audit",
  "nodes": [
    {
      "identifier": "baseline-check",
      "jobTemplateId": "template-1"
    },
    {
      "identifier": "remediation",
      "jobTemplateId": "template-2",
      "successNodes": ["verify-fix"]
    },
    {
      "identifier": "verify-fix",
      "jobTemplateId": "template-3"
    }
  ]
}
```

### 3. Azure AD Authentication

Enterprise SSO with role-based access control:

```bash
# In .env
AUTH__ENABLED=true
AUTH__CLIENT_ID=your-azure-app-id
AUTH__CLIENT_SECRET=your-azure-secret
AUTH__TENANT_ID=your-tenant-id
```

See [Authentication Guide](authentication.md) for setup details.

### 4. VCS Integration

Automatic sync of InSpec profiles from Git repositories:

```bash
# In .env
VCS__ENABLED=true
VCS__POLLING_INTERVAL=300  # 5 minutes
```

See [VCS Integration Guide](vcs-integration.md) for configuration.

### 5. PostgreSQL Database

Production-ready storage with:
- Connection pooling
- Type-safe operations (Prisma ORM)
- Hybrid mode for migration validation
- Automatic schema management

See [Database Setup Guide](database-setup.md) for details.

## API Endpoints

### Health & Info

```bash
# Health check
GET /health

# API information
GET /api/v1

# Prometheus metrics
GET /metrics
```

### Job Templates

```bash
# List templates
GET /api/v1/job-templates

# Create template
POST /api/v1/job-templates

# Get template
GET /api/v1/job-templates/{id}

# Update template
PUT /api/v1/job-templates/{id}

# Delete template
DELETE /api/v1/job-templates/{id}

# Launch job
POST /api/v1/job-templates/{id}/launch
```

### Jobs

```bash
# List jobs
GET /api/v1/jobs

# Get job
GET /api/v1/jobs/{id}

# Get job output
GET /api/v1/jobs/{id}/stdout

# Cancel job
POST /api/v1/jobs/{id}/cancel
```

### Workflows

```bash
# List workflows
GET /api/v1/workflow-templates

# Create workflow
POST /api/v1/workflow-templates

# Launch workflow
POST /api/v1/workflow-templates/{id}/launch
```

### VCS

```bash
# List credentials
GET /api/v1/vcs/credentials

# Add credential
POST /api/v1/vcs/credentials

# Sync repository
POST /api/v1/vcs/repositories/{id}/sync
```

## Configuration

### Environment Variables

```bash
# Server
HOST=0.0.0.0
PORT=8080
LOG_LEVEL=INFO

# Database
DATABASE__URL=postgresql://user:pass@localhost:5432/db

# Storage
STORAGE_BACKEND=database  # or 'hybrid' or 'file'
ENCRYPTION_KEY=<fernet-key>

# Authentication (Optional)
AUTH__ENABLED=true
AUTH__CLIENT_ID=<azure-app-id>
AUTH__CLIENT_SECRET=<azure-secret>
AUTH__TENANT_ID=<azure-tenant>
AUTH__JWT_SECRET=<secret>

# VCS (Optional)
VCS__ENABLED=true
VCS__POLLING_INTERVAL=300
```

### Storage Backends

- **database**: PostgreSQL only (production)
- **hybrid**: Dual-write to both file and database (migration)
- **file**: JSON files only (development)

## Monitoring

### Prometheus Metrics

Available at `/metrics`:

```bash
# Job metrics
ansible_inspec_jobs_total
ansible_inspec_jobs_successful
ansible_inspec_jobs_failed
ansible_inspec_job_duration_seconds

# Storage metrics
ansible_inspec_storage_operations_total
ansible_inspec_storage_operation_duration_seconds

# HTTP metrics
http_requests_total
http_request_duration_seconds
```

### Health Checks

```bash
# Basic health
curl http://localhost:8080/health

# Expected response
{
  "status": "healthy",
  "version": "0.4.0",
  "storage_backend": "database",
  "auth_enabled": true,
  "vcs_enabled": true
}
```

## Security Considerations

### Production Deployment

1. **Use HTTPS**: Deploy behind a reverse proxy (nginx, Traefik)
2. **Enable Authentication**: Set `AUTH__ENABLED=true`
3. **Secure Database**: Use strong passwords and limit network access
4. **Encrypt Credentials**: Always set `ENCRYPTION_KEY`
5. **Network Isolation**: Use firewall rules and VPCs
6. **Regular Updates**: Keep dependencies up to date

### Authentication

- Azure AD provides enterprise SSO
- JWT tokens for API access
- Role-based access control (admin, user, viewer)
- Session management with secure cookies

### Credential Encryption

All VCS credentials (SSH keys, tokens) are encrypted using Fernet symmetric encryption:

```python
from cryptography.fernet import Fernet

# Generate key
key = Fernet.generate_key()
print(key.decode())  # Add to .env as ENCRYPTION_KEY
```

## Troubleshooting

### Server won't start

```bash
# Check ports
lsof -i :8080

# Check database connection
psql postgresql://user:pass@localhost:5432/db

# View logs
docker compose logs api
```

### Database connection errors

```bash
# Verify PostgreSQL is running
docker ps | grep postgres

# Test connection
psql -h localhost -p 5432 -U ansible -d ansible_inspec

# Initialize schema
docker compose exec api prisma db push
```

### Authentication issues

```bash
# Disable auth for debugging
AUTH__ENABLED=false

# Check Azure AD configuration
# Verify CLIENT_ID, TENANT_ID, and redirect URIs
```

### Job execution failures

```bash
# Check job logs
curl http://localhost:8080/api/v1/jobs/{id}/stdout

# Verify ansible-inspec CLI works
ansible-inspec exec dev-sec/linux-baseline --supermarket

# Check inventory file permissions
ls -la /path/to/inventory.yml
```

## Best Practices

1. **Use Docker Compose**: Simplifies deployment and dependencies
2. **Enable Monitoring**: Set up Prometheus and Grafana
3. **Backup Database**: Regular PostgreSQL backups
4. **Use VCS Integration**: Auto-sync profiles instead of manual uploads
5. **Implement CI/CD**: Automate template creation and updates
6. **Monitor Metrics**: Track job success rates and durations
7. **Set Resource Limits**: Configure job timeouts and concurrency
8. **Use Workflows**: Chain related checks for complex scenarios

## Next Steps

- [Database Setup](database-setup.md) - Detailed database configuration
- [Authentication Guide](authentication.md) - Azure AD setup
- [VCS Integration](vcs-integration.md) - Git repository sync
- [API Documentation](../reference/api.md) - Complete API reference

## Support

- **Documentation**: https://github.com/Htunn/ansible-inspec/tree/main/docs
- **Issues**: https://github.com/Htunn/ansible-inspec/issues
- **Discussions**: https://github.com/Htunn/ansible-inspec/discussions
