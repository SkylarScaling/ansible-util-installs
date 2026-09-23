# ansible-util-installs

Ansible playbooks and roles for installing standalone tools and utilities used to support OpenShift/ACM lab and testing environments. Each utility is a self-contained playbook backed by one or more roles under `roles/<utility>/`.

## Structure

```
playbooks/
  ansible.cfg              # roles_path, callbacks
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
| `install-freeipa.yaml` | FreeIPA LDAP/Kerberos server (Podman container) + OCP test users and groups |

---

## FreeIPA

Installs a FreeIPA server as a Podman container on a target host, then creates an LDAP service account, OCP test groups, and test users. At the end, it prints the exact `ldap:`, `rbac_bindings:`, and `groupsync:` inventory snippets to paste into your OCP automation inventory.

### Prerequisites

- Target host running RHEL 8+, RHEL 9, or Fedora (must have `dnf`)
- `podman` installed (the role installs it if missing)
- SSH access from your local machine to the jumphost (key at `~/.ssh/id_ed25519`)
- At least **2 GB free RAM** on the target host
- Ports 389, 636, 80, 443, 88, 464 free on the target host

### Workflow

Run from your **local machine** after the jumphost has been provisioned and configured. The playbook SSHes to the jumphost and installs FreeIPA there — the same SSH pattern used by `provision-jumphost.yaml` Play 2.

```
1. ansible-playbook provision-jumphost.yaml -i inventory.yaml   # creates jumphost
2. ansible-playbook install-freeipa.yaml -i inventory-freeipa.yaml  # installs FreeIPA on it
3. ansible-playbook hub-spoke-disconnected-setup.yaml ...       # deploy OCP (from jumphost)
```

### Run

```bash
cd playbooks
ansible-playbook install-freeipa.yaml -i inventory-freeipa.yaml
```

Install server only (skip test data):
```bash
ansible-playbook install-freeipa.yaml -i inventory-freeipa.yaml --tags freeipa
```

Create/refresh test data only (server already running):
```bash
ansible-playbook install-freeipa.yaml -i inventory-freeipa.yaml --tags testdata
```

---

### Example Inventory

The jumphost public DNS name comes from the output of `provision-jumphost.yaml` — it is printed in the final summary as `Public DNS`.

```yaml
all:
  children:
    freeipa_server:
      hosts:
        jumphost:
          ansible_host: "ec2-<public-ip>.<region>.compute.amazonaws.com"
          ansible_user: "ec2-user"
          ansible_ssh_private_key_file: "~/.ssh/id_ed25519"
          ansible_ssh_extra_args: "-o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null"

          # FreeIPA server settings
          freeipa:
            hostname: "ipa.sandbox2915.opentlc.com"   # must resolve to the host IP
            domain: "sandbox2915.opentlc.com"
            realm: "SANDBOX2915.OPENTLC.COM"           # uppercase domain
            admin_password: "RedHat123!"
            directory_manager_password: "RedHat123!"
            container_name: "freeipa"
            data_dir: "/opt/freeipa/data"
            image: "quay.io/freeipa/freeipa-server:fedora-41"

          # LDAP service account — OCP will use this for bind operations
          freeipa_ldap_service_account:
            username: "ldap-svc"
            first_name: "LDAP"
            last_name: "ServiceAccount"
            password: "ServicePassword123!"

          # OCP groups to create
          freeipa_ocp_groups:
            - openshift-admins
            - openshift-developers
            - openshift-viewers

          # OCP test users and their group memberships
          freeipa_ocp_users:
            - username: ocp-admin
              first_name: OCP
              last_name: Admin
              password: "Password123!"
              groups:
                - openshift-admins
            - username: ocp-dev
              first_name: OCP
              last_name: Developer
              password: "Password123!"
              groups:
                - openshift-developers
            - username: ocp-viewer
              first_name: OCP
              last_name: Viewer
              password: "Password123!"
              groups:
                - openshift-viewers
