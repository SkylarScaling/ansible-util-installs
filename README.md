# ansible-util-installs

Ansible playbooks and roles for installing standalone tools and utilities used to support OpenShift/ACM lab and testing environments. Each utility is a self-contained playbook backed by one or more roles under `roles/<utility>/`.

## Structure

```
playbooks/
  ansible.cfg              # roles_path, callbacks, SSH keep-alive
  install-<utility>.yaml   # one playbook per utility
  group_vars/all.yaml      # shared defaults
roles/
  <utility>/
    install_<utility>/     # installation role
    configure_<utility>_*/  # optional configuration roles
```

> **All commands must be run from the `playbooks/` directory.** `ansible.cfg` lives there and sets `roles_path`.

---

## Utilities

| Playbook | What it installs |
|---|---|
| `install-ldap-server.yaml` | 389 Directory Server (Red Hat LDAP) + OCP test users and groups |

---

## 389 Directory Server

Installs a 389 DS LDAP server as a Podman container on the jumphost, then creates an LDAP service account, OCP test groups, and test users. At the end it prints the exact `ldap:`, `rbac_bindings:`, and `groupsync:` blocks to paste into your OCP automation inventory.

389 DS is the LDAP engine that FreeIPA uses internally. For OCP LDAP testing, it is the right tool: no Certificate Authority, no Kerberos, no Java — it starts in seconds and runs reliably in containers.

### Why 389 DS instead of FreeIPA

FreeIPA's CA (Dogtag PKI) is Java-based and fails in constrained container environments. For OCP LDAP IDP and GroupSync testing, only the Directory Server component is needed. 389 DS provides exactly that.

### Prerequisites

- Jumphost already provisioned (run `provision-jumphost.yaml` first)
- SSH access from your local machine to the jumphost

### Workflow

```
1. ansible-playbook provision-jumphost.yaml -i ~/inventories/disconnected-aws-inventory
2. ansible-playbook install-ldap-server.yaml -i ~/inventories/ldap-inventory.yaml
3. (add printed snippets to OCP inventory)
4. ansible-playbook hub-spoke-disconnected-setup.yaml ...  (from jumphost)
```

### Run

```bash
cd playbooks
ansible-playbook install-ldap-server.yaml -i ~/inventories/ldap-inventory.yaml
```

Install server only (skip test data):
```bash
ansible-playbook install-ldap-server.yaml -i ~/inventories/ldap-inventory.yaml --skip-tags testdata
```

Create/refresh test data only (server already running):
```bash
ansible-playbook install-ldap-server.yaml -i ~/inventories/ldap-inventory.yaml --tags testdata
```

---

### Example Inventory

The jumphost public DNS name comes from the `provision-jumphost.yaml` summary output.

```yaml
all:
  vars:
    # -----------------------------------------------------------------------
    # Update these two values when you provision a new test environment.
    # Everything else is derived from them.
    # -----------------------------------------------------------------------
    sandbox_domain: "sandbox2915.opentlc.com"                               # changes each environment
    jumphost_public_dns: "ec2-<public-ip>.us-east-2.compute.amazonaws.com"  # from provision-jumphost output

  children:
    ldap_server:
      hosts:
        jumphost:
          ansible_host: "{{ jumphost_public_dns }}"
          ansible_user: "ec2-user"
          ansible_ssh_private_key_file: "~/.ssh/id_ed25519"
          ansible_ssh_extra_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"

          # 389 DS server settings
          ds389:
            container_name: "389ds"
            image: "quay.io/389ds/dirsrv"
            data_dir: "/opt/389ds/data"
            root_dn: "cn=Directory Manager"
            root_password: "<dm_password>"
            suffix: "dc={{ sandbox_domain.split('.') | join(',dc=') }}"

          # LDAP service account — OCP uses this for bind operations
          ds389_ldap_service_account:
            username: "ldap-svc"
            cn: "LDAP Service Account"
            sn: "ServiceAccount"
            password: "<service_account_password>"

          # OCP groups to create
          ds389_ocp_groups:
            - openshift-admins
            - openshift-developers
            - openshift-viewers

          # OCP test users and their group memberships
          ds389_ocp_users:
            - username: ocp-admin
              cn: "OCP Admin"
              sn: "Admin"
              password: "<user_password>"
              groups:
                - openshift-admins
            - username: ocp-dev
              cn: "OCP Developer"
              sn: "Developer"
              password: "<user_password>"
              groups:
                - openshift-developers
            - username: ocp-viewer
              cn: "OCP Viewer"
              sn: "Viewer"
              password: "<user_password>"
              groups:
                - openshift-viewers
```

