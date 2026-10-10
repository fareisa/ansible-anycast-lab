# Ansible Anycast Lab

This Repo is just manifest from my ansible playbook that running configuration for my lab. If you want furthe information about my lab, you can got to this website 
Also, some explanation about my ansible is more detail in here (supposed to be) than my article.
Just info, this doc is mostly written by AI, im just too lazy to write docs.

## VM provisioning

The Ubuntu servers are provisioned separately with Terraform; this Ansible
project configures them after they exist. The server precheck runs first to
catch some hostname and address mismatches before configuration begins.

The [`9-vm-2gb/vm.yaml`](9-vm-2gb/vm.yaml) file is also available as a VM
definition template for
[Spawn-VM-banyak-di-Proxmox](https://github.com/fareisa/Spawn-VM-banyak-di-Proxmox).
If using that generator instead, keep its VM names and IP addresses aligned
with `inventory.ini` and `group_vars/`.

The CI configuration is currently being worked on. I just want to try Ci/Cd pipeline, but i think its not recomended for this lab.

## What the playbook does

[`site.yml`](site.yml) is the only playbook entry point. 

| Play | Inventory group | Roles | Purpose |
| --- | --- | --- | --- |
| Server precheck | `server` | `precheck_server` | Checks that each Ubuntu server's OS hostname matches its inventory name and that its expected IPv4 addresses are present. Runs before the configuration plays. |
| MikroTik config | `mikrotik` | `presetup_mikrotik`, `ipaddress_mikrotik`, `dhcp_mikrotik`, `bgp_mikrotik` | Creates the lab API user, sets router identity and DNS behavior, configures interfaces and IP addresses, creates DHCP services, and configures RouterOS BGP. |
| LB4 config | `lb4` | `presetup_anycast_int`, `haproxy`, `frr` | Creates the `anycast0` interface and configures the anycast address, installs HAProxy, installs/configures FRR, and validates BGP neighbors. |
| LB7 config | `lb7` | `haproxy` | Installs HAProxy and configures the outer load-balancing layer. |
| App config | `app` | `web_app` | Installs Nginx and publishes the global and zone-specific example pages. |

Run individual sections with tags when troubleshooting:

```bash
ansible-playbook site.yml --tags precheck
ansible-playbook site.yml --tags mikrotik
ansible-playbook site.yml --tags lb7
ansible-playbook site.yml --tags lb4
ansible-playbook site.yml --tags app
```

The precheck is the first play in a normal `site.yml` run. Run the `precheck`
tag by itself after Terraform creates or changes the servers to check them
before applying the configuration roles.

## Directory structure

```text
.
├── ansible.cfg                 # Ansible defaults and inventory location
├── inventory.ini               # Hosts, groups, and management IP addresses
├── site.yml                    # Main playbook and role execution order
├── requirement.txt             # Python package requirements
├── requirement.yml             # Ansible Galaxy collection requirements
├── 9-vm-2gb/
│   └── vm.yaml                 # Nine-VM Proxmox generator input
├── group_vars/                 # Variables shared by inventory groups
├── host_vars/                  # Router-specific interface and BGP data
├── roles/                      # Reusable configuration units
│   ├── bgp_mikrotik/
│   ├── dhcp_mikrotik/
│   ├── frr/
│   ├── haproxy/
│   ├── ipaddress_mikrotik/
│   ├── precheck_server/
│   ├── presetup_anycast_int/
│   ├── presetup_mikrotik/
│   └── web_app/
└── test-file-and-other/        # Older experiments and unused examples
```

Each role keeps its implementation under `tasks/`. Roles that render service
configuration also have `templates/`, and roles that need a service reload have
`handlers/`.

## Variables and inventory

`inventory.ini` describes the topology using nested groups:

- `ganjil` and `genap` represent the two lab zones.
- `mikrotik`, `lb7`, `lb4`, and `app` are the functional groups targeted by
	`site.yml`.
- `server` contains all Ubuntu server groups.

The inventory also supplies `ansible_host`, which is the management address
used to connect to each device. The addresses in the inventory are lab
addresses; replace them for another environment.

Variable files provide the data consumed by the roles:

- `group_vars/all.yml` contains the shared address map and the lab anycast IP.
- `group_vars/ganjil.yml` and `group_vars/genap.yml` contain per-zone ASNs,
	client networks, DHCP pools, BGP neighbors, and HAProxy backends.
- `group_vars/mikrotik.yml` selects the RouterOS connection type and API
	credentials.
- `group_vars/lb7.yml` and `group_vars/lb4.yml` select the HAProxy template and for lb7.yml there is a port mapping for haproxy.
- `group_vars/server.yml` contains the Ubuntu connection settings and shared
	application settings.
- `host_vars/<router>/host.yml` contains each MikroTik router's interface
	comments, IP addresses, BGP peers, address lists, and, where applicable,
	DHCP definitions.

The templates use `hostvars` and the shared address map to derive peer
addresses and ASNs. When changing a VM name or interface address, update the
inventory, group variables, host variables, and VM generator input together.

## Role overview

### MikroTik roles

- `presetup_mikrotik` creates the configured RouterOS API user, sets the system
	identity, and enables remote DNS requests for the lab (this playbook not idempotent so it always replace).
- `ipaddress_mikrotik` applies interface comments and IP addresses through the
	RouterOS API.
- `dhcp_mikrotik` creates DHCP pools, networks, and servers from each router's
	host variables.
- `bgp_mikrotik` creates the BGP address list, BGP instance, and BGP connections.

The RouterOS plays use `community.routeros` API modules. `gather_facts` is
disabled for these devices because i dont need it and it should use gather facts from community it self.

### Ubuntu server roles

- `precheck_server` gathers facts and checks that each server's OS hostname
	matches its inventory name and that all IPv4 addresses declared for that
	server in `group_vars/all.yml` are present on the host. It does not validate
	subnet masks, gateways, connectivity, or unexpected extra addresses, so it
	is a limited sanity check rather than proof that Terraform provisioned the
	server correctly. The server must be reachable over SSH for fact gathering.
- `presetup_anycast_int` runs on LB4 hosts, creates the `anycast0` dummy
	interface, and configures the anycast `/32` address and route.
- `frr` adds the FRR repository, installs FRR, renders `daemons` and `frr.conf`,
	and checks that expected BGP peers reach `Established` state. It runs on LB4.
- `haproxy` installs HAProxy, selects the LB4 or LB7 template based on group
	variables, validates the generated configuration, and restarts the service
	when it changes. It runs on both LB4 and LB7.
- `web_app` installs Nginx, creates the configured web roots, renders the global
	and zone pages, enables the Nginx site, and runs `nginx -t`.

## Prerequisites

You need:

- An Ansible control machine with Python and the packages in
	`requirement.txt`.
- The collection listed in `requirement.yml`.
- A Proxmox lab with the required bridges, datastores, and base VM template.
- Ubuntu servers provisioned by Terraform (or an equivalent process), with
	names and addresses matching `inventory.ini` and `group_vars/`.
- Network access from the control machine to the MikroTik management API and
	SSH access to the Ubuntu VMs.

Install the Ansible dependencies in a virtual environment, for example:

```bash
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirement.txt
ansible-galaxy collection install -r requirement.yml
```

Check connectivity before applying changes:

```bash
ansible-inventory --graph
ansible-inventory --list
ansible-playbook site.yml --check
```

Run the complete lab configuration with:

```bash
ansible-playbook site.yml
```

`--check` cannot fully predict all RouterOS API changes or service behavior, so
use it as an initial review rather than a guarantee that the run is safe.

## Lab-only warnings (generated by AI i think this usefull so im not remove it)

- Do not reuse the example passwords, API tokens, SSH keys, or addresses in a
	real environment. Store secrets with Ansible Vault or another secret manager.
- Review `9-vm-2gb/vm.yaml` before sharing it. It contains Proxmox connection
	settings and VM bootstrap credentials/keys.
- `host_key_checking=false` is enabled in `ansible.cfg` for convenience in the
	lab and is not an appropriate default for production.
- The roles install packages from external repositories and assume Ubuntu,
	systemd, and the repository's expected distribution release.
- The playbook can change routing, DHCP, DNS, proxy, and web-server state. Run
	it only against disposable lab devices.

## Archived examples

`test-file-and-other/` contains older test playbooks, templates, and backups.
They are kept as reference material and are not included by `site.yml`.