```

> **Note on hostname:** FreeIPA requires its `hostname` to resolve to the host's IP. The role writes a `/etc/hosts` entry automatically using the jumphost's primary interface IP, so the container initialises correctly. For the OCP `ldap.url` and `groupsync.ldap_url`, use the jumphost's **private IP or private DNS name** so cluster nodes can reach it within the VPC — the public DNS name is only reachable from outside AWS. The final summary prints both the LDAP URL and OCP snippet with the correct address.

---

### OCP Inventory Snippets (auto-printed at end of run)

After a successful run, the playbook prints the exact blocks to add to your OCP automation inventory. They look like this (values will reflect your actual IPs and domain):

```yaml
# In your OCP inventory all.vars:

ldap:
  name: "freeipa"
  url: "ldap://<freeipa-host-ip>:389/cn=users,cn=accounts,dc=sandbox2915,dc=opentlc,dc=com?uid"
  bind_dn: "uid=ldap-svc,cn=users,cn=accounts,dc=sandbox2915,dc=opentlc,dc=com"
  bind_password: "ServicePassword123!"
  insecure: false
  ca_cert: "{{ lookup('file', '~/freeipa-ca.crt') }}"   # written to ~/freeipa-ca.crt on the control node
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
  ldap_url: "ldap://<freeipa-host-ip>:389"
  bind_dn: "uid=ldap-svc,cn=users,cn=accounts,dc=sandbox2915,dc=opentlc,dc=com"
  bind_password: "ServicePassword123!"
  ca_cert: "{{ lookup('file', '~/freeipa-ca.crt') }}"
  groups_base_dn: "cn=groups,cn=accounts,dc=sandbox2915,dc=opentlc,dc=com"
  groups_filter: "(&(objectClass=ipausergroup)(cn=openshift-*))"
  users_base_dn: "cn=users,cn=accounts,dc=example,dc=com"
  group_uid_attribute: "dn"
  group_name_attributes: ["cn"]
  group_membership_attributes: ["member"]
  user_uid_attribute: "dn"
  user_name_attributes: ["uid"]
  tolerate_member_not_found: true
  tolerate_member_out_of_scope: true
```

The FreeIPA CA certificate is saved to `~/freeipa-ca.crt` on your control node so it can be read directly by the `lookup('file', ...)` calls above.

---

### Inventory Variable Reference

**`freeipa` dict (per-host):**

| Variable | Required | Default | Description |
|---|---|---|---|
| `freeipa.hostname` | Yes | — | FQDN for the IPA server — must resolve to the host IP |
| `freeipa.domain` | Yes | — | DNS domain (e.g. `sandbox2915.opentlc.com`) |
| `freeipa.realm` | Yes | — | Kerberos realm — uppercase domain by convention |
| `freeipa.admin_password` | Yes | — | IPA admin user password |
| `freeipa.directory_manager_password` | Yes | — | LDAP Directory Manager password |
| `freeipa.container_name` | No | `freeipa` | Podman container name |
| `freeipa.data_dir` | No | `/opt/freeipa/data` | Host path for persistent IPA data |
| `freeipa.image` | No | `quay.io/freeipa/freeipa-server:fedora-41` | Container image to use |

**`freeipa_ldap_service_account` dict:**

| Variable | Required | Default | Description |
|---|---|---|---|
| `username` | Yes | `ldap-svc` | IPA username for the bind account |
| `first_name` | No | `LDAP` | IPA user first name |
| `last_name` | No | `ServiceAccount` | IPA user last name |
| `password` | Yes | — | Bind account password (use ansible-vault) |

**`freeipa_ocp_groups` list:** Group names to create in IPA. Any group prefixed with `openshift-` is automatically included by the default `groups_filter` in the GroupSync config.

**`freeipa_ocp_users` list:** Each entry has `username`, `first_name`, `last_name`, `password`, and `groups` (list of group names to add the user to).
