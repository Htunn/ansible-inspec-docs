# Architecture & How It Works

This guide explains the internal architecture of ansible-inspec and how the different components work together.

## System Architecture

ansible-inspec is built with a layered architecture that separates concerns and provides maximum flexibility:

```
┌─────────────────────────────────────────────────────┐
│           User Interface Layer                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  CLI Interface (Argument Parsing)           │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│              Core Layer                              │
│  ┌──────────────┐  ┌───────────────────────────┐   │
│  │ Core Engine  │  │ Ansible Inventory         │   │
│  │ Orchestration│  │ Host Management           │   │
│  └──────────────┘  └───────────────────────────┘   │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│           Execution Layer                            │
│  ┌──────────────────┐  ┌───────────────────────┐   │
│  │ InSpec Adapter   │  │ Profile Converter     │   │
│  │ Profile Execution│  │ InSpec → Ansible      │   │
│  └──────────────────┘  └───────────────────────┘   │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│            Output Layer                              │
│  ┌───────────┐ ┌──────────────┐ ┌─────────────┐   │
│  │Multi-Format│ │Callback Plugin│ │Report Storage│   │
│  │ Reporter  │ │ InSpec-free   │ │.compliance- │   │
│  │           │ │ mode          │ │ reports/    │   │
│  └───────────┘ └──────────────┘ └─────────────┘   │
└─────────────────────────────────────────────────────┘
```

## Key Components

### 1. CLI Interface
- Entry point for all user interactions
- Argument parsing and validation
- Command routing (exec, convert, supermarket)

### 2. Core Engine
- Orchestrates the entire compliance workflow
- Manages execution modes (native InSpec, InSpec-free, Supermarket)
- Coordinates between adapters and reporters

### 3. Ansible Inventory
- Leverages existing Ansible inventory files
- Supports all Ansible inventory formats (YAML, INI, dynamic)
- Host and group variable resolution
- Connection management

### 4. InSpec Adapter
- Wraps InSpec CLI for profile execution
- Parallel processing using Ansible forks
- Result collection and aggregation
- Error handling and logging

### 5. Profile Converter
- Parses InSpec Ruby DSL
- Maps InSpec resources to Ansible modules
- Generates Ansible collection structure
- Bundles callback plugin for auto-reporting

### 6. Reporter System
- Multi-format report generation:
  - JSON (InSpec schema v5.22.0)
  - HTML (interactive dashboards)
  - JUnit (CI/CD integration)
  - CLI (real-time output)
- InSpec schema compatibility
- Report storage and archival

### 7. Callback Plugin
- Auto-bundled in converted collections
- Generates InSpec-compatible reports
- Works in InSpec-free mode
- Zero configuration required

## Execution Flow

### Native InSpec Mode

```
User Command
    ↓
Load InSpec Profile
    ↓
Parse Ansible Inventory
    ↓
Check InSpec Installation
    ↓
Run InSpec on Each Host (Parallel)
    ↓
Collect Results
    ↓
Generate Reports (JSON/HTML/JUnit)
    ↓
Save to .compliance-reports/
    ↓
Display Summary
```

### InSpec-Free Mode

```
User Command (convert)
    ↓
Load InSpec Profile
    ↓
Parse InSpec Ruby DSL
    ↓
Map Resources to Ansible Modules
    ↓
Generate Collection Structure
    ↓
Bundle Callback Plugin
    ↓
Create Playbooks & Roles
    ↓
Package as .tar.gz
    ↓
Install & Execute (Pure Ansible!)
    ↓
Callback Auto-Generates Reports
```

### Supermarket Mode

```
User Command (--supermarket)
    ↓
Search/Download from Chef Supermarket
    ↓
Cache Profile (~/.inspec/cache)
    ↓
Continue as Native InSpec Mode
```

## Parallel Execution

ansible-inspec leverages Ansible's parallelism:

```python
# Simplified parallel execution logic
for host in inventory.hosts:
    # Fork process for each host (up to --forks limit)
    process = fork_inspec_execution(host, profile)
    processes.append(process)

# Wait for all processes to complete
results = wait_for_completion(processes)

# Aggregate results
final_report = aggregate_results(results)
```

