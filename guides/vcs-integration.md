# VCS Integration Guide

This guide covers Git repository synchronization for InSpec profiles and job templates.

## Overview

ansible-inspec can automatically sync InSpec profiles from Git repositories, supporting:
- Periodic polling (configurable interval)
- Manual sync triggers
- Webhook integration (GitHub/GitLab)
- Encrypted credential storage
- Auto-import profiles as job templates

## Setup VCS Credentials

### Generate Encryption Key

```bash
python scripts/generate_encryption_key.py
```

Add to `.env`:
```bash
ENCRYPTION_KEY=your-fernet-key-here
```

### Create VCS Credential

#### Option 1: Using API

**SSH Key Authentication:**
```bash
curl -X POST http://localhost:8080/api/v1/vcs/credentials/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "github-deploy-key",
    "vcs_type": "github",
    "repository_url": "git@github.com:org/repo.git",
    "ssh_private_key": "-----BEGIN OPENSSH PRIVATE KEY-----\n...\n-----END OPENSSH PRIVATE KEY-----"
  }'
```

**Token Authentication:**
```bash
curl -X POST http://localhost:8080/api/v1/vcs/credentials/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "github-pat",
    "vcs_type": "github",
    "repository_url": "https://github.com/org/repo.git",
    "token": "ghp_your_personal_access_token"
  }'
```

#### Option 2: Using UI

1. Navigate to **VCS Configuration** page
2. Click **Add Credential**
3. Fill in details:
   - Name: `github-profiles`
   - Type: `GitHub`
   - Repository URL: `https://github.com/org/profiles.git`
   - Auth Method: SSH Key or Token
4. Click **Save** (credentials are encrypted automatically)

### GitHub Personal Access Token (PAT)

Create a PAT with `repo` scope:
1. Go to GitHub Settings > Developer settings > Personal access tokens
2. Click **Generate new token** (classic)
3. Select scopes: `repo` (full repository access)
4. Generate and copy token

### SSH Deploy Key

Generate SSH key pair:
```bash
ssh-keygen -t ed25519 -C "ansible-inspec@yourorg.com" -f ~/.ssh/ansibleinspec_deploy
```

Add public key to repository:
1. Go to Repository Settings > Deploy keys
2. Click **Add deploy key**
3. Paste public key content
4. Enable "Allow write access" if needed

Use private key in VCS credential configuration.

## Configure Repository Sync

### Enable VCS Sync

```bash
# .env
VCS__ENABLED=true
VCS__POLL_INTERVAL_MINUTES=15
VCS__CONFIG_DIR=./data/vcs_repos
```

### Add Repository

```bash
curl -X POST http://localhost:8080/api/v1/vcs/repositories/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "name": "compliance-profiles",
    "url": "https://github.com/org/compliance-profiles.git",
    "branch": "main",
    "credential_id": "credential-uuid-here",
    "sync_enabled": true,
    "poll_interval_minutes": 15,
    "auto_import": true
  }'
```

### Repository Structure

Organize your Git repository:

```
compliance-profiles/
├── linux-baseline/
│   ├── inspec.yml
│   ├── controls/
│   │   ├── ssh.rb
│   │   └── firewall.rb
│   └── README.md
├── windows-baseline/
│   ├── inspec.yml
│   ├── controls/
│   │   ├── security_policy.rb
│   │   └── audit.rb
│   └── README.md
└── README.md
```

The system automatically detects directories containing `inspec.yml`.

## Polling Configuration

### Automatic Polling

Polling jobs are scheduled automatically when repository is created with `sync_enabled: true`.

Default interval: 15 minutes (configurable per repository)

### Manual Sync Trigger

```bash
curl -X POST http://localhost:8080/api/v1/vcs/sync/ \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"repository_name": "compliance-profiles"}'
```

### View Sync History

```bash
curl http://localhost:8080/api/v1/vcs/repositories/compliance-profiles/history \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
{
  "repository": "compliance-profiles",
  "last_sync": "2026-01-16T10:30:00Z",
  "last_commit": "abc123def456",
  "sync_status": "success",
  "profiles_imported": 12,
  "history": [
    {
      "timestamp": "2026-01-16T10:30:00Z",
      "commit": "abc123def456",
      "status": "success",
      "changes": 3
    }
  ]
}
```

## Webhook Integration

### Enable Webhooks

```bash
# .env
VCS__WEBHOOK_ENABLED=true
VCS__WEBHOOK_SECRET=your-random-secret-here
```

Generate webhook secret:
```bash
python -c "import secrets; print(secrets.token_hex(32))"
```

### GitHub Webhook Setup

1. Go to Repository Settings > Webhooks
2. Click **Add webhook**
3. Configure:
   - **Payload URL**: `https://your-domain.com/api/v1/webhooks/github/`
   - **Content type**: `application/json`
   - **Secret**: Your webhook secret from `.env`
   - **Events**: Select "Just the push event"
4. Click **Add webhook**

### GitLab Webhook Setup

1. Go to Project Settings > Webhooks
2. Configure:
   - **URL**: `https://your-domain.com/api/v1/webhooks/gitlab/`
   - **Secret token**: Your webhook secret
   - **Trigger**: Push events
3. Click **Add webhook**

### Test Webhook

