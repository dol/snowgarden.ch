+++
title = "Least-Privilege Log Collection: Extending otelcol with systemd Drop-ins"
date = 2026-03-04
draft = true

[taxonomies]
tags = ["opentelemetry", "linux", "observability", "systemd", "platform-engineering", "security", "otelcol", "logs"]

[extra]
author = "Dominic Lüchinger"
+++

<!-- markdownlint-disable-next-line MD025 -->
# Least-Privilege Log Collection: Extending otelcol with systemd Drop-ins

The OpenTelemetry Collector is the de-facto standard for collecting, transforming, and forwarding
telemetry data on Linux hosts. When you install it via the official DEB or RPM package, it deliberately
runs as a non-privileged `otel` system user — a sound security default. That same default, however,
blocks you from collecting the very log files you installed it to collect.

This post explains why the default is correct, what the `adm` group convention means from a security
perspective, and how to use a systemd drop-in to extend the service's capabilities in a
package-manager-safe and auditable way.

## Why Packages Run Services as Non-Root

The official `otelcol` packaging creates a dedicated system user at install time:

```sh
# preinstall.sh — from dol/opentelemetry-collector-releases
useradd --system --user-group --no-create-home --shell /sbin/nologin otel
```

And the unit file confines the service:

```ini
# /lib/systemd/system/otelcol.service
[Service]
User=otel
Group=otel
```

This follows the principle of least privilege: a compromised `otelcol` process can only read files that
the `otel` user is authorised to access. It cannot write to `/etc`, read `/root`, or modify other
users' files. For a process that may parse untrusted log data from remote sources, this matters.

## The `adm` Group: A Linux Log-Access Convention

Most Debian/Ubuntu and RHEL/Fedora installations ship with log files owned `root:adm` and mode `640`:

```text
-rw-r----- 1 root adm  /var/log/syslog
-rw-r----- 1 root adm  /var/log/auth.log
-rw-r----- 1 root adm  /var/log/kern.log
-rw-r----- 1 root adm  /var/log/messages
```

The `adm` group is not an escalation path — it does not grant write access, no `sudo` capability, and
no special kernel permissions. It is a read-only monitoring gate, designed precisely for tools like log
collectors, SIEM forwarders, and audit agents that need to read system logs without operating as root.

The convention traces back to early Unix `/usr/adm` directories, carried forward through System V's
`/var/adm` and into Linux's `/var/log` — the group name survived even as the directory path changed.

## The Symptom

After installing `otelcol` and configuring a filelog receiver:

```yaml
receivers:
  filelog:
    include:
      - /var/log/syslog
      - /var/log/auth.log
```

The collector either silently skips the files or emits:

```text
failed to open /var/log/syslog: open /var/log/syslog: permission denied
```

The `otel` user exists. The `adm` group exists with the right permissions. They are simply not
connected.

## The Pattern: systemd Drop-ins for Safe Capability Extension

The correct solution is a systemd drop-in — a small `.conf` file placed in
`/etc/systemd/system/<service>.service.d/` that overrides or supplements the vendor-supplied unit file
without modifying it.

```bash
sudo mkdir -p /etc/systemd/system/otelcol.service.d

sudo tee /etc/systemd/system/otelcol.service.d/adm-group.conf <<'EOF'
[Service]
SupplementaryGroups=adm
EOF

sudo systemctl daemon-reload
sudo systemctl restart otelcol
```

Or interactively (recommended on interactive systems):

```bash
sudo systemctl edit otelcol.service
# Add under [Service]:
# SupplementaryGroups=adm
```

`SupplementaryGroups` adds the named group(s) to the process's supplementary group list at runtime.
The `otel` user's primary group remains `otel`; the `adm` membership applies only to this service,
only when it is running.

### Why This Approach Is Preferable to Alternatives

| Approach | Works | Package-safe | Auditable | Least-privilege |
|----------|-------|-------------|-----------|-----------------|
| `usermod -aG adm otel` | ✓ | ✗ (user may be recreated) | ✗ | ✗ (applies globally) |
| Edit `/lib/systemd/system/otelcol.service` | ✓ | ✗ (overwritten on upgrade) | ✗ | ✓ |
| `SupplementaryGroups=adm` drop-in | ✓ | ✓ | ✓ | ✓ |
| Run as root | ✓ | ✓ | ✗ | ✗ |

The drop-in lives under `/etc/systemd/system/` — operator-managed config, not package-managed — so
package upgrades never touch it. Commit it to your configuration management system (Ansible, Puppet,
Salt, or a Git-tracked `/etc`) for full auditability.

## Verification

```bash
# Confirm the drop-in is loaded and merged
sudo systemctl cat otelcol.service | grep -A2 "Drop-in\|SupplementaryGroups"

# Show effective group list for the running process
PID=$(systemctl show -p MainPID --value otelcol)
grep '^Groups:' /proc/$PID/status

# Functional test — SupplementaryGroups is only active inside the systemd service, not via sudo -u
sudo journalctl -u otelcol -n 20 | grep -i "filelog\|syslog\|error"
```

## Scope and Security Considerations

- **`adm` is read-only**: Group membership cannot write to log files, rotate them, or escalate
  privileges. The blast radius of a compromised `otelcol` expands from "no log access" to
  "read-only log access" — an acceptable and intended trust level.
- **Systemd hardening is additive**: The drop-in only adds a supplementary group. Existing sandbox
  directives (`NoNewPrivileges`, `ProtectSystem`, etc.) remain effective. You can layer additional
  restrictions alongside `SupplementaryGroups`.
- **RHEL-family caveat**: On RHEL/Rocky/AlmaLinux, some log files under `/var/log` may be
  `root:root 600` and managed exclusively by journald. Verify file permissions before assuming
  `adm` will cover everything; consider enabling `rsyslog` to write to files if journal-based
  logs are required.

## The Broader Pattern

The same drop-in pattern applies any time an installed service needs constrained access to additional
resources:

- `SupplementaryGroups=docker` — allow the collector to access Docker socket for container metadata
- `SupplementaryGroups=ssl-cert` — allow a web server to read TLS certificates managed by `certbot`
- `ReadWritePaths=/var/lib/myapp` — extend writable paths without running as root

Systemd drop-ins are the operator's toolkit for safely layering permissions onto vendor-distributed
services. Learn the pattern once; apply it everywhere.
