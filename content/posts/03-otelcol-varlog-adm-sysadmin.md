+++
title = "otelcol Can't Read /var/log? One Drop-in Fixes It"
date = 2026-03-04
draft = false

[taxonomies]
tags = ["opentelemetry", "linux", "observability", "systemd", "sysadmin", "otelcol", "logs"]

[extra]
author = "Dominic Lüchinger"
+++

<!-- markdownlint-disable-next-line MD025 -->
# otelcol Can't Read /var/log? One Drop-in Fixes It

> **TL;DR** — The `otelcol` package runs as the non-privileged `otel` user, which has no read access
> to `/var/log` files owned by the `adm` group. Create one systemd drop-in that adds
> `SupplementaryGroups=adm` and you are done.

## The Symptom

You install the OpenTelemetry Collector from the official DEB or RPM package, configure a
[filelog receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver)
pointing at `/var/log/syslog` or `/var/log/auth.log`, and the collector silently collects nothing — or
throws a permission error like:

```text
failed to open /var/log/syslog: open /var/log/syslog: permission denied
```

## Why It Happens

The official package creates a locked-down system user at install time:

```sh
# preinstall.sh (from the otelcol package)
useradd --system --user-group --no-create-home --shell /sbin/nologin otel
```

The installed unit file runs entirely as that user:

```ini
# /lib/systemd/system/otelcol.service  (managed by the package)
[Service]
User=otel
Group=otel
```

On Debian, Ubuntu, and most RHEL-family systems, critical log files are group-readable by `adm`, not
world-readable:

```text
-rw-r----- 1 root adm   /var/log/syslog
-rw-r----- 1 root adm   /var/log/auth.log
-rw-r----- 1 root adm   /var/log/kern.log
```

The `otel` user is not a member of `adm`. Result: permission denied.

## The Fix — systemd Drop-in (3 Commands)

Never edit `/lib/systemd/system/otelcol.service` directly — it is owned by the package and will be
overwritten on upgrade. Use a drop-in instead:

```bash
sudo mkdir -p /etc/systemd/system/otelcol.service.d

sudo tee /etc/systemd/system/otelcol.service.d/adm-group.conf <<'EOF'
[Service]
SupplementaryGroups=adm
EOF

sudo systemctl daemon-reload && sudo systemctl restart otelcol
```

That is it. The `otel` user now has supplementary group membership in `adm` at runtime, without any
modification to the package-managed unit file or to the `otel` user account itself.

Alternatively, if you prefer the interactive editor:

```bash
sudo systemctl edit otelcol.service
```

Add under `[Service]`:

```ini
[Service]
SupplementaryGroups=adm
```

Save, exit — systemd auto-reloads.

## Verify It Works

```bash
# Confirm the drop-in is loaded
sudo systemctl cat otelcol.service | grep SupplementaryGroups

# Check the effective groups of the running process
sudo cat /proc/$(systemctl show -p MainPID --value otelcol)/status | grep Groups

# Confirm log events are flowing (the supplementary group is only active inside the service)
sudo journalctl -u otelcol -n 20 | grep -i "filelog\|syslog\|error"
```

## Why Not Just `usermod -aG adm otel`?

Adding the user to the group persistently works too, but the drop-in approach is cleaner:

- **No account mutation** — the `otel` user stays locked down everywhere except this one service.
- **Survives reinstalls** — package upgrades may recreate the user; your `/etc/systemd/system` drop-in is never touched.
- **Auditable** — the permission grant is in one config file under version control, not buried in `/etc/group`.

## A Brief History of the `adm` Group

The `adm` group is a Unix convention that predates Linux by two decades.

Early Unix systems (Bell Labs, Version 7, the BSDs) kept administrative files in `/usr/adm` —
login records, process accounting, error logs. The `adm` group controlled who could read them,
letting designated operators monitor the system without needing root.

As Unix commercialised through the 1980s (System V, HP-UX, Solaris, AIX), variable data migrated
from `/usr` into its own hierarchy. Logs moved to `/var/adm`. The group name and its purpose stayed
the same.

Linux, emerging in the early 1990s, made one further change: logs live in `/var/log`, not
`/var/adm`. The Filesystem Hierarchy Standard formalised this. But the `adm` group survived the
path change, and Debian's documentation still describes it plainly:

> *"Group `adm` is used for system monitoring tasks. Members of this group can read many log files
> in `/var/log`."*

Typical permissions on a fresh Debian/Ubuntu install:

```text
/var/log/syslog     root:adm  640
/var/log/auth.log   root:adm  640
/var/log/kern.log   root:adm  640
```

The `adm` group cannot run `sudo`, cannot write files, cannot touch system configuration. It is a
read-only monitoring gate — exactly what an OpenTelemetry Collector needs.

## RHEL/Rocky/AlmaLinux Caveat

On RHEL-family systems, some log files under `/var/log` may be `root:root 600` and written
exclusively via journald. The `adm` group trick will not help for those. Check first:

```bash
ls -l /var/log/messages /var/log/secure 2>/dev/null
```

If the files are `root:root 600`, consider enabling `rsyslog` to write plaintext files — it sets
`root:adm 640` by default, making `SupplementaryGroups=adm` effective. Alternatively, use the
[journald receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/journaldreceiver)
instead of the filelog receiver.
