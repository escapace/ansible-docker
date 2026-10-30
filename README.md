# ansible-docker

Configures rootful Docker CE on EL10 with native nftables, SELinux integration,
and a default bridge with IPv4 and optional IPv6. Container output goes to the
systemd journal. Native build-cache GC and daily old dangling-image cleanup
manage disposable data. The role configures boot policy without starting inactive
services or timers; the deployment caller activates a newly prepared host.

Run with root privileges, gathered facts, and systemd tools. The caller supplies
`/etc/sysctl.d`, provided by EL10's `systemd-udev` package.

## Variables

- `docker_users` defaults to `[]`. Supply a list of existing, non-empty usernames
  to add to the `docker` group. Missing accounts, `null`, and non-list values fail.
  Other group memberships remain unchanged. Removing a name or emptying the list
  does not revoke existing access.
- `docker_tailscale_integration` defaults to `false` and requires a YAML boolean.
  Set `true` to allow Docker-zone forwarding through a discovered Tailscale zone.
  Setting `false` removes only Docker's forwarding exception.
- `docker_build_cache_min_free_space` defaults to `"10GB"` (10 GiB). Supply a
  positive byte-size string with `B`, `KB`, `MB`, `GB`, or `TB` units. Units are
  case-insensitive and binary: `1GB` is 1,073,741,824 bytes. Decimal quantities
  are accepted and rounded to whole bytes. IEC suffixes such as `GiB`, unitless
  values, zero, empty strings, and `null` are invalid. The byte count must fit
  a signed 64-bit integer.
- `docker_dangling_image_prune_after` defaults to `"168h"` (seven days). Supply
  a whole-hour duration from `1h` to `2562047h`, or `""` to disable the cleanup
  timer. `null` is invalid. This does not disable native build-cache GC.
- `docker_storage_device` defaults to `""`, which skips dedicated storage
  configuration. A non-empty string containing an absolute `/dev/` block-device
  path enables it. Boot-time setup can overwrite the selected device. Use a path
  such as `/dev/disk/by-id/virtio-docker-data`. A dedicated disk or partition is
  accepted. Prefer a stable `/dev/disk/by-id/` path; wildcards and surrounding
  whitespace are invalid. The device must exist on the booted host, not
  necessarily during image construction. `null` is invalid.
- `docker_storage_volume_group` defaults to `docker`. Set an LVM volume group
  name for the dedicated device. The Docker role owns its complete declaration.
- `docker_bridge_ipv4` defaults to `172.17.0.1/16`. Supply a private IPv4
  gateway and prefix that do not overlap host or routed networks.
- `docker_bridge_ipv6_subnet` defaults to `fd3b:7a4e:6b20:1::/64`. A non-empty
  string enables default-bridge IPv6 and must be a canonical private IPv6 /64
  outside Tailscale's range. Set boolean `false` or `""` to disable it; an actually
  undefined value also disables it. Omitting an override retains the default
  subnet. `true`, `null`, and other types are invalid. This does not disable host
  IPv6, forwarding, or IPv6 on user-defined networks. Docker requires default
  bridge IPv4. Kresd integration uses the IPv6 subnet's first host as its gateway.

Docker-group membership grants effective root access. Users need a new login
session after gaining membership. Membership changes do not restart Docker or
refresh existing sessions.

Example host variables, using the stable path of the dedicated device:

```yaml
docker_users:
  - developer
docker_storage_volume_group: docker
docker_storage_device: /dev/disk/by-id/virtio-docker-data
```

With a device specified, the role conditionally invokes `escapace.storage_setup`
and writes `/etc/storage-setup.d/20-docker.conf` containing the volume group,
logical volume, and filesystem declarations. Dedicated-storage callers include
`escapace.storage_setup` in their role requirements. Without a device, Docker does
not invoke that role. `storage_setup` only installs provisioning machinery;
Docker owns the storage declaration.

Optional storage uses the volume group's remaining free space for one XFS
filesystem mounted at `/var/lib/docker`. Containerd stores images and snapshots
in `/var/lib/docker/containerd`; Docker stores container metadata and volumes
alongside it. Both daemons refuse to start without the expected filesystem.
Existing data is not migrated, and an established volume-group mapping cannot
be changed through this role. Clearing the device skips configuration; it does
not remove existing storage or previously written provisioning configuration.

## Logging and cleanup

Journald is the default for newly created containers. Existing containers retain
their logging configuration until deliberately recreated by their owner; the role
does not recreate them. `docker logs` remains available through Docker access.
The host owns journal persistence, size limits, retention, rate limiting, and
permissions. A dedicated Docker filesystem does not move the host journal.

