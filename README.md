# hetzner_infra

Declarative Hetzner Cloud infrastructure management for Ansible. Define your desired state in YAML files and the role reconciles it against the live Hetzner API — the same mental model as Terraform, but with native Ansible inventory.

## How it works

```text
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

define `vault_hetzner_api_token` or export the token:

```bash
export HCLOUD_TOKEN=your_token_here
```

Execute playbook:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/playbook.yml
```

## Inventory structure

```text
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

```text
pre_flight
├── project/          (run_once)
│   ├── ssh_keys
│   ├── network/networks
│   └── network/firewalls
├── server
├── network/loadbalancers   (run_once)
├── storage/
│   ├── volumes
│   └── storage_boxes
├── network/floating_ip
└── purge             (run_once, optional)
```

## Variable reference

### Server — per-host, in `host_vars`

| Variable | Default | Description |
| --- | --- | --- |
| `hetzner_server_state` | `present` | `present` · `absent` · `stopped` · `started` · `restarted` |
| `hetzner_server_type` | `cx23` | Hetzner server type (e.g. `cx22`, `cx32`, `ccx13`) |
| `hetzner_location` | `nbg1` | Datacenter location: `nbg1` · `fsn1` · `hel1` · `ash` · `hil` · `sin` |
| `hetzner_image` | `ubuntu-24.04` | OS image name |
| `hetzner_ssh_keys` | `[]` | SSH key names to inject — must exist in Hetzner (see `hetzner_ssh_keys_upload`) |
| `hetzner_labels` | `{}` | Key/value labels applied to the server — see [Labels](#labels) |
| `hetzner_networks` | `[]` | Private network names to attach the server to |
| `hetzner_firewalls` | `[]` | Firewall names to attach to the server |
| `hetzner_enable_ipv4` | `true` | Assign a public IPv4 address |
| `hetzner_enable_ipv6` | `true` | Assign a public IPv6 address |
| `hetzner_backups` | `false` | Enable Hetzner automated backups |
| `hetzner_protection.delete` | `false` | Protect the server against accidental deletion |
| `hetzner_protection.rebuild` | `false` | Protect the server against rebuilds |
| `hetzner_server_ipv4_private` | — | Static private IP to assign within the network (optional; requires `hetzner_server_ipv4_private_network`) |
| `hetzner_server_ipv4_private_network` | — | Network name to attach the server to with the static private IP (required when `hetzner_server_ipv4_private` is set) |

### Labels

`hetzner_labels` is a flat mapping of string key/value pairs attached to the Hetzner server resource. Labels are reconciled on every run — adding, changing, or removing a key updates the server in-place without recreating it.

```yaml
hetzner_labels:
  env: production
  role: web
  team: platform
```

**Constraints** (enforced by the Hetzner API):

- Keys and values must be strings.
- Keys may contain letters, digits, `-`, `_`, and `.`, with an optional `prefix/` namespace (e.g. `example.com/env`).
- Maximum 256 labels per resource.

**Use with load balancer `label_selector` targets**

The primary reason to label servers is so load balancers can target them dynamically. When a server carries a matching label, the load balancer picks it up automatically — no manual target list to maintain:

```yaml
# host_vars/web01/main.yml
hetzner_labels:
  role: web

# group_vars/hetzner_servers.yml
hetzner_loadbalancers_config:
  - name: web-lb
    load_balancer_type: lb11
    location: nbg1
    targets:
      - type: label_selector
        label_selector: role=web   # matches any server with this label
```

Adding a new host with `role: web` in its `hetzner_labels` is enough for it to appear behind the load balancer on the next run.

### Volumes — per-host, in `host_vars`

Defined as a list under `hetzner_volumes`. Each entry creates and attaches a block storage volume.

| Key | Required | Description |
| --- | --- | --- |
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

### Storage boxes — per-host, in `host_vars`

Defined as a list under `hetzner_storage_boxes`. Each entry provisions a Hetzner Storage Box.

| Key | Required | Description |
| --- | --- | --- |
| `name` | yes | Storage box name in Hetzner |
| `product` | yes | Storage box plan (e.g. `BX11`, `BX21`, `BX41`) |
| `location` | no | Datacenter location — defaults to `hetzner_location` |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` |

```yaml
hetzner_storage_boxes:
  - name: db01-backups
    product: BX21
    labels:
      purpose: backup
  - name: db01-archive
    product: BX41
    location: fsn1
```

Facts set: `hetzner_storage_box_devices` — list of storage box objects returned by the API.

### Floating IPs — per-host, in `host_vars`

Defined as a list under `hetzner_floating_ips`. Each entry creates a floating IP and, by default, assigns it to the host. Re-runs are idempotent — the IP is only (re)assigned when it is not already pointing at the correct server.

| Key | Required | Description |
| --- | --- | --- |
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

### SSH keys — project-level, in `group_vars` (`tasks/project/ssh_keys.yml`)

Uploads public keys to the Hetzner project before servers are created. Runs once per play.

| Key | Required | Description |
| --- | --- | --- |
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

### Networks — project-level, in `group_vars` (`tasks/network/networks.yml`)

Creates private networks and their subnets before servers are attached. Runs once per play.

