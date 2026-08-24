# Zabbix 7.0 LTS Server Deployment Playbook  
**Target:** RHEL 10.2 (standalone)  
**Control plane:** Ansible Automation Platform 2.7 (containerized Execution Environments)  
**Database:** PostgreSQL 16 + optional TimescaleDB  
**Web frontend:** Nginx + PHP-FPM (HTTPS)  
**SELinux:** Enforcing (policy package installed, never disabled)

---

## 1. AAP Custom Credential Types

Create the following three Custom Credential Types in AAP before launching the job.

### 1.1 Zabbix Database

**Input Configuration**
```yaml
fields:
  - id: zabbix_db_password
    type: string
    label: Zabbix Database Password
    secret: true
required:
  - zabbix_db_password
```

**Injector Configuration**
```yaml
extra_vars:
  zabbix_db_password: '{{ zabbix_db_password }}'
```

### 1.2 Zabbix Admin

**Input Configuration**
```yaml
fields:
  - id: zabbix_admin_password
    type: string
    label: Zabbix Admin (Web UI) Password
    secret: true
required:
  - zabbix_admin_password
```

**Injector Configuration**
```yaml
extra_vars:
  zabbix_admin_password: '{{ zabbix_admin_password }}'
```

### 1.3 TLS Certificate

**Input Configuration**
```yaml
fields:
  - id: zabbix_tls_cert
    type: string
    label: TLS Certificate (PEM)
    multiline: true
  - id: zabbix_tls_key
    type: string
    label: TLS Private Key (PEM)
    secret: true
    multiline: true
required:
  - zabbix_tls_cert
  - zabbix_tls_key
```

**Injector Configuration**
```yaml
extra_vars:
  zabbix_tls_cert: '{{ zabbix_tls_cert }}'
  zabbix_tls_key: '{{ zabbix_tls_key }}'
```

---

## 2. Variables the Operator Must Set

| Variable | Location | Description | Example |
|----------|----------|-------------|---------|
| `ansible_host` | inventory/hosts.yml | IP or resolvable name of the RHEL 10.2 host | `10.0.0.50` |
| `ansible_user` | inventory/hosts.yml | SSH user with sudo rights | `ansible` |
| `zabbix_server_fqdn` | group_vars/zabbix_servers.yml | FQDN used in Nginx server_name and certificate | `zabbix01.example.com` |
| `zabbix_server_timezone` | group_vars | System + PHP timezone | `UTC` or `Europe/Berlin` |
| `zabbix_timescaledb_enabled` | group_vars | Enable TimescaleDB extension | `true` / `false` |
| `zabbix_smtp_*` | group_vars | SMTP settings for the Email media type | see defaults |
| `zabbix_backup_retention_days` | group_vars | Days to keep pg_dump backups | `7` |
| Secrets | AAP Credentials | `zabbix_db_password`, `zabbix_admin_password`, `zabbix_tls_cert`, `zabbix_tls_key` | injected at runtime |

All other values have production-ready defaults inside the role `defaults/main.yml` files.

---

## 3. Prerequisite Media / Repository Staging

1. RHEL 10.2 host is already registered with RHSM and has **BaseOS** + **AppStream** enabled.
2. Outbound HTTPS access to:
   - `https://repo.zabbix.com` (Zabbix 7.0 LTS release RPM and packages)
   - (optional) TimescaleDB repository if the package is not pulled from the Zabbix repo.
3. A valid TLS certificate + private key whose CN/SAN matches `zabbix_server_fqdn`.
4. AAP 2.7 controller with the ability to build Execution Environments from `execution-environment.yml`.

---

## 4. Operator Guide – Importing into AAP 2.7

### 4.1 Build the Execution Environment

```bash
# On a machine with ansible-builder and access to registry.redhat.io
ansible-builder build -f execution-environment.yml -t registry.example.com/ee-zabbix-rhel9:1.0
podman push registry.example.com/ee-zabbix-rhel9:1.0
```

