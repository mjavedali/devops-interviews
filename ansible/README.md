```
ELI7. Complete Ansible knowledge with a project example
```
Ansible is a tool to configure servers automatically.

Think of it like a remote-control checklist:

> “Install Nginx, copy my app files, start the service, and make sure it stays running.”

Instead of logging into 20 servers and repeating those steps, Ansible does it consistently.

## 1. Core concepts

| Term | ELI7 meaning | Interview meaning |
|---|---|---|
| Control node | Your laptop/CI server running Ansible | Machine where Ansible commands execute |
| Managed nodes | Target servers | Hosts configured by Ansible |
| Inventory | Address book of servers | Static/dynamic host definitions and groups |
| Playbook | Instruction book | YAML file containing plays/tasks |
| Play | A plan for one server group | Maps hosts to roles/tasks |
| Task | One action | A single module invocation |
| Module | A built-in worker/tool | Idempotent unit that makes a change |
| Role | Reusable folder of automation | Standard structure for tasks, vars, handlers, templates, etc. |
| Collection | Package of modules and roles | Namespaced Ansible content distribution |
| Handler | “Do this only if something changed” | Event-driven task, commonly service restart |
| Variable | Reusable value | Data supplied at different precedence levels |
| Template | File with placeholders | Jinja2-rendered configuration file |
| Vault | Locked secret box | Encrypts secrets at rest |

## 2. Why teams use Ansible

- Agentless: targets normally need only SSH and Python; Windows uses WinRM.
- Declarative: describe the desired final state.
- Idempotent: running it again should not create unnecessary changes.
- Simple YAML, reusable roles, easy CI/CD integration.
- Good for provisioning, configuration management, deployments, patching, and operational tasks.

Important interview line:

> Ansible is primarily configuration orchestration. Terraform is usually preferred for provisioning cloud infrastructure, while Ansible configures the OS and applications running on it.

## 3. Basic project layout

```text
ansible-project/
├── ansible.cfg
├── inventory/
│   ├── dev.ini
│   └── prod.ini
├── group_vars/
│   ├── all.yml
│   └── web.yml
├── playbooks/
│   └── deploy.yml
├── roles/
│   ├── common/
│   │   └── tasks/main.yml
│   └── webapp/
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── templates/nginx.conf.j2
│       ├── files/
│       ├── defaults/main.yml
│       └── vars/main.yml
└── requirements.yml
```

## 4. Inventory

`inventory/dev.ini`

```ini
[web]
web1 ansible_host=10.0.1.10
web2 ansible_host=10.0.1.11

[app]
app1 ansible_host=10.0.2.10

[db]
db1 ansible_host=10.0.3.10

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/dev-key.pem
```

Useful commands:

```bash
ansible-inventory -i inventory/dev.ini --graph
ansible all -i inventory/dev.ini -m ping
ansible web -i inventory/dev.ini -m command -a "uptime"
```

For cloud environments, dynamic inventory can discover instances from AWS, Azure, GCP, Kubernetes, etc.

## 5. Playbook anatomy

`playbooks/deploy.yml`

```yaml
---
- name: Configure and deploy the web application
  hosts: web
  become: true
  gather_facts: true

  vars:
    app_name: demo-app
    app_port: 8080

  roles:
    - common
    - webapp
```

- `hosts`: selected inventory group.
- `become: true`: use privilege escalation, typically `sudo`.
- `gather_facts`: collect host data such as OS, IP, memory.
- `roles`: reusable automation units.

Run it:

```bash
ansible-playbook -i inventory/dev.ini playbooks/deploy.yml
```

Dry run:

```bash
ansible-playbook -i inventory/dev.ini playbooks/deploy.yml --check --diff
```

## 6. Example production-style project: deploy a Python app behind Nginx

Goal:

1. Install OS packages.
2. Create a secure application user.
3. Deploy application artifacts.
4. Create a systemd service.
5. Configure Nginx.
6. Restart services only when configs or code change.

### Common role

`roles/common/tasks/main.yml`

```yaml
---
- name: Update package cache
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600

- name: Install common packages
  ansible.builtin.apt:
    name:
      - curl
      - git
      - unzip
      - python3
    state: present

- name: Create application user
  ansible.builtin.user:
    name: appuser
    system: true
    shell: /usr/sbin/nologin
    create_home: false
```