| Key | Required | Description |
| --- | --- | --- |
| `name` | yes | Network name in Hetzner |
| `ip_range` | yes | Network CIDR (e.g. `10.0.0.0/8`) |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` |
| `subnets` | no | List of subnet definitions (see below) |

**Subnet keys:**

| Key | Required | Description |
| --- | --- | --- |
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

### Firewalls — project-level, in `group_vars` (`tasks/network/firewalls.yml`)

Creates firewalls and their rules before servers reference them. Runs once per play.

| Key | Required | Description |
| --- | --- | --- |
| `name` | yes | Firewall name in Hetzner |
| `rules` | no | List of rule definitions (see below) |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` |

**Rule keys:**

| Key | Required | Description |
| --- | --- | --- |
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

### Load balancers — project-level, in `group_vars` (`tasks/network/loadbalancers.yml`)

Creates load balancers and attaches their networks, services, and targets. Runs once per play.

| Key | Required | Description |
| --- | --- | --- |
| `name` | yes | Load balancer name in Hetzner |
| `load_balancer_type` | yes | LB type (e.g. `lb11`, `lb21`, `lb31`) |
| `location` | no | Datacenter location — mutually exclusive with `network_zone` |
| `network_zone` | no | Network zone — mutually exclusive with `location` |
| `algorithm` | no | `round_robin` (default) or `least_connections` |
| `labels` | no | Key/value labels |
| `state` | no | `present` (default) or `absent` |
| `networks` | no | List of private network names to attach the LB to |
| `services` | no | List of service definitions (see below) |
| `targets` | no | List of target definitions (see below) |

**Service keys:**

| Key | Required | Description |
| --- | --- | --- |
| `protocol` | yes | `http` · `https` · `tcp` |
| `listen_port` | yes | Port the LB listens on |
| `destination_port` | no | Port forwarded to targets (defaults to `listen_port`) |
| `health_check` | no | Health check definition (see below) |

**Health check keys** (nested under a service's `health_check`):

| Key | Required | Description |
| --- | --- | --- |
| `protocol` | yes | `http` or `tcp` |
| `port` | yes | Port to probe on each target |
| `interval` | no | Seconds between probes (default: `15`) |
| `timeout` | no | Seconds before a probe times out (default: `10`) |
| `retries` | no | Failed probes before marking unhealthy (default: `3`) |
| `http.path` | no | HTTP path to request (default: `/`) |
| `http.status_codes` | no | Accepted status code patterns (e.g. `["2??"]`) |
| `http.response` | no | String the response body must contain (e.g. `"OK"`) |

**Target keys:**

| Key | Required | Description |
| --- | --- | --- |
| `type` | yes | `server` · `label_selector` · `ip` — determines which field below is used |
| `server` | no | Server name — required when `type: server` |
| `label_selector` | no | Label selector string — required when `type: label_selector` |
| `ip` | no | IP address — required when `type: ip` |

The three target types behave differently:

- **`server`** — adds a single named server as a target. The server must exist in the same Hetzner project. Use this when you manage a fixed, known set of backends.

  ```yaml
  targets:
    - type: server
      server: web01
    - type: server
      server: web02
  ```

- **`label_selector`** — adds all servers matching a label selector as targets. The LB tracks membership dynamically — servers added or removed from the group are picked up automatically. Use this for auto-scaling groups or any set of servers managed by label.

  ```yaml
  targets:
    - type: label_selector
      label_selector: role=web
  ```

- **`ip`** — adds an arbitrary IP address as a target. Use this to route traffic to servers outside the Hetzner project (e.g. bare metal, other clouds) or to a floating IP.

  ```yaml
  targets:
    - type: ip
      ip: 203.0.113.10
  ```

```yaml
hetzner_loadbalancers_config:
  - name: web-lb
    load_balancer_type: lb11
    location: nbg1
    algorithm: round_robin
    labels:
      env: production
    networks:
      - prod-network
    services:
      - protocol: http
        listen_port: 80
        destination_port: 8080
      - protocol: https
        listen_port: 443
        destination_port: 8443
        health_check:
          protocol: http
          port: 8080
          interval: 15
          timeout: 10
          retries: 3
          http:
            path: /health
            status_codes: ["2??"]
            response: "OK"
    targets:
      - type: label_selector
        label_selector: role=web
```

### Role behaviour

| Variable | Default | Description |
| --- | --- | --- |
| `hetzner_network_zone` | `eu-central` | Default network zone used for subnets that do not specify one |
| `hetzner_manage_absent` | `false` | When `true`, delete any Hetzner server not present in the inventory group |
| `hetzner_api_token` | `$HCLOUD_TOKEN` | API token — falls back to the `HCLOUD_TOKEN` environment variable |

## Facts set by the role

The following facts are available on each host after the role runs and can be used in subsequent tasks or roles.

| Fact | Description |
| --- | --- |
| `hetzner_server_ipv4` | Public IPv4 address |
| `hetzner_server_ipv6` | Public IPv6 address |
| `hetzner_server_id` | Hetzner server ID |
| `hetzner_server_status` | Current server status |
| `hetzner_volume_devices` | List of volume objects for this host |
| `hetzner_storage_box_devices` | List of storage box objects for this host |
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