In AAP → Execution Environments → Add → point to the image above.

### 4.2 Create the Project

1. AAP → Projects → Add.
2. Source Control Type = Git (or Manual).
3. Point to the repository that contains this playbook tree.
4. SCM Branch = main (or the branch you use).

### 4.3 Create Inventory

1. AAP → Inventories → Add.
2. Add the RHEL 10.2 host (or sync from a source).
3. Place the host in the group `zabbix_servers`.
4. Optionally upload `inventory/group_vars/zabbix_servers.yml` as a group variable file or paste the values.

### 4.4 Create Credentials

1. Create the three Custom Credential Types shown in section 1.
2. Create one Credential of each type and fill in the real secrets.
3. Also create a standard **Machine** credential (SSH key or password) for the target host.

### 4.5 Create the Job Template

1. AAP → Templates → Add → Job Template.
2. Inventory = the inventory from 4.3.
3. Project = the project from 4.2.
4. Playbook = `site.yml`.
5. Execution Environment = the EE built in 4.1.
6. Credentials = Machine + Zabbix Database + Zabbix Admin + TLS Certificate.
7. Enable **Privilege Escalation**.
8. (Optional) set tags if you want to run only a subset of roles.

### 4.6 Launch

Click **Launch**. The playbook is fully idempotent; subsequent runs are safe.

---

## 5. Deployment Sequence

| Order | Role | Tag | What it does |
|-------|------|-----|--------------|
| 1 | prereqs | `prereqs` | Ping, assert RHSM, install Zabbix 7.0 release RPM, baseline packages, timezone, firewalld ports, SELinux policy package (`zabbix-selinux-policy-main`), SELinux booleans |
| 2 | postgresql | `postgresql` | Install PostgreSQL 16, initdb, template pg_hba.conf + postgresql.conf, create user/DB, optional TimescaleDB extension |
| 3 | zabbix_server | `zabbix_server` | `community.zabbix.zabbix_server` role, start service, optional TimescaleDB schema migration |
| 4 | zabbix_web | `zabbix_web` | `community.zabbix.zabbix_web` + custom Nginx server block, TLS material, PHP-FPM tuning |
| 5 | zabbix_agent | `zabbix_agent` | `community.zabbix.zabbix_agent` (Agent 2) for self-monitoring |
| 6 | post_config | `postconfig` | Wait for API, change default Admin password, register host, configure Email media type, daily pg_dump cron |

---

## 6. Design Notes & Guarantees

- **Official collections first** – `community.zabbix`, `community.postgresql`, `ansible.posix` are used wherever they cover the task.
- **Everything as code** – all configuration files are Jinja2 templates.
- **Idempotent** – package modules, `community.postgresql_*`, `creates`/`when` guards, and API existence checks make re-runs safe.
- **Secrets never on disk in the repo** – only AAP Custom Credential Types inject them at runtime; sensitive tasks use `no_log: true`.
- **SELinux stays enforcing** – the playbook installs `zabbix-selinux-policy-main` and removes the conflicting `selinux-policy-targeted-extra` if present.
- **No Apache** – Nginx is the sole web server.
- **No MySQL** – PostgreSQL 16 only.
- **API modules** that require `httpapi` are always `delegate_to: localhost` so they never fight with the SSH connection used for the rest of the play.

---

## 7. Post-deployment Verification

```bash
# On the Zabbix server
systemctl status zabbix-server zabbix-agent2 nginx php-fpm postgresql
ss -tlnp | grep -E '10051|10050|443'
curl -k https://$(hostname -f)/api_jsonrpc.php   # should return JSON-RPC error (expected without auth)
sudo -u zabbix psql -d zabbix_server -c '\dx'    # timescaledb should appear when enabled
```

Log in to the web UI with the Admin password you supplied via the credential. The server should already appear under **Monitoring → Hosts** linked to the *Zabbix server health* template.
