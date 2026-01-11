# Real-World Use Cases & Examples

This guide showcases practical scenarios where ansible-inspec solves real compliance and security challenges.

## Use Case #1: Monthly PCI Compliance Reporting

### The Challenge
Generate monthly PCI-DSS compliance reports for 120 payment processing servers for auditors.

### The Old Way
```bash
# Manual process taking 8-10 hours:
# 1. SSH into each server group
# 2. Run various compliance checks
# 3. Export results to spreadsheets  
# 4. Manually compile into a report
# 5. Send to auditors
```

### The ansible-inspec Way
```bash
# Single command, 5 minutes total
ansible-inspec exec pci-dss-baseline/ -i pci-servers.yml \
  --reporter "html:pci-compliance-$(date +%Y%m).html"
```

### Results
- ⏱️ Time reduced from 10 hours to 5 minutes
- 📊 Consistent report format auditors love
- ✅ Automated monthly execution via cron
- 📈 100% reproducible results

---

## Use Case #2: Zero-Trust Migration

### The Challenge
Migrate 200+ servers to zero-trust architecture. Need to verify each server meets security baseline before migration.

### The Problem
- Impossible to verify manually at scale
- Would require hiring contractors
- Project timeline at risk

### The Solution

**Step 1: Convert baseline to Ansible collection**
```bash
ansible-inspec convert zero-trust-baseline/ \
  --namespace security \
  --collection-name zerotrust
```

**Step 2: Distribute to all teams**
```bash
# Build collection
cd collections/ansible_collections/security/zerotrust
ansible-galaxy collection build

# Share .tar.gz with all teams
```

**Step 3: Each team verifies independently**
```bash
# No InSpec dependency needed!
ansible-galaxy collection install security-zerotrust-*.tar.gz
ansible-playbook security.zerotrust.compliance_check -i their-inventory.yml
```

### Results
- 🚀 Migration completed 2 months ahead of schedule
- 👥 Every team could verify compliance independently
- 💰 No consultant fees required
- 📦 Reusable collection for future migrations

---

## Use Case #3: CI/CD Compliance Gates

### The Challenge
Prevent non-compliant servers from reaching production. Manual checks sometimes let issues slip through.

### The Solution: GitLab CI Integration

```yaml
# .gitlab-ci.yml
stages:
  - test
  - deploy

compliance_check:
  stage: test
  image: htunnthuthu/ansible-inspec:latest
  script:
    - ansible-inspec exec security-baseline/ \
        -i staging-inventory.yml \
        --reporter junit --output compliance.xml
  artifacts:
    reports:
      junit: compliance.xml
    when: always
  only:
    - merge_requests
    - main

deploy_production:
  stage: deploy
  dependencies:
    - compliance_check
  script:
    - ansible-playbook deploy.yml -i production.yml
  only:
    - main
```

### Results
- ✅ Compliance is now part of the pipeline
- 🛡️ Non-compliant changes caught automatically
- 📊 Compliance history tracked in CI/CD
- 0️⃣ Zero production compliance incidents since implementation

---

## Use Case #4: Multi-Region Cloud Compliance

### The Challenge
Ensure consistent security baseline across AWS regions (us-east-1, eu-west-1, ap-southeast-1).

### The Inventory Setup

```yaml
# inventory.yml
all:
  children:
    aws_us_east:
      hosts:
        web[01:10]:
          ansible_host: "{{ lookup('aws_ec2', 'web-' + inventory_hostname) }}"
    aws_eu_west:
      hosts:
        web[01:08]:
          ansible_host: "{{ lookup('aws_ec2', 'web-' + inventory_hostname) }}"
    aws_ap_southeast:
      hosts:
        web[01:12]:
          ansible_host: "{{ lookup('aws_ec2', 'web-' + inventory_hostname) }}"
  vars:
    ansible_user: ubuntu
    ansible_ssh_private_key_file: ~/.ssh/cloud-key.pem
```

### The Execution

```bash
# Scan all regions in parallel
ansible-inspec exec cis-ubuntu-22.04/ -i inventory.yml \
  --forks 30 \
  --reporter "json:compliance-$(date +%Y%m%d).json" \
              "html:compliance-$(date +%Y%m%d).html"

# Filter by region
ansible-inspec exec cis-ubuntu-22.04/ -i inventory.yml \
  --limit aws_us_east \
  --reporter html:us-east-compliance.html
```