### Application role

`roles/webapp/defaults/main.yml`

```yaml
---
app_name: demo-app
app_dir: /opt/demo-app
app_port: 8080
app_version: "1.0.0"
```

`roles/webapp/tasks/main.yml`

```yaml
---
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: present
    update_cache: true

- name: Create application directory
  ansible.builtin.file:
    path: "{{ app_dir }}"
    state: directory
    owner: appuser
    group: appuser
    mode: "0755"

- name: Copy application files
  ansible.builtin.copy:
    src: app/
    dest: "{{ app_dir }}/"
    owner: appuser
    group: appuser
    mode: "0644"
  notify: Restart application

- name: Install systemd service
  ansible.builtin.template:
    src: demo-app.service.j2
    dest: "/etc/systemd/system/{{ app_name }}.service"
    mode: "0644"
  notify:
    - Reload systemd
    - Restart application

- name: Configure Nginx virtual host
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: "/etc/nginx/sites-available/{{ app_name }}"
    mode: "0644"
  notify: Restart Nginx

- name: Enable Nginx site
  ansible.builtin.file:
    src: "/etc/nginx/sites-available/{{ app_name }}"
    dest: "/etc/nginx/sites-enabled/{{ app_name }}"
    state: link
  notify: Restart Nginx

- name: Ensure application is running
  ansible.builtin.systemd:
    name: "{{ app_name }}"
    state: started
    enabled: true

- name: Ensure Nginx is running
  ansible.builtin.systemd:
    name: nginx
    state: started
    enabled: true
```

`roles/webapp/handlers/main.yml`

```yaml
---
- name: Reload systemd
  ansible.builtin.systemd:
    daemon_reload: true

- name: Restart application
  ansible.builtin.systemd:
    name: "{{ app_name }}"
    state: restarted

- name: Restart Nginx
  ansible.builtin.systemd:
    name: nginx
    state: restarted
```

`roles/webapp/templates/nginx.conf.j2`

```nginx
server {
    listen 80;
    server_name {{ domain_name }};

    location / {
        proxy_pass http://127.0.0.1:{{ app_port }};
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Why this is good Ansible:

- Uses specific modules rather than shell commands.
- Is idempotent: packages are present, directories exist, services run.
- Restarts services only if a related file changes.
- Separates defaults, tasks, templates, and handlers.
- Can use different inventory and variables for dev/stage/prod.

## 7. Variables and precedence

Variables can come from many places:

```yaml
# group_vars/web.yml
app_port: 8080
domain_name: app.example.com
```

```yaml
# host_vars/web1.yml
app_port: 8081
```

```bash
ansible-playbook -e "app_version=1.2.0" playbooks/deploy.yml
```

A practical precedence order to remember:

1. Role defaults — weakest; safe defaults.
2. Inventory/group/host variables.
3. Playbook variables.
4. Role vars — stronger; use sparingly.
5. Extra vars (`-e`) — strongest.

Interview answer:

> I use role defaults for configurable defaults, group variables for environment-level configuration, host variables only for host-specific exceptions, and Vault for secrets. I avoid relying on complex precedence because it makes automation hard to debug.

## 8. Facts and conditionals

Facts are automatically gathered host information.

```yaml
- name: Install package on Debian
  ansible.builtin.apt:
    name: nginx
    state: present
  when: ansible_facts["os_family"] == "Debian"
```

```yaml
- name: Install package on RedHat
  ansible.builtin.dnf:
    name: nginx
    state: present
  when: ansible_facts["os_family"] == "RedHat"
```

Useful fact:

```yaml
{{ ansible_facts["default_ipv4"]["address"] }}
```

## 9. Loops, blocks, register, and retries

```yaml
- name: Install packages
  ansible.builtin.apt:
    name: "{{ item }}"
    state: present
  loop:
    - nginx
    - curl
    - git
```

```yaml
- name: Check application health
  ansible.builtin.uri:
    url: "http://localhost:{{ app_port }}/health"
    status_code: 200
  register: health_check
  retries: 5
  delay: 10
  until: health_check.status == 200
