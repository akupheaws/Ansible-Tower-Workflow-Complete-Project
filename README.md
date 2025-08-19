## Ansible Tower Workflow Template Project Arch
![TowerWorkflowProjectArch](https://github.com/awanmbandi/aws-real-world-projects/blob/project-resources-docs/images/ansible_tower_automation_architecture_v5.png)

### 1) Create And Setup Your Ansible Tower Environment
- Ansible Tower Installation and Setup Runbook: https://scribehow.com/shared/Ansible_Tower_Setup_and_Configuration_Runbook_V2__uHuL7Z7eQCySip-xiK7ySA 

# Project Overview

This repository delivers a **complete, reproducible Ansible Tower / AWX Workflow** project that automates everything from **controller object bootstrapping** (organizations, inventories, credentials, projects) to **job templates**, **workflow templates**, **surveys**, and **notifications**—all as code.  
It’s built to be **portfolio‑ready** and **team‑friendly**: new environments can be spun up quickly, workflows are version‑controlled, and Day‑2 changes are tracked by pull requests. The repo also includes guidance for both **AWX (open‑source)** and **Ansible Automation Platform (AAP) Controller** (enterprise) using the modern **`awx.awx`** and **`ansible.controller`** collections.

---

# Ansible Tower / AWX Workflow — Complete Project

![Automation](https://img.shields.io/badge/Automation-Ansible-red)
![Controller](https://img.shields.io/badge/Controller-AWX%20%7C%20AAP-blue)
![IaC](https://img.shields.io/badge/Everything%20as%20Code-Controller%20Objects-blueviolet)
![License](https://img.shields.io/badge/license-MIT-informational)

---

## ✨ What this project does

- **Bootstraps Controller**: creates organizations, teams, users, inventories, hosts, groups, credentials, execution environments, projects, job templates, workflow job templates, schedules, notifications—**as code**.
- **Defines Reusable Workflows**: end‑to‑end workflows that chain job templates (e.g., build → test → deploy → verify), with conditionals, approvals, and failure branches.
- **Environment Parity**: one repo; multiple inventories (`dev`, `staging`, `prod`) and variable overlays.
- **GitOps‑friendly**: changes flow through PRs; optional CI validates object syntax before applying.
- **Idempotent & Auditable**: re‑runs safely; every controller change is captured in VCS.

---

## 📦 Suggested repository structure

> If your tree differs, keep the same intent: **objects and data are code** and live alongside playbooks and roles.

```
.
├─ playbooks/
│  ├─ bootstrap-controller.yml        # Creates orgs/teams/creds/projects/etc.
│  ├─ define-job-templates.yml        # Creates job templates
│  └─ define-workflows.yml            # Creates workflow job templates
├─ roles/                             # Optional roles for app tasks
│  └─ <your-roles>/
├─ inventories/
│  ├─ dev/
│  │  ├─ hosts.ini
│  │  └─ group_vars/
│  ├─ staging/
│  └─ prod/
├─ controller_objects/                # “Controller-as-code” definitions (YAML vars)
│  ├─ organizations.yml
│  ├─ teams.yml
│  ├─ users.yml
│  ├─ credentials.yml
│  ├─ projects.yml
│  ├─ inventories.yml
│  ├─ job_templates.yml
│  └─ workflows.yml
├─ vars/                              # shared vars, EE images, default creds ids
│  └─ main.yml
├─ collections/requirements.yml       # awx.awx / ansible.controller, etc.
├─ requirements.yml                   # Ansible roles (galaxy)
├─ scripts/                           # helper scripts (lint, apply, smoke tests)
├─ .github/workflows/                 # optional CI to lint/apply object definitions
└─ README.md
```

---

## 🧭 Architecture (workflow at a glance)

```
Developer PR → main
       │
       ▼
  CI Lint & Validate (ansible-lint, yamllint)
       │
       ▼
 Apply Controller Objects (awx.awx / ansible.controller)
       │
       ▼
   Job Templates created/updated ──┐
       │                           │
       ▼                           │
  Workflow Templates link stages ◄─┘
       │
       ▼
 Schedules / Surveys / Notifications
```

---

## 🚀 Getting Started

### Prerequisites

- **Ansible ≥ 2.14**
- Access to **AWX** or **AAP Controller** (URL + token or username/password)
- Python libs: `awxkit` (optional), `requests`
- Collections installed:
  ```bash
  ansible-galaxy collection install awx.awx ansible.controller
  ansible-galaxy install -r requirements.yml   # if using roles
  ```

### Authentication setup

Create an environment file or use Ansible vars for controller connection:

```yaml
# vars/controller.yml
controller_host: "https://awx.example.com"
controller_username: "admin"
controller_password: "!ChangeMe!"
controller_verify_ssl: false   # set true with valid certs
```

> Prefer **tokens** in CI. Store secrets in GitHub Actions / Jenkins credentials, not in Git.

### Bootstrap controller objects

Run the bootstrap playbook to create orgs, teams, credentials, inventories, and projects:

```bash
ansible-playbook -e @vars/controller.yml playbooks/bootstrap-controller.yml
```

Then create job templates and workflows:

```bash
ansible-playbook -e @vars/controller.yml playbooks/define-job-templates.yml
ansible-playbook -e @vars/controller.yml playbooks/define-workflows.yml
```

---

## 🧩 Example: controller-as-code (objects via vars)

**`controller_objects/projects.yml`**

```yaml
controller_projects:
  - name: "Platform Playbooks"
    description: "Core ops playbooks"
    organization: "Platform"
    scm_type: "git"
    scm_url: "https://github.com/your-org/platform-playbooks.git"
    scm_branch: "main"
    update_project: true
    wait: true
```

**`controller_objects/job_templates.yml`**

```yaml
controller_job_templates:
  - name: "Deploy App"
    organization: "Platform"
    inventory: "Prod Inventory"
    project: "Platform Playbooks"
    playbook: "playbooks/deploy.yml"
    execution_environment: "ee-supported:latest"
    credentials:
      - "Prod SSH Key"
      - "Vault"
    extra_vars:
      release_version: "latest"
```

**`controller_objects/workflows.yml`**

```yaml
controller_workflows:
  - name: "Blue-Green Deploy"
    organization: "Platform"
    state: "present"
    simplified_workflow_nodes:
      - identifier: "Build"
        unified_job_template: "Build Artifact"
      - identifier: "Deploy-Blue"
        unified_job_template: "Deploy App (blue)"
        success_nodes: ["Swap-Prod"]
      - identifier: "Swap-Prod"
        unified_job_template: "Switch Traffic to Blue"
        failure_nodes: ["Rollback"]
      - identifier: "Rollback"
        unified_job_template: "Rollback to Green"
```

---

## 🧪 CI integration (optional)

Add a minimal pipeline to lint, validate, and apply object code:

```yaml
# .github/workflows/controller.yml
name: Controller Objects
on: [push, pull_request]
jobs:
  lint-apply:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.11" }
      - name: Install Ansible & collections
        run: |
          pip install ansible-core ansible-lint yamllint
          ansible-galaxy collection install awx.awx ansible.controller
      - name: Lint
        run: |
          yamllint .
          ansible-lint
      - name: Apply to controller (on main only)
        if: github.ref == 'refs/heads/main'
        env:
          CONTROLLER_HOST: ${{ secrets.CONTROLLER_HOST }}
          CONTROLLER_USERNAME: ${{ secrets.CONTROLLER_USERNAME }}
          CONTROLLER_PASSWORD: ${{ secrets.CONTROLLER_PASSWORD }}
        run: |
          ansible-playbook -e controller_host=$CONTROLLER_HOST                            -e controller_username=$CONTROLLER_USERNAME                            -e controller_password=$CONTROLLER_PASSWORD                            playbooks/bootstrap-controller.yml
```

> Switch to **tokens** with `controller_oauthtoken` for production and set `controller_verify_ssl: true` with a valid CA chain.

---

## 🔐 Security & best practices

- **Store secrets** in CI secret stores (GitHub Actions, Jenkins) or Ansible Vault.
- Use **EE (Execution Environments)** with pinned collections and Python deps.
- Enforce **least privilege** on organization/team permissions and credentials.
- Enable **notifications** (Slack/Email) on workflow success/failure.
- Use **surveys** to collect runtime inputs (e.g., `release_version`).

---

## 🛣️ Roadmap

- [ ] Add **RBAC as code** (roles for teams/users).
- [ ] Add **Schedules** for recurring workflows (patching windows).
- [ ] Add **Policy checks** (ansible-lint rulesets, content signing).
- [ ] Add **Inventory sources** (cloud dynamic inventory) with cache control.
- [ ] Add **Approvals** in workflows for prod deploys.

---

## 🆘 Troubleshooting

- **401/403** → verify controller credentials and user permissions.  
- **SCM project sync fails** → validate repo URL/branch, webhook token, and PAT.  
- **Job template can’t find inventory/creds** → ensure those objects exist and names match.  
- **Workflow edges wrong** → inspect `simplified_workflow_nodes` identifiers; they must match.  

---

## 📜 License

MIT © Akuphe