### Results
- 🌍 Consistent compliance across all regions
- ⚡ Parallel scanning of 30+ servers simultaneously
- 📊 Region-specific reports when needed
- 🔄 Automated via scheduled jobs

---

## Use Case #5: GDPR Data Protection Baseline

### The Challenge
New GDPR requirements. Need data protection baseline but unsure where to start.

### Discovering Pre-Built Profiles

```bash
# Search Chef Supermarket
ansible-inspec supermarket search data protection

# Found useful profiles:
# - dev-sec/linux-baseline (includes GDPR controls)
# - hardening-io/os-hardening
```

### Testing Profiles

```bash
# Test on staging first
ansible-inspec exec dev-sec/linux-baseline --supermarket \
  -i staging.yml \
  --reporter html:gdpr-test.html

# Review HTML report
open gdpr-test.html

# Customize for production
ansible-inspec convert dev-sec/linux-baseline \
  --namespace gdpr \
  --collection-name data_protection

# Modify roles/tasks as needed for GDPR specifics
```

### Results
- 📚 Started with battle-tested community profiles
- ⏱️ Saved months of development time
- ✅ Customized for organization-specific GDPR requirements
- 🔄 Continuous compliance monitoring

---

## Use Case #6: Parallel Fleet Scanning

### The Challenge
Weekly security scans of 500+ web servers across multiple data centers.

### The Setup

```bash
# inventory.yml with 500+ hosts
all:
  children:
    datacenter_1:
      hosts:
        web[001:200]:
    datacenter_2:
      hosts:
        web[201:400]:
    datacenter_3:
      hosts:
        web[401:500]:
```

### High-Performance Execution

```bash
# Maximize parallelism
ansible-inspec exec web-security-baseline/ \
  -i inventory.yml \
  --forks 50 \
  --reporter "json:weekly-scan-$(date +%Y%m%d).json" \
  | tee scan-output.log

# Monitor progress
tail -f scan-output.log
```

### Results
- ⚡ 500 servers scanned in under 10 minutes
- 💪 Efficient resource utilization
- 📊 Comprehensive JSON reports for analysis
- 🔄 Automated weekly via cron

---

## Use Case #7: Air-Gapped Environment

### The Challenge
Compliance testing in air-gapped government network. No internet access, can't install InSpec.

### The Approach

**Outside Air-Gap: Prepare Collection**
```bash
# On internet-connected system
ansible-inspec convert cis-rhel-8/ \
  --namespace government \
  --collection-name security_baseline

# Build collection
cd collections/ansible_collections/government/security_baseline
ansible-galaxy collection build

# Transfer government-security_baseline-1.0.0.tar.gz via approved method
```

**Inside Air-Gap: Deploy and Run**
```bash
# Install collection (no internet needed!)
ansible-galaxy collection install government-security_baseline-1.0.0.tar.gz

# Run compliance checks
ansible-playbook government.security_baseline.compliance_check \
  -i classified-inventory.yml

# Reports auto-generated
ls .compliance-reports/
```

### Results
- ✅ Full compliance testing without internet
- 🔒 Zero external dependencies
- 📦 Self-contained distribution
- 🛡️ Meets security requirements

---

## Use Case #8: Vendor Compliance Verification

### The Challenge
Verify third-party vendor systems meet security requirements before integration.

### The Workflow

**Step 1: Define Requirements**
```bash
# Create vendor-requirements profile
ansible-inspec init profile vendor-security-requirements

# Add controls for:
# - TLS version requirements
# - Authentication standards
# - Data encryption requirements
# - Logging requirements
```

**Step 2: Send to Vendor**
```bash
# Package as InSpec-free collection
ansible-inspec convert vendor-security-requirements/ \
  --namespace client \
  --collection-name vendor_verification

ansible-galaxy collection build

# Send vendor the .tar.gz and instructions
```

**Step 3: Vendor Self-Assessment**
```bash
# Vendor runs on their systems
ansible-galaxy collection install client-vendor_verification-1.0.0.tar.gz
ansible-playbook client.vendor_verification.compliance_check \
  -i vendor-systems.yml \
  --reporter html:compliance-report.html

# Vendor sends back compliance-report.html
```

