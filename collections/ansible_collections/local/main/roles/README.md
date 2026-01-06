# Ansible Roles

This directory contains the Ansible roles that implement the benchmark execution phases.

## Roles Overview

| Role | Phase | Purpose |
|------|-------|---------|
| `environment_setup` | 1 | Create directories, install dependencies, clone submodule |
| `benchmark_validator` | 2 | Validate Docker, Python, jq, client configs |
| `benchmark_runner` | 3 | Execute benchmarks, collect results and logs |
| `postgres_publisher` | 4 | Publish results to PostgreSQL (optional) |

## Execution Flow

```
environment_setup → benchmark_validator → benchmark_runner → postgres_publisher
     (setup)            (validate)          (per client)        (optional)
```

## Usage Example

### Inventory Configuration

```yaml
# inventory/hosts.yml
---
all:
  hosts:
    # Remote server example
    benchmark-server:
      ansible_host: 192.168.1.100
      ansible_user: benchmark

      # Host-specific overrides (optional)
      benchmark_clients:
        - nethermind
        - geth

  vars:
    # Required: workspace directory on target host
    workspace_root: /home/benchmark/gas-benchmarks-v2

    # Optional overrides (defaults in group_vars/all.yml)
    benchmark_runs: 3
```

### Running Benchmarks

```bash
# Run all phases
ansible-playbook collections/ansible_collections/local/main/playbooks/run_benchmarks.yml \
  -i inventory/hosts.yml

# Skip setup phase (already initialized)
ansible-playbook collections/ansible_collections/local/main/playbooks/run_benchmarks.yml \
  -i inventory/hosts.yml \
  --skip-tags setup

# Run only benchmarks (skip validation)
ansible-playbook collections/ansible_collections/local/main/playbooks/run_benchmarks.yml \
  -i inventory/hosts.yml \
  --tags benchmarks

# Include PostgreSQL ingestion
ansible-playbook collections/ansible_collections/local/main/playbooks/run_benchmarks.yml \
  -i inventory/hosts.yml \
  --tags benchmarks,postgres
```

## Role Details

### environment_setup

Prepares the execution environment:
- Creates workspace directories
- Clones/updates gas-benchmarks submodule
- Installs Python dependencies via uv
- Installs system tools (yq, dotnet, rsync)
- Builds Nethermind.Tools.Kute

**Tags:** `setup`

### benchmark_validator

Validates prerequisites before execution:
- Checks tool versions (Docker, Python, jq, dotnet)
- Verifies Docker daemon is running
- Validates client configurations exist
- Checks disk space
- Creates lock file to prevent concurrent runs

**Tags:** `always`

### benchmark_runner

Executes benchmarks for each client sequentially:
- Cleans previous execution state
- Deploys client Docker container
- Runs warmup tests (if configured)
- Executes benchmark tests
- Collects Docker logs
- Syncs results to timestamped directories

**Tags:** `benchmarks`

### postgres_publisher

Publishes results to PostgreSQL (optional):
- Parses CSV result files
- Connects to configured database
- Inserts metrics into specified table

**Tags:** `postgres`

**Required vars:** `postgres_host`, `postgres_user`, `postgres_password`

## Creating New Roles

```bash
make new-role NAME=my_new_role
```

This creates the standard role structure under `roles/my_new_role/`.
