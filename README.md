This repository automates provisioning and day‑2 maintenance for edge hosts based on Raspberry Pi and Ubuntu. It focuses on preparing devices for IoT workloads and lightweight services with idempotent playbooks, a modest security baseline, and repeatable execution across heterogeneous hardware.

The approach is pragmatic: small, single‑purpose playbooks that can be composed per site or group. Targets include Ubuntu 22.04 and 24.04 (both Raspberry Pi and x86_64) and Raspbian Buster. Core workflows cover SSH key distribution, patching with optional reboots, DNS adjustments, optional k3s prerequisites, and a basic Node‑RED installation as a service. Everything is designed to be safe to preview with Ansible check mode and easy to track in Git or GitLab.

The result is a predictable edge baseline that reduces manual effort. New devices reach a consistent state in minutes; existing devices receive controlled changes with a clear audit trail. Teams can roll out fixes and features with confidence, knowing each step is idempotent and reversible.

**Overview**
- Purpose: provision and maintain Raspberry Pi and Ubuntu edge hosts for IoT and lightweight services.
- Supported targets: Ubuntu 22.04 and 24.04 (Raspberry Pi and x86_64) and Raspbian Buster.
- Minimum Ansible: `ansible-core >= 2.15` on the control node.
- Optional components: k3s prerequisites and k3s agent join; Node‑RED 4.0 as a system service on Node.js 22.
- Safe previews: every playbook supports `--check` dry‑runs to evaluate impact.

**Approach and Architecture**
- Inventory-driven execution with host groups such as `edge`, `rpi_ubuntu`, and `rpi_com` to target specific fleets.
- Small, focused playbooks under `playbooks/` that can be run independently or chained.
- Variables can be organized per group using `group_vars/` (see `group_vars.example/` for typical patterns) and per host in your inventory.
- Secrets (e.g., SSH passwords or API tokens) should be handled via environment variables and/or Ansible Vault. Keep `.env` files unversioned and store only non‑sensitive defaults in `*.example`.
- No external Ansible collections are strictly required; built‑in modules are used throughout. If your environment uses collection modules, declare them explicitly in your own `collections/requirements.yml`.

Versions reference
- Ubuntu 22.04, 24.04; Raspbian Buster
- Node‑RED 4.0 on Node.js 22 (if installing Node‑RED)

**Whats Included**
- `playbooks/copy_ssh_keys.yml` — Distributes a local public key (defaults to `/home/user/.ssh/id_rsa.pub`) to target users using `authorized_key`.
- `playbooks/update_upgrade.yml` — Applies apt update/upgrade/autoremove with module fallbacks; useful for regular patching windows.
- `playbooks/update_nameserver.yml` — Updates DNS on Ubuntu (replaces entries in `/etc/netplan/50-cloud-init.yaml` then `netplan apply`) and on Raspbian (`/etc/dhcpcd.conf` then service restart). Placeholders reflect example addresses.
- `playbooks/prep_k3s_iptables.yml` — Prepares networking for k3s by flushing rules and switching `iptables`/`ip6tables` to legacy alternatives; triggers a reboot.
- `playbooks/node_k3s_install.yml` — Optional k3s install/join via upstream script; expects `K3S_TOKEN` and `K3S_URL` supplied at runtime.
- `playbooks/install_node-red.yml` — Currently performs apt update/upgrade/autoremove. Treat as a scaffold to extend with Node‑RED 4.0 on Node.js 22 service setup.
- `playbooks/deploy_tmserveragent.yml` — Copies and extracts a provided tarball under `/tmp/TM/` and runs `./tmxbc install`.
- Utilities: `playbooks/prep_boot_rbs.yml` (adds cgroup params to `/boot/cmdline.txt` and `arm_64bit=1` in `/boot/config.txt`), `playbooks/reboot.yml` (controlled reboot), `playbooks/ssh_to_hosts.yml` (ad‑hoc SSH using `sshpass`, delegated to localhost).
- `inventory/` — Inventory groups and hosts (e.g., `rpi_ubuntu`, `rpi_com`, etc.). Align exclusion markers like `!otsrvcibernetica` vs `!otservcibernetica` to your environment.

Repository layout (selected)
- `inventory/` — Your per‑environment host lists and group definitions (e.g., `inventory.ini`, `inventory.example`).
- `playbooks/` — Task‑specific playbooks designed to be run in isolation.
- `group_vars.example/` — Illustrative per‑group variables structure (e.g., resolver addresses, Node‑RED service options). Copy to `group_vars/` and adapt.

**Setup**
Prerequisites
- Control node with `ansible-core >= 2.15` and Python 3.
- SSH access to targets (keys preferred); privilege escalation (`become`) enabled for administrative tasks.
- Optional: environment variables for sensitive values (e.g., `ANSIBLE_SSH_PASS`, `ANSIBLE_SUDO_PASS`, `K3S_TOKEN`).

Sample inventory (`inventory.example`)
```
[edge]
edge01 ansible_host=10.0.10.10 ansible_user=pi
edge02 ansible_host=10.0.10.11 ansible_user=cib

[rpi_ubuntu]
10.0.7.204 ansible_user=com
10.0.7.205 ansible_user=com

[rpi_com]
10.0.13.228 ansible_user=pi
10.0.13.242 ansible_user=pi

[all:vars]
ansible_ssh_pass={{ ANSIBLE_SSH_PASS }}
ansible_sudo_pass={{ ANSIBLE_SUDO_PASS }}
```

Common commands
- Dry‑run a patch window on Ubuntu Raspberry Pis:
  `ansible-playbook -i inventory.example playbooks/update_upgrade.yml -l rpi_ubuntu --check`
- Apply DNS changes to all edge hosts:
  `ansible-playbook -i inventory.example playbooks/update_nameserver.yml -l edge`
- Prepare iptables legacy mode before joining k3s:
  `ansible-playbook -i inventory.example playbooks/prep_k3s_iptables.yml -l rpi_ubuntu --check`
- Install Node‑RED service on selected hosts:
  `ansible-playbook -i inventory.example playbooks/install_node-red.yml -l rpi_com`

Notes
- Reboots and iptables changes can disrupt connectivity. Always start with `--check`, and consider limiting scope with `-l <group>`.
- The k3s installer requires `K3S_TOKEN` and `K3S_URL`. Provide them via environment variables or Vault; do not hard‑code secrets.
 - The `ssh_to_hosts.yml` helper invokes SSH via `sshpass` and a local key path; use only in controlled environments.

**Configuration and Security**
- Variables: keep sensitive values in environment variables or Ansible Vault. Use `*.example` files to share non‑sensitive defaults.
- TLS: prefer TLS for API calls and control planes (e.g., k3s server URL). Validate certificates in production.
- Least privilege: playbooks request `become` only where necessary; run with a non‑root SSH user that can escalate.
- Idempotence: built‑in modules are used where possible; shell commands are limited to cases without a suitable module.
- Version control: manage changes with Git/GitLab for traceability and rollbacks.

**Limitations and Next Steps**
- The Node‑RED installation playbook provides a baseline; further hardening (e.g., systemd overrides, resource limits) can be added per site.
- The k3s installation uses the upstream convenience script; organizations may prefer pinned artifacts and offline mirrors.
- Current iptables handling switches to legacy mode; migrating to nftables‑compatible setups is a sensible next step on newer distributions.
- Future improvements: convert playbooks into roles, add preflight validations, and integrate health checks post‑change.

**License**
- MIT License. See the repository’s LICENSE file if present; otherwise, the project is offered under MIT terms.
- Contact: see profile for LinkedIn or email.