---

### OCP Inventory Update (printed at end of run)

The playbook prints the **one value** that needs to be set per environment. Everything else is pre-populated in the OCP inventory using Jinja2 templates driven by `base_domain`.

```
jumphost_private_ip: "10.0.0.x"   ← add/update this in disconnected-aws-inventory
```

**Pre-populate these blocks once in your OCP inventory** — they never change between environments because `base_domain` and `jumphost_private_ip` drive all the derived values:

```yaml
# In disconnected-aws-inventory all.vars:

jumphost_private_ip: ""   # ← fill in after running install-ldap-server.yaml

ntp_servers:
  - 169.254.169.123
  - 0.rhel.pool.ntp.org

ldap:
  name: "389ds"
  url: "ldap://{{ jumphost_private_ip }}:389/ou=people,dc={{ base_domain.split('.') | join(',dc=') }}?uid"
  bind_dn: "uid=ldap-svc,ou=people,dc={{ base_domain.split('.') | join(',dc=') }}"
  bind_password: "<service_account_password>"
  insecure: true
  ca_cert: ""
  attributes:
    id: ["dn"]
    email: ["mail"]
    name: ["cn"]
    preferred_username: ["uid"]

rbac_bindings:
  - group: "openshift-admins"
    cluster_role: "cluster-admin"
  - group: "openshift-developers"
    cluster_role: "edit"
  - group: "openshift-viewers"
    cluster_role: "view"

groupsync:
  schedule: "0 * * * *"
  ldap_url: "ldap://{{ jumphost_private_ip }}:389"
  bind_dn: "uid=ldap-svc,ou=people,dc={{ base_domain.split('.') | join(',dc=') }}"
  bind_password: "<service_account_password>"
  ca_cert: ""
  insecure: true
  groups_base_dn: "ou=groups,dc={{ base_domain.split('.') | join(',dc=') }}"
  groups_filter: "(&(objectClass=groupOfNames)(cn=openshift-*))"
  users_base_dn: "ou=people,dc={{ base_domain.split('.') | join(',dc=') }}"
  group_uid_attribute: "dn"
  group_name_attributes: ["cn"]
  group_membership_attributes: ["member"]
  user_uid_attribute: "dn"
  user_name_attributes: ["uid"]
  tolerate_member_not_found: true
  tolerate_member_out_of_scope: true
```

> **Note:** The OCP disconnected inventory uses `base_domain` (not `sandbox_domain`). The templates above match that convention.

---

### Inventory Variable Reference

**`ds389` dict (per-host):**

| Variable | Required | Default | Description |
|---|---|---|---|
| `ds389.container_name` | No | `389ds` | Podman container name |
| `ds389.image` | No | `quay.io/389ds/dirsrv` | Container image |
| `ds389.data_dir` | No | `/opt/389ds/data` | Host path for persistent data |
| `ds389.root_dn` | No | `cn=Directory Manager` | Directory Manager bind DN |
| `ds389.root_password` | Yes | — | Directory Manager password |
| `ds389.suffix` | Yes | — | LDAP suffix (e.g. `dc=example,dc=com`) |

**`ds389_ldap_service_account` dict:** `username`, `cn`, `sn`, `password` — the bind account OCP uses.

**`ds389_ocp_groups` list:** Group names to create. Any group matching `cn=openshift-*` is included by the default GroupSync filter.

**`ds389_ocp_users` list:** Each entry has `username`, `cn`, `sn`, `password`, and `groups` (list of group names).
