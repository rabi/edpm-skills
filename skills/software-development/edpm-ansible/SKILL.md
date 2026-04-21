---
name: edpm-ansible
description: >
  Work with the osp.edpm Ansible collection — the External Data Plane Management
  project for OpenStack on OpenShift. Covers role development, molecule testing,
  plugin authoring, playbook creation, linting, and container image builds.
trigger: Working in ~/work/openshift/edpm-ansible or any task involving EDPM Ansible roles, playbooks, or plugins.
---

# EDPM Ansible Collection (osp.edpm)

## What This Is

The `osp.edpm` Ansible collection manages the "external data plane" (compute and
network nodes) for OpenStack deployments running on OpenShift/Kubernetes. Part of
the `openstack-k8s-operators` ecosystem. Licensed Apache 2.0, by Red Hat.

- **Namespace/Name:** osp.edpm
- **Repo:** ~/work/openshift/edpm-ansible
- **Runtime:** Requires Ansible >= 2.15.0

## Repository Structure

```
edpm-ansible/
├── roles/                  # 51 Ansible roles (edpm_*)
├── playbooks/              # 38 playbooks (bootstrap, nova, ovn, neutron, etc.)
├── plugins/
│   ├── modules/            # 9 Python modules (os_net_config, nftables, container_manage, etc.)
│   ├── filter/             # Filter plugins (helpers.py + YAML filters)
│   └── action/             # Action plugin (container_systemd.py)
├── molecule/               # Shared molecule test resources
├── openstack_ansibleee/    # Container image (ansible-runner, UBI9-minimal based)
├── docs/                   # Sphinx/RST documentation
├── contribute/             # Role skeleton template (_skeleton_role_)
├── scripts/                # Utility scripts (molecule_venv.sh, test_roles.py)
├── tests/                  # Python unit tests (pytest)
├── zuul.d/                 # Zuul CI job definitions
├── galaxy.yml              # Collection metadata
├── requirements.yml        # Galaxy collection dependencies
├── molecule-requirements.txt # Python deps for molecule
├── Makefile                # Build/test automation
├── OWNERS / OWNERS_ALIASES # K8s-style code ownership
└── .ansible-lint           # Lint config (production profile)
```

## Key Commands

```bash
# 1. Setup local test environment
./scripts/molecule_venv.sh

# 2. Run molecule tests (containerized, requires Podman)
make execute_molecule

# 3. Run molecule tests locally
make execute_molecule_local

# 4. Run linters (pre-commit hooks: ansible-lint, flake8, yamllint, bashate)
make linters

# 5. Build the ansible execution environment container image
make openstack_ansibleee_build

# 6. Push the container image
make openstack_ansibleee_push

# 7. Build HTML documentation
make html_docs
```

## Creating a New Role

1. Generate from the skeleton:
   ```bash
   cd ~/work/openshift/edpm-ansible
   ansible-galaxy init --role-skeleton=contribute/_skeleton_role_ roles/edpm_<name>
   ```

2. Follow the naming convention: `edpm_<role_name>`

3. Required files in each role:
   - `meta/argument_specs.yml` — typed variable specs with defaults and descriptions
   - `defaults/main.yml` — default variable values
   - `tasks/main.yml` — entry point
   - `molecule/default/` — at least one molecule test scenario
   - Documentation in `docs/source/roles/role-<name>.rst`

4. Variable naming: `edpm_<role_name>_<variable_purpose>`

## Ansible Coding Standards (MANDATORY)

- **FQCN always:** Use `ansible.builtin.copy` not `copy`, `ansible.builtin.service` not `service`
- **Every task needs a name:** Descriptive, meaningful task names required
- **No Jinja in conditionals:** Use `when: my_var` not `when: "{{ my_var }}"`
- **Modules over shell:** Prefer specific modules over `shell` or `command`
- **Idempotency:** Tasks must be idempotent — running twice = same result
- **Variables in defaults:** Use `defaults/main.yml`, don't hardcode in tasks
- **Become at task level:** Use `become: true` on individual tasks, not at play level
- **Proper reporting:** Always set `changed_when` and `failed_when` on command/shell tasks

## Molecule Testing

Each role has molecule tests under `roles/edpm_<name>/molecule/`.

```bash
# Test a specific role locally
cd roles/edpm_<role_name>
molecule test

# Test with verbose output
molecule test -v

# Just converge (don't destroy)
molecule converge

# Run verify step only
molecule verify
```

Molecule uses the **podman** driver. Tests run on Ubuntu 22.04 CI with Python 3.9.

## Plugin Development

### Custom Modules (plugins/modules/)
Existing modules: os_net_config, nftables, container_manage, etc.
Follow standard Ansible module conventions with DOCUMENTATION, EXAMPLES, RETURN docstrings.

### Filter Plugins (plugins/filter/)
Custom Jinja2 filters in `helpers.py` and YAML-based filter definitions.

### Action Plugins (plugins/action/)
`container_systemd.py` — custom action plugin for container systemd units.

## Dependencies

**Ansible Galaxy collections (requirements.yml):**
- ansible.posix
- community.general
- containers.podman
- fedora.linux_system_roles

**Python testing deps (molecule-requirements.txt):**
- ansible-core >= 2.15.0
- molecule >= 5.1.0, < 6.0.0
- molecule-plugins[podman,vagrant] >= 23.5.0
- pytest, pytest-testinfra, pytest-cov

## CI/CD

**GitHub Actions:**
- `molecule.yaml` — Per-role molecule tests on PRs (matrix of ~25 roles, only tests modified roles)
- `build-collection.yaml` — Builds and installs collection on PRs
- `openstack-ansibleee-runner.yaml` — Builds/pushes container to quay.io
- `documentation.yaml` — Sphinx docs to GitHub Pages
- `codeql.yml` — Weekly security scanning
- `release-branch-sync.yaml` — Hourly release branch sync

**Zuul CI:**
- 16 per-role molecule jobs (cifmw-molecule-base)
- Integration tests: edpm-ansible-tempest-multinode, edpm-ansible-baremetal-minor-update

## Container Image

The `openstack-ansibleee-runner` image is the execution environment:
- Base: UBI9-minimal (multi-stage build)
- Contains: ansible-core >= 2.16.1, ansible-runner >= 2.4.0, ansible-builder >= 3.1.0
- Registry: quay.io/openstack-k8s-operators/openstack-ansibleee-runner
- Build: `make openstack_ansibleee_build`
- Push: `make openstack_ansibleee_push`
- Makefile vars: `IMAGE_TAG_BASE`, `IMAGE_TAG`

## Code Ownership

K8s-style OWNERS/OWNERS_ALIASES with approver groups:
- edpm, ci, openstack, compute, telemetry, storage, network

## Pitfalls

1. **Always run linters before committing** — `make linters` runs pre-commit hooks
   (ansible-lint production profile, flake8, yamllint, bashate)
2. **Molecule needs Podman** — containerized tests require podman installed
3. **FQCN enforcement** — ansible-lint will fail on non-FQCN module references
4. **The AGENTS.md has duplicate content** — sections are repeated; refer to the
   first half for accurate info
5. **Variable naming is strict** — must follow `edpm_<role>_<purpose>` pattern
6. **argument_specs.yml is mandatory** — roles without typed specs will fail review
