# ansible-inspec Documentation

Welcome to the **ansible-inspec** documentation! This tool combines Ansible's infrastructure automation with InSpec's compliance testing framework.

## What is ansible-inspec?

ansible-inspec enables you to:

- ✅ **Test infrastructure** configurations programmatically
- ✅ **Validate compliance** and security requirements
- ✅ **Integrate testing** into Ansible workflows
- ✅ **Convert InSpec profiles** to Ansible collections
- ✅ **Access 100+ profiles** from Chef Supermarket
- ✅ **Generate reports** in multiple formats (JSON, HTML, JUnit)

## Key Features

### 🚀 Easy Integration
Use your existing Ansible inventory for compliance testing across your infrastructure.

### 🔄 Profile Conversion
Convert Ruby-based InSpec profiles to native Ansible collections that run without InSpec.

### 📊 Flexible Reporting
Generate compliance reports in JSON, HTML, and JUnit formats for CI/CD integration.

### 🐳 Docker Ready
Available as a Docker image for easy deployment and isolation.

### 📚 Chef Supermarket Access
Leverage 100+ pre-built compliance profiles from the community.

## Quick Start

```bash
# Install
pip install ansible-inspec

# Run a compliance check
ansible-inspec exec profile/ --target local:// --reporter html:report.html

# Convert InSpec profile to Ansible collection
ansible-inspec convert my-profile --namespace myorg --collection-name compliance

# Use Chef Supermarket profiles
ansible-inspec exec dev-sec/linux-baseline --supermarket -i inventory.yml
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