BuildKit GC has no fixed cache-size ceiling and no protected minimum cache
footprint. It reclaims eligible cache toward `docker_build_cache_min_free_space`.
Docker's native age/type policies still apply, including preferential cleanup of
regenerable cache unused for 48 hours and older general cache unused for 60 days.
These are selection priorities, not guaranteed retention periods. Active builds,
shared content, and application data can prevent reaching the free-space target.
The policy is the same with or without dedicated storage.

The daily timer directly invokes Docker's dangling-image prune command against
`/run/docker.sock`, without `--all` or a cleanup script. Runs are delayed by up to
one hour; missed calendar runs can catch up when the timer is activated. Cleanup
skips inactive Docker rather than starting it. The age filter is a creation-age
cutoff, not time since last use or since losing a tag. Tag valuable build or
rollback images to retain them; an untagged image ID is not a retention mechanism.
Tagged images and images required by existing containers are not cleanup targets.
Containers, volumes, networks, and bind-mounted data are not pruned.

The role enables `docker-dangling-image-prune.timer` at boot when cleanup is
configured, but does not start an inactive timer. Deployment callers activate it
after Docker is ready. An empty cleanup age disables the timer and stops its
active cleanup units without changing Docker's state.

## Networking

Firewalld is optional and discovered through `/etc/firewalld/policies`. Docker
does not invoke `escapace.firewalld` or configure the base firewall. It validates
assembled native configuration only after changing its own firewall objects,
then reloads firewalld only when it is running.

DNS integration is automatic when both the firewalld policy directory and
`/etc/knot-resolver/config.d` exist. Missing components leave DNS unmanaged rather
than expose an unprotected bridge listener. Discovered installations with missing
required validators or configuration fail with a diagnostic. Without integration,
a loopback-only host resolver can cause the legacy default bridge to select
public fallback servers; user-defined networks keep their normal DNS behavior.

When discovered, kresd listens on the default bridge's IPv4 gateway and,
when enabled, its IPv6 gateway. Docker uses those addresses as default upstreams;
user-defined bridges keep `127.0.0.11` and container-specific DNS overrides still
work. Existing workloads may need owner-controlled recreation to adopt the DNS
defaults; the role does not recreate them. Firewalld admits TCP/UDP 53 to those
destinations from Docker's dynamically maintained `docker` zone and rejects other
network ingress. Reserve these addresses for Docker-facing DNS, not non-Docker
clients. Other kresd listener configuration, forwarding policies, and host
`/etc/resolv.conf` are unchanged. The integration requires kresd's sorted Lua
fragment loader and `view.rule_dst`, available in Knot Resolver 6.5.1.

This covers ordinary Docker-managed bridge networks, not host, macvlan, ipvlan,
or intentionally isolated internal networks. IPv6-only clients need the IPv6
bridge endpoint enabled or their own reachable DNS override. The role does not
silently enable bridge IPv6 or install systemd-resolved.

The role writes persistent configuration in tasks. Change-triggered handlers
validate native configuration and reconcile only active services. Docker starts
after kresd when both are activated, but does not pull an inactive resolver up.
The deployment caller activates firewalld and kresd before Docker.

For live endpoint changes or disablement, the caller retires the previous
Docker-only listeners while their firewall protection remains in place. It also
orders firewall application before any resolver restart that loads new listeners.
The owned listener fragment is `/etc/knot-resolver/config.d/700-docker.lua`.
When integration is unavailable, the role removes only that fragment and its
owned DNS policies, preserving Docker's zone and other kresd configuration.
Image preparation does not require active services or perform a live cutover.

Tailscale forwarding requires explicit opt-in with
`docker_tailscale_integration: true`, the firewalld policy directory, and
`/etc/firewalld/zones/tailscale.xml`. Prepare that zone before invoking this role.
Missing integrations remain unmanaged. When opted in, Docker owns a
`docker` → `tailscale` accept policy at priority `-500`,
before Tailscale's forwarding deny at `-100`. It permits all protocols and both IP
families from Docker's zone through Tailscale, including public destinations
routed through it. Other ingress zones receive no exception. Containers share
the host's tailnet identity and remain subject to its tailnet access rules.

The role neither invokes `escapace.tailscale` nor changes its configuration. The
caller sets `tailscale_stateful_filtering: false` when forwarded connections are
wanted, and applies the firewall exception before disabling that filter. When the
integration is disabled or disappears, Docker removes only its own forwarding
policy; existing
connections can persist until their connection-tracking state expires.

Docker Engine 29 or newer is required. Its native nftables backend is
experimental and does not support Swarm. Docker owns its generated firewall
rules and retains normal port-publishing behavior. The role does not select VFS
or schedule general resource pruning.
