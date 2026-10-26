# Kubernetes Cluster Provisioning with Ansible

An **Ansible** project that automatically provisions a **Kubernetes** cluster.

## Architecture

```
Ansible Controller
│
└───────────────┬ Kubernetes Control Plane  (control-plane-01)
│
└───────────────┬ Worker 1                  (worker-01)
│
└───────────────┬ Worker 2                  (worker-02)
```

- **1 Ansible Controller** — only manages the other servers over SSH; NOT part of the cluster.
- **1 Kubernetes Control Plane** — runs kubeadm, kubelet, kubectl.
- **2 Kubernetes Worker Nodes** — run kubelet and containerd.

## Infrastructure

| Host              | Role          | IP              |
|-------------------|---------------|-----------------|
| control-plane-01  | control-plane | 10.1.0.10       |
| worker-01         | worker        | 10.1.0.8        |
| worker-02         | worker        | 10.1.0.6        |

> IPs are set in `inventory/production/hosts.yml`. Update them if your VM addresses differ.
## Project Structure — Every Folder Explained

```
ansible-project/
├── ansible.cfg                  # Ansible configuration
├── .gitignore                   # Files Git should ignore
├── .ansible-lint                # Linting rules
├── README.md                    # You are here
│
├── inventory/                   # THE SOURCE OF TRUTH for hosts
│   ├── production/               #   Production environment
│   │   ├── hosts.yml             #     Host definitions + groups
│   │   ├── group_vars/           #     Group variables
│   │   └── host_vars/            #     Host-specific variables
│   └── staging/                 #   Staging environment
│       ├── hosts.yml
│       └── group_vars/
│
├── group_vars/                  # Alternative location (root level)
├── host_vars/                   # Alternative location (root level)
│
├── playbooks/                   # Playbook entry-points
│   ├── site.yml                 #   Master playbook
│   ├── bootstrap.yml            #   Initial server setup
│   └── kubernetes-cluster.yml   #   K8s provisioning
│
├── roles/                       # REUSABLE automation units
│   ├── common/                  #   OS prep (packages, swap, sysctl)
│   │   ├── tasks/              #     main.yml - the actual work
│   │   ├── handlers/           #     Service restarts etc.
│   │   ├── templates/           #     Jinja2 templates
│   │   ├── files/               #     Static files
│   │   ├── defaults/            #     Default variables (lowest)
│   │   ├── vars/                #     Internal variables
│   │   └── meta/                #     Metadata, deps
│   ├── containerd/              #   Container runtime
│   ├── kubernetes/              #   kubeadm, kubelet, kubectl
│   ├── control-plane/           #   kubeadm init
│   └── worker/                  #   Join cluster
│
├── templates/                   # Shared Jinja2 templates
├── files/                       # Shared static files
├── scripts/                     # Helper shell scripts
├── docs/                        # Documentation, diagrams
└── tests/                       # Molecule / test scenarios
```
---

## Why Each Folder Exists

### `ansible.cfg`
- **Why:** Pins Ansible behaviour so everyone on the team gets the same results.
- **When loaded:** Every time you run an Ansible command.
- **Optional?** Yes, but professionals always include it.

### `inventory/`
- **Why:** The inventory is Ansible's phone book — maps hostnames to IPs and groups.
- **When loaded:** Every playbook run starts by parsing the inventory.
- **What belongs:** Host definitions, group membership, connection vars.
- **What NOT:** Playbooks, roles, templates.

### `inventory/production/` vs `inventory/staging/`
- **Why:** Separate environments need separate host lists.
- **Switching:** Change `ansible.cfg` or use `-i inventory/staging/`.

### `group_vars/` (inside inventory)
- **Why:** Keeps variables separate from host definitions for readability.
- **When loaded:** Automatically every playbook run.
- **What belongs:** Environment-level settings.
- **What NOT:** Secrets (use Ansible Vault).

### `host_vars/` (inside inventory)
- **Why:** Host-specific overrides for truly unique settings.
- **When loaded:** Alongside group_vars for the specific host.

### `playbooks/`
- **Why:** Entry points that map roles to hosts.
- **When loaded:** Explicitly (`ansible-playbook playbooks/site.yml`).

### `roles/`
- **Why:** The HEART of Ansible reusability. Self-contained units of automation.
- **When loaded:** When a playbook references the role.

#### `roles/<role>/tasks/`
Contains the ordered task list. `main.yml` is the auto-loaded entry point.

#### `roles/<role>/handlers/`
Special tasks that run ONLY when notified, ONCE at play end.

#### `roles/<role>/templates/`
Jinja2 templates for dynamic config files (variables replaced at copy time).

#### `roles/<role>/files/`
Static files copied as-is (no templating).

#### `roles/<role>/defaults/`
Default variable values — LOWEST precedence. Overridable by anything.

#### `roles/<role>/vars/`
Internal variables — higher precedence than defaults, not meant for override.

#### `roles/<role>/meta/`
Role metadata: dependencies, author, platforms. Resolved before the role runs.

### `templates/` and `files/` (root)
Shared templates/files used by multiple roles.

### `scripts/`
Utility shell scripts for non-Ansible tasks.

### `docs/`
Project documentation, architecture diagrams, runbooks.

### `tests/`
Automated tests (Molecule scenarios).

---

## How to Use This Project

1. **Clone the repo** on your Ansible Controller.
2. **Install Ansible** (see docs.ansible.com).
3. **Update IPs** in `inventory/production/hosts.yml`.
4. **Set up SSH keys** for the `ansible` user.
5. **Run the bootstrap:**
   ```bash
   ansible-playbook playbooks/bootstrap.yml
   ```
6. **Provision the cluster:**
   ```bash
   ansible-playbook playbooks/kubernetes-cluster.yml
   ```
7. **Verify:**
   ```bash
   ansible-playbook playbooks/verify.yml
   ```

---

## Ansible Concepts Covered

| Concept        | Stage | Why It Matters                             |
|----------------|-------|--------------------------------------------|
| Inventory      | 3     | Defines hosts and groups                   |
| Variables      | 4     | Makes playbooks reusable                   |
| Roles          | 5-9   | The unit of reusability                    |
| Handlers       | 5     | React to changes (restart on change)       |
| Templates      | 5     | Dynamic config files (Jinja2)              |
| Facts          | 5     | Automatic system info                      |
| Register       | 8     | Capture task output into variables         |
| Conditionals   | 5+    | Run tasks only when needed (idempotency)   |
| Loops          | 5     | Repeat tasks across items                  |
| Notify         | 5     | Trigger handlers from tasks                |
| Blocks/Rescue  | 11    | Error handling                             |
| Tags           | All   | Run subsets of tasks                       |
| Vault          | 4+    | Encrypt secrets in the repo                |
| Check Mode     | All   | Dry-run to see what WOULD change           |

---

## Best Practices Followed

- **Idempotency** — every task can be run repeatedly without side effects.
- **Separation of concerns** — one role = one responsibility.
- **No hardcoded values** — everything configurable via variables.
- **Self-documenting** — every task has a clear `name:`.
- **Variable naming** — descriptive, lowercase, underscore-separated.
- **Handlers for restarts** — avoid restarting on every run.
- **Check mode safe** — tasks use `check_mode:` where needed.
- **.gitignore + .ansible-lint** — professional quality out of the box.

---

## License

This project is for educational purposes. Use freely, adapt, and learn.