```

```yaml
- name: Deployment block
  block:
    - name: Deploy release
      ansible.builtin.copy:
        src: app/
        dest: /opt/demo-app/

  rescue:
    - name: Restore previous release
      ansible.builtin.debug:
        msg: "Rollback would run here"

  always:
    - name: Record deployment completion
      ansible.builtin.debug:
        msg: "Deployment attempt finished"
```

## 10. Templates vs files

- `copy`: copies a static file exactly as-is.
- `template`: renders Jinja2 variables and logic first.
- `file`: manages directories, permissions, symlinks, or files.
- `lineinfile`: changes one line; useful but avoid turning it into config-file management.
- `blockinfile`: manages a marked block in an existing file.

## 11. Secrets with Ansible Vault

Create encrypted values:

```bash
ansible-vault create group_vars/prod/vault.yml
ansible-vault edit group_vars/prod/vault.yml
ansible-vault encrypt secrets.yml
```

Example encrypted file before encryption:

```yaml
db_password: "super-secret-password"
```

Use it:

```bash
ansible-playbook -i inventory/prod.ini playbooks/deploy.yml --ask-vault-pass
```

Better CI/CD pattern: provide the vault password through a secure secret manager or protected CI variable, never commit plaintext secrets.

## 12. Tags, limits, and safer production runs

```yaml
- name: Install Nginx
  ansible.builtin.apt:
    name: nginx
    state: present
  tags: nginx
```

```bash
ansible-playbook deploy.yml --tags nginx
ansible-playbook deploy.yml --skip-tags database
ansible-playbook deploy.yml --limit web1
ansible-playbook deploy.yml --check --diff
```

For rolling deployment:

```yaml
- name: Rolling deployment
  hosts: web
  serial: 1
  max_fail_percentage: 20
  become: true
  roles:
    - webapp
```

This updates one server at a time, protecting availability.

## 13. Error handling and reliability

Useful options:

```yaml
any_errors_fatal: true
max_fail_percentage: 20
```

Common patterns:

- Validate configuration before restart:

```yaml
- name: Validate Nginx config
  ansible.builtin.command: nginx -t
  changed_when: false
```

- Use `failed_when` and `changed_when` for commands.
- Prefer modules over `shell` or `command`.
- Use `serial` for production rollouts.
- Add health checks after deployment.
- Test first with `--check`, `--diff`, `--limit`.
- Keep tasks small and meaningful.

## 14. `command` vs `shell`

`command`:

```yaml
ansible.builtin.command: nginx -t
```

- Does not process pipes, redirects, variables, wildcard expansion.
- Safer and preferred.

`shell`:

```yaml
ansible.builtin.shell: "ps aux | grep demo-app"
```

- Runs through a shell.
- Needed for pipes/redirection, but riskier and less predictable.
- Use only when necessary.

Interview answer:

> I default to purpose-built modules, then `command`, and use `shell` only when shell behavior is genuinely required. When using commands, I explicitly define `changed_when` and `failed_when` to preserve idempotency and accurate reporting.

## 15. Roles, collections, and Galaxy

Install third-party content:

`requirements.yml`

```yaml
---
collections:
  - name: community.general
  - name: ansible.posix

roles:
  - name: geerlingguy.nginx