**Benefits:**
- Scan 100+ servers simultaneously
- Configurable parallelism via `--forks`
- Efficient resource utilization
- Faster compliance audits

## Resource Translation

When converting InSpec profiles to Ansible:

```ruby
# InSpec resource
describe security_policy do
  its("MaximumPasswordAge") { should cmp == 365 }
end
```

Converts to:

```yaml
# Ansible tasks
- name: Export security policy
  ansible.windows.win_shell: secedit /export /cfg secpol.cfg
  register: secpol_export

- name: Parse MaximumPasswordAge
  ansible.builtin.set_fact:
    max_password_age: "{{ secpol_export | parse_secedit }}"

- name: Validate security policy
  ignore_errors: true
  ansible.builtin.assert:
    that:
      - max_password_age | int == 365
  register: control_result
```

**Translation Rules:**
- `security_policy` → `win_shell` with secedit
- `registry_key` → `win_reg_stat`
- `audit_policy` → `win_shell` with auditpol
- `service` → `win_service_info`
- `windows_feature` → `win_feature`
- `file` → `stat` or `win_stat`

## Report Generation

### InSpec Schema Compatibility

Generated JSON reports follow InSpec schema v5.22.0:

```json
{
  "platform": {
    "name": "ubuntu",
    "release": "20.04"
  },
  "profiles": [{
    "name": "linux-baseline",
    "version": "2.9.1",
    "controls": [{
      "id": "sysctl-01",
      "title": "Kernel parameter hardening",
      "results": [{
        "status": "passed",
        "code_desc": "Kernel parameter should eq 1",
        "run_time": 0.002
      }]
    }]
  }],
  "statistics": {
    "duration": 5.23
  }
}
```

This ensures compatibility with:
- Chef Automate
- InSpec reporters
- Compliance visualization tools

## Error Handling

ansible-inspec implements comprehensive error handling:

1. **Profile Loading Errors**
   - Invalid profile structure
   - Missing dependencies
   - Syntax errors in InSpec code

2. **Execution Errors**
   - Connection failures
   - Permission issues
   - InSpec runtime errors

3. **Conversion Errors**
   - Unsupported InSpec resources
   - Complex Ruby logic
   - Custom resource mapping

Each error includes:
- Clear error messages
- Suggested remediation
- Stack traces (when needed)
- Non-zero exit codes for CI/CD

## Performance Optimization

### Caching
- Downloaded Supermarket profiles cached in `~/.inspec/cache`
- Converted collections reusable across runs
- Ansible fact caching supported

### Parallelism
- Configurable via `--forks` flag
- Default matches Ansible configuration
- Scales to 100+ concurrent hosts

### Resource Efficiency
- Lightweight callback plugin (<100 lines)
- Minimal memory footprint
- Efficient result aggregation

## Security Considerations

1. **Credential Management**
   - Uses Ansible's credential system
   - Supports ansible-vault for encrypted vars
   - No credentials stored in reports

2. **Execution Isolation**
   - InSpec runs in separate processes
   - No shared state between hosts
   - Clean environment per execution

3. **Report Security**
   - Reports stored locally by default
   - No automatic upload to external systems
   - User controls report distribution

## Extension Points

ansible-inspec supports customization:

1. **Custom Reporters**
   - Implement ReporterInterface
   - Add to reporter registry
   - Generate custom report formats

2. **Custom Resource Translators**
   - Map InSpec resources to Ansible modules
   - Register in translator registry
   - Extend conversion capabilities

3. **Callback Plugin Customization**
   - Modify bundled callback
   - Add custom report fields
   - Integrate with external systems

## Next Steps

- [Quick Reference](quick-reference.md) - Command examples
- [Profile Conversion Guide](profile-conversion.md) - Detailed conversion process
- [Reporter Modes](reporter-modes.md) - Report format details
- [API Documentation](../reference/api.md) - Python API reference
