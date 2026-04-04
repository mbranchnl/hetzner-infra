# hetzner_infra

Declarative Hetzner Cloud infrastructure management for Ansible. Define your desired state in YAML files and the role reconciles it against the live Hetzner API — the same mental model as Terraform, but with native Ansible inventory.

## How it works

```
inventory/host_vars/web01/main.yml   ← desired state (YAML)
          └─► hetzner_infra role     ← reconcile
                    └─► Hetzner API  ← actual state
```

- **Add a host** to the inventory → server gets created on next run
- **Set `hetzner_server_state: absent`** on a host → server gets deleted
- **Enable `hetzner_manage_absent: true`** → any Hetzner server not present in the inventory group is deleted (full Terraform parity)
- Runs are **idempotent** — re-running with no changes is a no-op

## Requirements

- Ansible >= 2.14
- `hetzner.hcloud` collection: `ansible-galaxy collection install hetzner.hcloud`
- Hetzner Cloud API token (project → Security → API Tokens)

## Quick start

```bash
export HCLOUD_TOKEN=your_token_here
ansible-playbook -i examples/inventory/hosts.yml examples/playbook.yml
```

## Inventory structure

```
inventory/
├── hosts.yml                         # groups and host list
├── group_vars/
│   └── hetzner_servers.yml           # shared defaults + project-level resources
└── host_vars/
    ├── web01/
    │   └── main.yml                  # server-specific overrides
    ├── web02/
    │   └── main.yml
    └── db01/
        └── main.yml
```

The `inventory_hostname` is used as the **Hetzner server name**.

## Execution order

Resources are processed in dependency order so that project-level resources always exist before servers reference them:

```
pre_flight → ssh_keys → networks → firewalls → server → volumes → floating_ip → [purge]
             └─────────────────────────────┘   └──────────────────────────────┘
                  project-level (run_once)             per-host
```

## Variable reference

### Server — per-host, in `host_vars`

| Variable | Default | Description |
|---|---|---|
| `hetzner_server_state` | `present` | `present` · `absent` · `stopped` · `started` · `restarted` |
| `hetzner_server_type` | `cx23` | Hetzner server type (e.g. `cx22`, `cx32`, `ccx13`) |
| `hetzner_location` | `nbg1` | Datacenter location: `nbg1` · `fsn1` · `hel1` · `ash` · `hil` · `sin` |
| `hetzner_image` | `ubuntu-24.04` | OS image name |
| `hetzner_ssh_keys` | `[]` | SSH key names to inject — must exist in Hetzner (see `hetzner_ssh_keys_upload`) |
| `hetzner_labels` | `{}` | Key/value labels applied to the server |
| `hetzner_networks` | `[]` | Private network names to attach the server to |
| `hetzner_firewalls` | `[]` | Firewall names to attach to the server |
| `hetzner_enable_ipv4` | `true` | Assign a public IPv4 address |
| `hetzner_enable_ipv6` | `true` | Assign a public IPv6 address |
| `hetzner_backups` | `false` | Enable Hetzner automated backups |
| `hetzner_protection.delete` | `false` | Protect the server against accidental deletion |
| `hetzner_protection.rebuild` | `false` | Protect the server against rebuilds |

### Volumes — per-host, in `host_vars`

Defined as a list under `hetzner_volumes`. Each entry creates and attaches a block storage volume.

| Key | Required | Description |
|---|---|---|
| `name` | yes | Volume name in Hetzner |
| `size` | yes | Size in GB |
| `format` | no | Filesystem: `ext4` or `xfs` — only applied on first creation |
| `automount` | no | Mount automatically via `/etc/fstab` (default: `false`) |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` to detach and delete |

```yaml
hetzner_volumes:
  - name: db01-data
    size: 100
    format: ext4
    automount: true
    labels:
      purpose: database
  - name: db01-backup
    size: 200
```

Facts set: `hetzner_volume_devices` — list of volume objects returned by the API.

### Floating IPs — per-host, in `host_vars`

Defined as a list under `hetzner_floating_ips`. Each entry creates a floating IP and, by default, assigns it to the host. Re-runs are idempotent — the IP is only (re)assigned when it is not already pointing at the correct server.

| Key | Required | Description |
|---|---|---|
| `name` | yes | Floating IP name in Hetzner |
| `type` | no | `ipv4` (default) or `ipv6` |
| `home_location` | no | Defaults to `hetzner_location` |
| `assign` | no | Assign to this server (default: `true`) |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` |