```

```bash
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install -r requirements.yml
```

Use fully qualified collection names in production:

```yaml
ansible.builtin.apt:
community.general.ufw:
amazon.aws.ec2_instance:
kubernetes.core.k8s:
```

This prevents ambiguity about where a module comes from.

## 16. Ansible configuration

`ansible.cfg`

```ini
[defaults]
inventory = inventory/dev.ini
roles_path = roles
host_key_checking = True
stdout_callback = yaml
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_ask_pass = False
```

Avoid disabling host-key checking for real environments.

## 17. Testing and code quality

A mature Ansible workflow includes:

```bash
ansible-lint
yamllint .
ansible-playbook --syntax-check playbooks/deploy.yml
ansible-playbook --check --diff playbooks/deploy.yml
```

For role testing, commonly use:

- Molecule: test Ansible roles in disposable environments.
- Docker, VMs, or cloud test accounts: integration testing.
- CI pipelines: lint, syntax check, test inventory, controlled deployment.

## 18. Ansible vs Terraform vs Kubernetes

| Tool | Best use |
|---|---|
| Terraform | Create infrastructure: VPCs, EC2/VMs, load balancers, databases, IAM |
| Ansible | Configure machines and deploy applications |
| Kubernetes | Run and manage containerized workloads |
| Helm | Package and deploy Kubernetes applications |
| Jenkins/GitHub Actions/GitLab CI | Trigger pipelines; may call Terraform and Ansible |

Strong answer:

> Terraform creates the infrastructure and tracks its desired resource state. Ansible configures those provisioned hosts and deploys applications. In Kubernetes environments, I generally use Kubernetes manifests or Helm for workloads, and Ansible for supporting infrastructure, bootstrapping, or tasks outside the cluster.

## 19. Common interview questions

### What is idempotency?

Running the same automation multiple times produces the same desired state without unnecessary changes.

Example: `state: present` installs Nginx only if it is missing.

### What is the difference between a playbook and a role?

A playbook orchestrates work across hosts. A role packages reusable automation in a standard directory structure.

### What are handlers?

Tasks triggered only when another task reports a change. They are commonly used to restart or reload services after config changes.

### How does Ansible connect to Linux servers?

Usually SSH. It is agentless and typically requires SSH access plus Python on the managed node.

### What does `become: true` do?

It enables privilege escalation, usually equivalent to using `sudo`.

### How do you manage different environments?

Separate inventories and environment variables, for example `inventory/dev`, `inventory/stage`, and `inventory/prod`; shared roles; secrets encrypted with Vault.

### How do you handle secrets?

Ansible Vault encrypts files/variables at rest. In CI/CD, vault credentials should come from protected secret storage, not source control.

### How do you deploy with no downtime?

Use load-balancer draining plus `serial` rolling deployment, health checks, validation before restart, and rollback-capable release artifacts.

### How do you troubleshoot a failed playbook?

```bash
ansible-playbook -vvv deploy.yml
ansible-inventory --graph
ansible all -m ping
ansible-playbook --syntax-check deploy.yml
```

Then inspect the failing host, task output, resolved variables, permissions, and service logs.

### What is dynamic inventory?

An inventory generated from a source of truth such as AWS, Azure, GCP, Kubernetes, CMDB, or an API, rather than manually maintained host lists.

### Why avoid `shell`?

It is less secure, less portable, harder to make idempotent, and can conceal failures. Prefer a dedicated module or `command`.

## 20. A polished interview project explanation

> I built an Ansible project to deploy a Python web application across development, staging, and production environments. The project used separate inventories, group variables, reusable roles for common OS setup and web application deployment, Jinja2 templates for Nginx and systemd configurations, and handlers to restart services only after relevant changes.  
>
> I used Ansible Vault for environment secrets and added linting, syntax checks, dry-run validation, and health-check tasks in CI. For production, the playbook used serial execution so hosts were updated gradually, validated through the load balancer, and failures were contained. Terraform provisioned cloud infrastructure, while Ansible configured the VMs and deployed the application.

## 21. Practical command cheat sheet

```bash
# Test connectivity
ansible all -i inventory/dev.ini -m ping

# Run a command
ansible web -i inventory/dev.ini -m command -a "uptime"

# Execute playbook
ansible-playbook -i inventory/dev.ini playbooks/deploy.yml

# Check syntax
ansible-playbook --syntax-check playbooks/deploy.yml

# Dry run and show changes
ansible-playbook -i inventory/dev.ini playbooks/deploy.yml --check --diff

# Deploy only one host
ansible-playbook -i inventory/prod.ini playbooks/deploy.yml --limit web1

# Run only tagged tasks
ansible-playbook playbooks/deploy.yml --tags nginx

# Debug verbose output
ansible-playbook -vvv playbooks/deploy.yml

# Encrypt a secret file
ansible-vault encrypt group_vars/prod/vault.yml
```

For interviews, master these especially: inventory, playbooks, modules, idempotency, roles, variables, templates, handlers, Vault, error handling, dynamic inventory, rolling deployment, and Ansible-vs-Terraform.