```bash
# Simulate GitHub webhook
curl -X POST http://localhost:8080/api/v1/webhooks/github/ \
  -H "Content-Type: application/json" \
  -H "X-Hub-Signature-256: sha256=..." \
  -d '{
    "ref": "refs/heads/main",
    "repository": {
      "clone_url": "https://github.com/org/profiles.git"
    }
  }'
```

## Profile Auto-Import

When `auto_import: true`, the system automatically:

1. Scans repository for InSpec profiles (`inspec.yml`)
2. Creates/updates job templates for each profile
3. Sets profile path to synced repository location
4. Preserves manual customizations to job templates

### Job Template Naming

Auto-generated templates use format:
```
[Repository]_[Profile]
```

Example: `compliance-profiles_linux-baseline`

### Customize Auto-Import Behavior

Create `.ansible-inspec.yml` in repository root:

```yaml
import:
  enabled: true
  prefix: "CIS"  # Prefix for job template names
  default_reporter: "cli json html"
  default_target: "all"
  tags:
    - compliance
    - automated
  extra_vars:
    scan_level: "level1"
```

## Monitoring VCS Sync

### Prometheus Metrics

```bash
curl http://localhost:8080/metrics | grep vcs_sync
```

Metrics:
- `vcs_sync_total{repository, status}`: Total sync operations
- `vcs_sync_duration_seconds`: Sync duration histogram

### View Scheduler Jobs

```bash
curl http://localhost:8080/api/v1/vcs/jobs/ \
  -H "Authorization: Bearer $TOKEN"
```

Response:
```json
[
  {
    "id": "poll_compliance-profiles",
    "name": "VCS Poll: https://github.com/org/profiles.git",
    "next_run": "2026-01-16T11:00:00Z",
    "trigger": "interval[0:15:00]"
  }
]
```

## Advanced Configuration

### Multiple Repositories

```bash
# .env
VCS__REPOSITORIES=[
  {"name":"linux-profiles","url":"https://github.com/org/linux.git","branch":"main"},
  {"name":"windows-profiles","url":"https://github.com/org/windows.git","branch":"production"}
]
```

### Custom Poll Intervals

Different repositories can have different intervals:

```python
# Repository 1: Check every 5 minutes (development)
{
  "name": "dev-profiles",
  "poll_interval_minutes": 5
}

# Repository 2: Check hourly (production)
{
  "name": "prod-profiles",
  "poll_interval_minutes": 60
}
```

### Branch Strategies

**Development + Production**:
```python
[
  {"name": "profiles-dev", "branch": "develop", "poll_interval_minutes": 5},
  {"name": "profiles-prod", "branch": "main", "poll_interval_minutes": 60}
]
```

## Troubleshooting

### Sync Failures

Check logs:
```bash
docker compose logs app | grep VCS
```

Common issues:
- **Authentication failure**: Verify credentials are not expired
- **Network error**: Check firewall rules, DNS resolution
- **SSH key permission**: Ensure private key has correct permissions (0600)

### Profile Not Imported

Verify profile structure:
```bash
# Clone repository locally
git clone $REPO_URL
cd $REPO_NAME

# Check for inspec.yml
find . -name "inspec.yml"

# Validate InSpec profile
inspec check ./profile-name/
```

### Webhook Not Triggering

1. Check webhook delivery in GitHub/GitLab (Settings > Webhooks > Recent Deliveries)
2. Verify payload URL is accessible from internet
3. Check webhook secret matches `.env` configuration
4. Review server logs for signature validation errors

### Slow Sync Performance

- Reduce poll frequency for large repositories
- Use shallow clone (configure in VCS settings)
- Enable delta sync (only check for changes, not full clone)

## Security Best Practices

1. **Use read-only deploy keys** when possible
2. **Rotate credentials regularly** (every 90 days)
3. **Use dedicated service accounts** for Git access
4. **Restrict repository access** to necessary profiles only
5. **Audit sync logs** for unauthorized access
6. **Validate webhook signatures** before processing
7. **Use HTTPS** for all repository URLs
8. **Store credentials encrypted** (automatic with Fernet)

## Migration from Manual Updates

### Step 1: Create Git Repository

```bash
mkdir compliance-profiles
cd compliance-profiles
git init

# Move existing profiles
mv /path/to/profiles/* .
git add .
git commit -m "Initial import of InSpec profiles"

# Push to remote
git remote add origin https://github.com/org/profiles.git
git push -u origin main
```

### Step 2: Configure VCS Sync

Follow credential and repository setup above.

### Step 3: Initial Sync

```bash
# Trigger manual sync
curl -X POST http://localhost:8080/api/v1/vcs/sync/ \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"repository_name": "compliance-profiles"}'
```

### Step 4: Verify Import

```bash
# List job templates
curl http://localhost:8080/api/v1/job_templates/ \
  -H "Authorization: Bearer $TOKEN" | jq '.[] | .name'
```

## References

- [Git Documentation](https://git-scm.com/doc)
- [GitHub Deploy Keys](https://docs.github.com/en/developers/overview/managing-deploy-keys)
- [GitLab Deploy Tokens](https://docs.gitlab.com/ee/user/project/deploy_tokens/)
- [GitHub Webhooks](https://docs.github.com/en/developers/webhooks-and-events/webhooks)
