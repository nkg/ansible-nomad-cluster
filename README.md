# ansible-nomad-cluster

Ansible role that bootstraps a [Nomad](https://www.nomadproject.io)
cluster — installs the binary (skips when a pre-baked version is
present at the right tag), drops `nomad.hcl` + systemd unit, joins
servers + clients, registers the podman driver on clients.

Targets **Debian 13 (trixie)** so it matches the LXC templates from
[`nkg/distrobuilder-proxmox-lxc-images`](https://github.com/nkg/distrobuilder-proxmox-lxc-images)
and the VMs built from the Ubuntu cloud-image template that
[`nkg/terraform-proxmox-fleet`](https://github.com/nkg/terraform-proxmox-fleet)
provisions.

## Scope (v0.1.0)

Does:
- Installs Nomad at a pinned version (`/usr/local/bin/nomad`)
- Installs the `nomad-driver-podman` plugin on clients
- Renders `/etc/nomad.d/nomad.hcl` per role (server vs client)
- Drops a systemd unit + enables + starts it
- Server-join via `server_join.retry_join` (3-server quorum default)
- Client-join via `client.servers`

Doesn't (yet):
- ACL bootstrap / token management
- mTLS between agents
- Consul integration
- Vault integration
- Multi-distro support (Debian 13 only)
- Molecule tests (CI runs `--syntax-check` + lint, not full convergence)

ACLs / mTLS / Consul belong in v0.2+. The current trust model: agents
on a private VLAN behind a firewall, plain-text RPC. Mirrors the
threat model the rest of the
[`nkg/`](https://github.com/orgs/nkg) Proxmox runner platform
assumes.

## Quick start

```yaml
# inventory.ini
[nomad_servers]
nomad-server-1 ansible_host=192.168.1.121
nomad-server-2 ansible_host=192.168.1.122
nomad-server-3 ansible_host=192.168.1.123

[nomad_clients]
nomad-client-1 ansible_host=192.168.1.151
nomad-client-2 ansible_host=192.168.1.152
```

```yaml
# site.yml
- hosts: nomad_servers
  become: true
  vars:
    nomad_role: server
    nomad_servers: ["192.168.1.121", "192.168.1.122", "192.168.1.123"]
  roles: [nomad_cluster]

- hosts: nomad_clients
  become: true
  vars:
    nomad_role: client
    nomad_servers: ["192.168.1.121", "192.168.1.122", "192.168.1.123"]
  roles: [nomad_cluster]
```

```bash
ansible-playbook -i inventory.ini site.yml
```

A complete worked example with both files lives under [`examples/`](examples/).

## Variables

The full set lives in [`defaults/main.yml`](defaults/main.yml). The
ones you'll typically set per deployment:

| Variable | Default | Description |
|---|---|---|
| `nomad_role` | `client` | One of `server`, `client`, `both`. |
| `nomad_servers` | `[]` (**required**) | Every server's IP/hostname. Both server `retry_join` and client `servers` read this. |
| `nomad_server_bootstrap_expect` | `3` | Number of servers before the cluster elects a leader. |
| `nomad_datacenter` | `dc1` | Logical DC name. Single-DC fleets keep the default. |
| `nomad_version` | `1.8.4` | Pinned Nomad version. Skipped if `/usr/local/bin/nomad -v` already matches. |
| `nomad_driver_podman_version` | `0.6.2` | Pinned podman driver version. |
| `nomad_advertise_addr` | `ansible_default_ipv4.address` | Address other nodes use to reach this one. Set explicitly on multi-NIC hosts. |
| `nomad_podman_socket` | `unix:///run/podman/podman.sock` | Where the podman driver finds the podman API. |

## Idempotency

Re-running the role on a host that's already converged:

- Doesn't re-download the binary (version check, file-existence check)
- Doesn't restart Nomad unless `nomad.hcl` or the systemd unit changed
- Handlers fire only on real config changes

## Why a Nomad binary install, not the apt repo?

Two reasons:

1. **Version pinning is explicit.** The HashiCorp apt repo tracks
   stable channels and your `apt upgrade` could float Nomad mid-week.
   A binary install pinned by SHA-equivalent (the URL contains the
   version) keeps reproduction air-tight.
2. **Matches the distrobuilder template.** The `nomad-client.tar.xz`
   LXC template from [nkg/distrobuilder-proxmox-lxc-images](https://github.com/nkg/distrobuilder-proxmox-lxc-images)
   pre-installs the binary at the same path with the same version
   scheme, so clients hit the "already installed at right version,
   skipping" path and the role becomes a thin "render config + start
   service" pass over them.

## Requirements

| Tool | Version |
|---|---|
| Ansible | core ≥ 2.16 |
| Target OS | Debian 13 (trixie) |
| Target arch | amd64 (default); set `nomad_arch: arm64` for ARM hosts |

## License

MIT.
