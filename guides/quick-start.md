# Quick Start Guide - ansible-inspec v0.4.0

This guide gets you running with ansible-inspec in under 5 minutes.

## Prerequisites

- Python 3.8+
- Docker (optional, for database mode)

## Quick Start Options

### Option 1: Instant Start (No Database)

Perfect for testing and local development:

```bash
# Install
pip install ansible-inspec

# Start server immediately
ansible-inspec start-server
```

**Access:**
- API: http://localhost:8080
- Docs: http://localhost:8080/docs

**Features:**
- ✅ Job templates & execution
- ✅ Profile management
- ✅ File-based storage in `/data`
- ❌ No authentication
- ❌ No VCS integration

### Option 2: Production Setup (Docker Compose)

For full enterprise features with database:

#### Step 1: Clone Repository

```bash
git clone https://github.com/Htunn/ansible-inspec.git
cd ansible-inspec
```

#### Step 2: Configure Environment

```bash
# Copy example environment file
cp .env.docker .env

# Generate encryption key
python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"

# Update .env with generated key and set POSTGRES_PASSWORD
nano .env
```

**Minimum required configuration:**
```bash
POSTGRES_PASSWORD=your-secure-password-here
ENCRYPTION_KEY=your-generated-key-here
STORAGE_BACKEND=database
DATABASE__URL=postgresql://ansibleinspec:your-password@postgres:5432/ansibleinspec
```

#### Step 3: Start Services

```bash
# Start PostgreSQL and application
docker compose up -d

# Check status
docker compose ps

# View logs
docker compose logs -f app
```

### Step 4: Initialize Database

```bash
# Run migrations
docker compose exec app alembic upgrade head

# Verify tables
docker compose exec postgres psql -U ansibleinspec -d ansibleinspec -c '\dt'
```

### Step 5: Access Applications

- **API Server**: http://localhost:8080
- **API Docs**: http://localhost:8080/docs
- **Streamlit UI**: http://localhost:8081
- **Prometheus Metrics**: http://localhost:8080/metrics

## Enable Azure AD Authentication

### Step 1: Register Azure AD App

Follow guide: `ansible-inspec-docs/guides/authentication.md`

Key steps:
1. Create app registration in Azure Portal
2. Add redirect URI: `http://localhost:8081/auth/callback`
3. Grant API permissions: User.Read
4. Create client secret
5. Note tenant ID, client ID, and secret

### Step 2: Update Configuration

```bash
# .env
AUTH__ENABLED=true
AUTH__AZURE_TENANT_ID=your-tenant-id
AUTH__AZURE_CLIENT_ID=your-client-id
AUTH__AZURE_CLIENT_SECRET=your-client-secret
```

### Step 3: Restart Application

```bash
docker compose restart app
```

Now authentication is required to access UI and API.

## Configure VCS Sync

### Step 1: Create VCS Credential

```bash
# Generate encryption key (if not done)
python scripts/generate_encryption_key.py

# Create GitHub personal access token
# Go to GitHub Settings > Developer settings > Personal access tokens
# Create token with 'repo' scope
```

### Step 2: Add Credential via API

```bash
curl -X POST http://localhost:8080/api/v1/vcs/credentials/ \
  -H "Content-Type: application/json" \
  -d '{
    "name": "github-profiles",
    "vcs_type": "github",
    "repository_url": "https://github.com/your-org/profiles.git",
    "token": "ghp_YourGitHubToken"
  }'
```

### Step 3: Add Repository

```bash
curl -X POST http://localhost:8080/api/v1/vcs/repositories/ \
  -H "Content-Type: application/json" \
  -d '{
    "name": "compliance-profiles",
    "url": "https://github.com/your-org/profiles.git",
    "branch": "main",
    "credential_id": "credential-id-from-step-2",
    "sync_enabled": true,
    "poll_interval_minutes": 15,
    "auto_import": true
  }'
```

Profiles will automatically sync every 15 minutes.

## Manual Installation (without Docker)

### Step 1: Install Dependencies

```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install ansible-inspec with all features
pip install -e ".[server,enterprise]"
```

### Step 2: Install PostgreSQL

```bash
# macOS
brew install postgresql@15
brew services start postgresql@15

# Create database
createdb ansibleinspec
```

### Step 3: Configure Environment

```bash
cp .env.example .env
# Edit .env with your configuration
```

### Step 4: Initialize Database

```bash
# Initialize Alembic
python scripts/init_alembic.py

# Run migrations
alembic upgrade head
```

### Step 5: Start Services

```bash
# Terminal 1: API Server
uvicorn ansible_inspec.server.api:app --host 0.0.0.0 --port 8080

# Terminal 2: Streamlit UI
streamlit run lib/ansible_inspec/server/ui.py --server.port 8081
```

## Usage Examples

### Create Job Template

```bash
curl -X POST http://localhost:8080/api/v1/job_templates/ \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Linux Security Baseline",
    "description": "CIS Linux Benchmark",
    "profile_path": "dev-sec/linux-baseline",
    "supermarket": true,
    "target": "linux_servers",
    "reporter": "cli json html"
  }'
```

### Launch Job

```bash
curl -X POST http://localhost:8080/api/v1/job_templates/{template_id}/launch/ \
  -H "Content-Type: application/json"
```

### View Job Status

```bash
curl http://localhost:8080/api/v1/jobs/{job_id}/
```

## Monitoring

### Prometheus Metrics

```bash
curl http://localhost:8080/metrics | grep storage
curl http://localhost:8080/metrics | grep vcs_sync
curl http://localhost:8080/metrics | grep jobs
```

### Check Hybrid Storage Validation

```bash
curl http://localhost:8080/api/v1/storage/validation-status
```

## Troubleshooting

### Container Won't Start

```bash
# Check logs
docker compose logs app

# Check PostgreSQL connection
docker compose exec app psql -U ansibleinspec -h postgres -d ansibleinspec
```

### Authentication Errors

```bash
# Verify environment variables
docker compose exec app env | grep AUTH

# Check Azure AD app registration
# Verify redirect URI matches exactly
```

### VCS Sync Failures

```bash
# Check logs
docker compose logs app | grep VCS

# Test credential
curl http://localhost:8080/api/v1/vcs/credentials/{credential_id}/test
```

## Next Steps

- **Read guides**: `ansible-inspec-docs/guides/`
- **Configure profiles**: Add your InSpec profiles to Git
- **Setup webhooks**: For immediate sync on push events
- **Enable auto-cutover**: After 30-day validation period
- **Add users**: Configure Azure AD roles for team members

## Support

- **Documentation**: `ansible-inspec-docs/`
- **Issues**: https://github.com/Htunn/ansible-inspec/issues
- **Discussions**: https://github.com/Htunn/ansible-inspec/discussions

## Version History

- **v0.4.0** (2026-01): Enterprise features (Auth, Database, VCS)
- **v0.3.0** (2025-12): Web UI and REST API
- **v0.2.0** (2025-06): Native Ansible translation
- **v0.1.0** (2024-12): Initial release