### Results
- 📋 Standardized vendor verification
- 🤝 Self-service for vendors
- ✅ Objective compliance evidence
- 📊 Easy to review HTML reports

---

## Use Case #9: Desktop Fleet Management

### The Challenge
Ensure 1,000+ employee laptops meet security baseline (Windows 10/11).

### The Inventory

```yaml
# inventory.yml
all:
  children:
    windows_laptops:
      hosts:
        laptop[001:1000]:
          ansible_host: "{{ inventory_hostname }}.corp.local"
      vars:
        ansible_connection: winrm
        ansible_winrm_transport: kerberos
        ansible_port: 5985
```

### The Execution

```bash
# Use CIS Windows 10 benchmark from Supermarket
ansible-inspec exec cis-windows-10-enterprise/ --supermarket \
  -i inventory.yml \
  --limit "windows_laptops[001:100]" \
  --forks 20 \
  --reporter "json:laptop-compliance-batch1.json"

# Batch process all laptops in groups of 100
```

### Results
- 💻 Automated laptop compliance verification
- 📊 Identify non-compliant systems
- 🔧 Generate remediation reports
- 🔄 Quarterly compliance audits

---

## Use Case #10: Container Security Baseline

### The Challenge
Verify Docker host security configuration before deploying containerized applications.

### The Profile

```bash
# Use Docker security baseline from Supermarket
ansible-inspec exec dev-sec/docker-baseline --supermarket \
  -i docker-hosts.yml \
  --reporter "cli json html"
```

### Integration with CI/CD

```yaml
# .github/workflows/docker-compliance.yml
name: Docker Host Compliance

on:
  schedule:
    - cron: '0 2 * * *'  # Daily at 2 AM

jobs:
  compliance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Install ansible-inspec
        run: pip install ansible-inspec
      
      - name: Run Docker compliance check
        run: |
          ansible-inspec exec dev-sec/docker-baseline --supermarket \
            -i inventory/docker-hosts.yml \
            --reporter junit:docker-compliance.xml
      
      - name: Publish results
        uses: actions/upload-artifact@v3
        with:
          name: compliance-report
          path: docker-compliance.xml
```

### Results
- 🐳 Automated Docker host security verification
- 🔄 Daily compliance monitoring
- 📊 Historical compliance tracking
- ✅ Automated alerts for drift

---

## Pro Tips from Production

### Tip #1: Parallel Execution Tuning

```bash
# Start conservative
ansible-inspec exec profile/ -i inventory.yml --forks 10

# Monitor system resources
# Increase if CPU/network allows
ansible-inspec exec profile/ -i inventory.yml --forks 25

# Maximum speed (be careful!)
ansible-inspec exec profile/ -i inventory.yml --forks 50
```

### Tip #2: Report Archiving

```bash
# Create monthly archives
ansible-inspec exec profile/ -i prod.yml \
  --reporter "json html" \
  --output-dir ./audit-archive/$(date +%Y-%m)

# Compress for storage
tar czf compliance-$(date +%Y%m%d).tar.gz .compliance-reports/
```

### Tip #3: Filter by Tags

```bash
# Only critical controls
ansible-inspec exec profile/ -i inventory.yml \
  --tags critical,security

# Skip experimental controls
ansible-inspec exec profile/ -i inventory.yml \
  --skip-tags experimental
```

### Tip #4: Use with Ansible Vault

```bash
# Protect sensitive inventory data
ansible-vault encrypt inventory.yml

# Run with vault password
ansible-inspec exec profile/ -i inventory.yml \
  --ask-vault-pass \
  --reporter html
```

### Tip #5: Custom Report Naming

```bash
# Include environment and date
ENV=production
ansible-inspec exec cis-ubuntu/ -i ${ENV}.yml \
  --reporter "html:${ENV}-cis-$(date +%Y%m%d-%H%M%S).html"
```

## Next Steps

- [Getting Started Guide](getting-started.md) - Basic setup
- [Profile Conversion](profile-conversion.md) - InSpec-free mode details
- [Reporter Modes](reporter-modes.md) - Report format options
- [Architecture](architecture.md) - How it all works
