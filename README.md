# ansible-inspec Documentation

Welcome to the **ansible-inspec** documentation! This tool bridges infrastructure automation and compliance testing by combining Ansible's inventory management with InSpec's compliance framework.

## What is ansible-inspec?

ansible-inspec is an enterprise-ready compliance automation platform that transforms how you approach infrastructure compliance. It provides both a powerful CLI for local testing and a production-grade server with REST API for automated compliance workflows at scale.

### The Problem It Solves

Traditional compliance testing involves:
- SSH-ing into servers manually or writing custom scripts
- Maintaining separate inventories for compliance tools
- Manually compiling results into spreadsheets
- Spending hours generating reports for security teams
- Repeating this process monthly for audits

With ansible-inspec, you:
- ✅ Use your existing Ansible inventory for compliance testing
- ✅ Run InSpec profiles without learning new tools
- ✅ Generate audit-ready reports automatically
- ✅ Convert InSpec profiles to pure Ansible (no InSpec needed!)
- ✅ Access 100+ pre-built profiles from Chef Supermarket
- ✅ Run compliance checks in parallel across your entire fleet
- ✅ **NEW v0.4.0**: Enterprise REST API with job templates and workflows
- ✅ **NEW v0.4.0**: Azure AD authentication and PostgreSQL database
- ✅ **NEW v0.4.0**: VCS integration with automatic Git sync

## Key Features

### 🌐 Server Features (v0.4.0)

**Enterprise Compliance Platform**
- **REST API Server**: FastAPI-based async API for automation
- **Job Templates**: Reusable compliance check configurations
- **Workflow Orchestration**: Chain multiple checks together
- **Azure AD Authentication**: Enterprise SSO with RBAC
- **PostgreSQL Database**: Scalable storage with Prisma ORM
- **VCS Integration**: Auto-sync InSpec profiles from Git
- **Monitoring**: Prometheus metrics and health checks

See [Server Guide](guides/server.md) for complete server documentation.

### 🚀 CLI Features

**1. Native InSpec Execution**
Run existing InSpec profiles using your Ansible inventory.

```bash
ansible-inspec exec profile/ -i inventory.yml --reporter html
```

**2. Profile Conversion (InSpec-Free Mode)**
Convert Ruby-based InSpec profiles to pure Ansible collections.

```bash
ansible-inspec convert cis-benchmark/ --namespace security
```

**3. Chef Supermarket Integration**
Access 100+ pre-built compliance profiles instantly.

```bash
ansible-inspec exec dev-sec/linux-baseline --supermarket -i inventory.yml
```

### 📊 Multi-Format Reporting

Generate compliance reports in multiple formats:
- **JSON** - InSpec schema v5.22.0 compatible (works with Chef Automate)
- **HTML** - Interactive dashboards with pass/fail statistics
- **JUnit** - CI/CD integration for automated testing
- **CLI** - Real-time output during execution

### 🐳 Flexible Deployment

Available in multiple formats:
- **PyPI** - `pip install ansible-inspec`
- **Docker** - Pre-built images with PostgreSQL
- **Docker Compose** - Full stack deployment (API + Database)
- **Source** - Install from GitHub for latest features

### 🔄 InSpec-Free Operation

Converted collections run with:
- Zero InSpec dependency
- Pure Ansible native modules
- Automatic report generation via callback plugin
- Ready-to-distribute .tar.gz files

## Quick Start

### Server Deployment (Recommended for Production)

**Docker Compose:**
```bash
git clone https://github.com/Htunn/ansible-inspec.git
cd ansible-inspec
cp .env.docker .env
# Edit .env with your settings
docker-compose up -d
docker-compose exec api prisma db push
curl http://localhost:8080/health
```

See [Server Guide](guides/server.md), [Database Setup](guides/database-setup.md), and [Authentication](guides/authentication.md).

### CLI Installation

**Option 1: PyPI (Recommended for CLI)**
```bash
pip install ansible-inspec
ansible-inspec --version
```

**Option 2: Docker**
```bash
docker pull htunnthuthu/ansible-inspec:latest
docker run --rm htunnthuthu/ansible-inspec:latest --help
```

**Option 3: From Source**
```bash
git clone https://github.com/htunn/ansible-inspec.git
cd ansible-inspec
pip install -e .
```

### Your First Compliance Check

**Step 1: Create an inventory file**
```yaml
# inventory.yml
all:
  hosts:
    webserver:
      ansible_host: 192.168.1.10
    database:
      ansible_host: 192.168.1.20
```

**Step 2: Run a compliance check**
```bash
# Using Chef Supermarket profiles
ansible-inspec exec dev-sec/linux-baseline --supermarket \
  -i inventory.yml \
  --reporter html --output report.html

# Using local InSpec profile
ansible-inspec exec ./my-profile/ -i inventory.yml \
  --reporter json --output compliance.json
```

**Step 3: Convert to InSpec-free mode**
```bash
# Convert profile to Ansible collection
ansible-inspec convert dev-sec/linux-baseline \
  --namespace devsec \
  --collection-name linux_baseline

# Build and install
cd collections/ansible_collections/devsec/linux_baseline
ansible-galaxy collection build
ansible-galaxy collection install devsec-linux_baseline-*.tar.gz

# Run without InSpec!
ansible-playbook devsec.linux_baseline.compliance_check -i inventory.yml
```

## Use Cases

### Monthly Compliance Reporting
```bash
ansible-inspec exec pci-dss-baseline/ -i pci-servers.yml \
  --reporter "html:pci-compliance-$(date +%Y%m).html"
```

### CI/CD Integration
```yaml
# .gitlab-ci.yml
compliance_check:
  stage: test
  script:
    - ansible-inspec exec security-baseline/ \
        -i staging-inventory.yml \
        --reporter junit --output compliance.xml
  artifacts:
    reports:
      junit: compliance.xml
```

### Parallel Fleet Scanning
```bash
# Scan 100+ servers in parallel
ansible-inspec exec cis-ubuntu-20.04/ -i production.yml \
  --forks 50 \
  --reporter "cli json html"
```

## Getting Help

- **GitHub Issues**: [Report bugs or request features](https://github.com/Htunn/ansible-inspec/issues)
- **Documentation**: Browse the sections in this guide
- **API Reference**: See the [API Documentation](reference/api.md)

## License

ansible-inspec is licensed under **GPL-3.0-or-later**, combining:
- Ansible (GPL-3.0)
- InSpec (Apache-2.0)

See the LICENSE file for full details.

---

Ready to get started? Head to the [Getting Started Guide](guides/getting-started.md) →