```yaml
hetzner_floating_ips:
  - name: web01-public
    type: ipv4
    assign: true
    labels:
      env: production
```

Facts set: `hetzner_floating_ip_addresses` — list of floating IP objects returned by the API.

### SSH keys — project-level, in `group_vars`

Uploads public keys to the Hetzner project before servers are created. Runs once per play.

| Key | Required | Description |
|---|---|---|
| `name` | yes | Key name in Hetzner |
| `public_key` | yes | Public key string (use `lookup('file', ...)` for local files) |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` to remove the key |

```yaml
hetzner_ssh_keys_upload:
  - name: admin-key
    public_key: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
  - name: ci-key
    public_key: "ssh-ed25519 AAAA..."
```

### Networks — project-level, in `group_vars`

Creates private networks and their subnets before servers are attached. Runs once per play.

| Key | Required | Description |
|---|---|---|
| `name` | yes | Network name in Hetzner |
| `ip_range` | yes | Network CIDR (e.g. `10.0.0.0/8`) |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` |
| `subnets` | no | List of subnet definitions (see below) |

**Subnet keys:**

| Key | Required | Description |
|---|---|---|
| `ip_range` | yes | Subnet CIDR — must be within the network range |
| `network_zone` | no | Defaults to `hetzner_network_zone` — `eu-central` · `us-east` · `us-west` · `ap-southeast` |
| `type` | no | `cloud` (default) · `server` · `vswitch` |

```yaml
hetzner_networks_config:
  - name: prod-network
    ip_range: 10.0.0.0/8
    subnets:
      - ip_range: 10.0.1.0/24
      - ip_range: 10.0.2.0/24
        network_zone: eu-central
        type: cloud
```

### Firewalls — project-level, in `group_vars`

Creates firewalls and their rules before servers reference them. Runs once per play.

| Key | Required | Description |
|---|---|---|
| `name` | yes | Firewall name in Hetzner |
| `rules` | no | List of rule definitions (see below) |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` |

**Rule keys:**

| Key | Required | Description |
|---|---|---|
| `direction` | yes | `in` or `out` |
| `protocol` | yes | `tcp` · `udp` · `icmp` · `esp` · `gre` |
| `port` | no | Port or range (e.g. `"22"`, `"8000-9000"`) — required for `tcp`/`udp` |
| `source_ips` | no | List of source CIDRs for inbound rules |
| `destination_ips` | no | List of destination CIDRs for outbound rules |
| `description` | no | Human-readable label |

```yaml
hetzner_firewalls_config:
  - name: web-firewall
    rules:
      - direction: in
        protocol: icmp
        source_ips: ["0.0.0.0/0", "::/0"]
        description: Ping
      - direction: in
        protocol: tcp
        port: "22"
        source_ips: ["0.0.0.0/0", "::/0"]
        description: SSH
      - direction: in
        protocol: tcp
        port: "443"
        source_ips: ["0.0.0.0/0", "::/0"]
        description: HTTPS
```

### Role behaviour

| Variable | Default | Description |
|---|---|---|
| `hetzner_network_zone` | `eu-central` | Default network zone used for subnets that do not specify one |
| `hetzner_manage_absent` | `false` | When `true`, delete any Hetzner server not present in the inventory group |
| `hetzner_api_token` | `$HCLOUD_TOKEN` | API token — falls back to the `HCLOUD_TOKEN` environment variable |

## Facts set by the role

The following facts are available on each host after the role runs and can be used in subsequent tasks or roles.

| Fact | Description |
|---|---|
| `hetzner_server_ipv4` | Public IPv4 address |
| `hetzner_server_ipv6` | Public IPv6 address |
| `hetzner_server_id` | Hetzner server ID |
| `hetzner_server_status` | Current server status |
| `hetzner_volume_devices` | List of volume objects for this host |
| `hetzner_floating_ip_addresses` | List of floating IP objects for this host |

## Removing servers

**Option A** — declare the desired state in `host_vars` and run the playbook:

```yaml
# host_vars/web02/main.yml
hetzner_server_state: absent
```

**Option B** — remove the host from `hosts.yml` and run with purge mode enabled. This deletes every Hetzner server that is not listed in the inventory group:

```bash
ansible-playbook -i inventory/hosts.yml playbook.yml -e hetzner_manage_absent=true
```

## License

AGPL-3.0 license
