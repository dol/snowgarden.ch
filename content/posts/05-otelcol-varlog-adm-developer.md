+++
title = "From /usr/adm to OpenTelemetry: Why Your Log Collector Can't Read /var/log (and How to Fix It)"
date = 2026-03-04
draft = true

[taxonomies]
tags = ["opentelemetry", "linux", "observability", "systemd", "otelcol", "logs", "linux-history", "sysadmin"]

[extra]
author = "Dominic Lüchinger"
+++

<!-- markdownlint-disable-next-line MD025 -->
# From /usr/adm to OpenTelemetry: Why Your Log Collector Can't Read /var/log (and How to Fix It)

You have just installed the OpenTelemetry Collector on a Linux machine. You have written a neat
configuration that tails `/var/log/syslog` to ship logs to your observability backend. You restart
the service, check the pipeline — and nothing arrives. Or worse: `permission denied`.

Before you reach for `chmod 777` or `sudo`, let me take you on a short journey through Unix history
that will not only solve your immediate problem, but explain a design decision made five decades ago
that is still shaping your infrastructure today.

## What Is the OpenTelemetry Collector?

The [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) (`otelcol`) is an open-source
agent that collects, processes, and exports telemetry data — metrics, traces, and logs. It is one of the
most popular observability agents in the Cloud Native ecosystem, maintained under the CNCF umbrella.

On Linux, you can install it as a DEB (Debian/Ubuntu) or RPM (RHEL/Fedora/Rocky) package. Once
installed, it runs as a background service managed by systemd.

## Setting Up the Filelog Receiver

The collector uses a pipeline model: **receivers** pull in data, **processors** transform it, and
**exporters** send it somewhere. To collect system logs from files, you use the
[filelog receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver):

```yaml
# /etc/otelcol/config.yaml
receivers:
  filelog:
    include:
      - /var/log/syslog
      - /var/log/auth.log

processors:
  batch:

exporters:
  otlp:
    endpoint: "my-backend:4317"

service:
  pipelines:
    logs:
      receivers: [filelog]
      processors: [batch]
      exporters: [otlp]
```

Save this, restart the collector with `sudo systemctl restart otelcol` — and encounter the wall.

## The Wall: Permission Denied

Check the collector's logs:

```bash
sudo journalctl -u otelcol -f
```

You will likely see something like:

```text
failed to open /var/log/syslog: open /var/log/syslog: permission denied
```

Or you will see the collector start successfully, but no log events will flow. Let us find out why.

### Who Is Running This Thing?

```bash
systemctl show otelcol | grep -E "^User|^Group"
# User=otel
# Group=otel
```

```bash
id otel
# uid=998(otel) gid=998(otel) groups=998(otel)
```

The collector runs as a locked-down system user called `otel`. It has no login shell, no home
directory, and belongs only to its own group. Now check the log file:

```bash
ls -l /var/log/syslog
# -rw-r----- 1 root adm 1234567 Mar  4 22:00 /var/log/syslog
```

The file is readable by group `adm` and nobody else outside of root. The `otel` user is not in `adm`.
The collector cannot read the file. Case closed — but the real question is: *why is it designed this way?*

---

## A Detour Through Unix History: The Story of `/usr/adm`

To understand the `adm` group, we need to go back to early Unix in the 1970s.

### The Original Unix Administrative Directory

Early Unix systems (Bell Labs, Version 7, the BSDs) maintained a directory called `/usr/adm`. It held
files that system administrators needed to monitor: `wtmp` (login records), `acct` (process accounting),
error logs, and similar files. Access to `/usr/adm` was controlled by the `adm` group — an unprivileged
group that let designated administrators read system data without requiring root.

The design philosophy was deliberate: operators who monitored logs should not need superuser access.
Give them just enough permission to read, no more.

### System V and the Move to `/var/adm`

As Unix commercialised through the 1980s (System V, HP-UX, Solaris, AIX), the Filesystem Hierarchy
Standard began to take shape. The growing convention separated variable data (frequently changing
files) from the `/usr` hierarchy. Logs moved from `/usr/adm` to `/var/adm`.

The group name `adm` and its purpose remained intact.

### Linux and the Arrival of `/var/log`

Linux, emerging in the early 1990s, adopted most of the Unix conventions but made one notable
change: logs live in `/var/log`, not `/var/adm`. The Filesystem Hierarchy Standard (FHS) formalised
this. But the `adm` group convention survived the path change.

Debian's documentation describes it plainly:

> *"Group `adm` is used for system monitoring tasks. Members of this group can read many log files
> in `/var/log`."*

Today, on a fresh Ubuntu or Debian install:

```text
/var/log/syslog          root:adm  640
/var/log/auth.log        root:adm  640
/var/log/kern.log        root:adm  640
/var/log/mail.log        root:adm  640
/var/log/dpkg.log        root:adm  640
```

On RHEL/Fedora/Rocky:

```text
/var/log/messages        root:adm  640   (when rsyslog is writing)
/var/log/secure          root:root 600   (some files more restricted)
```

The `adm` group is not for privilege escalation. It cannot run `sudo`, cannot modify files, cannot
touch system configuration. It is a least-privilege read gate for monitoring tools — exactly what an
OpenTelemetry Collector is.

---

## The Right Fix: A systemd Drop-in

Now that we understand the problem, the fix is elegant. We need to tell systemd to add `adm` as a
supplementary group for the `otelcol` service at runtime.

### What Is a systemd Drop-in?

When you install a package, it places a unit file in `/lib/systemd/system/`. This is package territory —
the package manager owns it and will overwrite it on upgrades. Operators must never edit these files
directly.

Instead, systemd supports **drop-in directories**: a folder named `<service>.service.d/` under
`/etc/systemd/system/` where you can place `.conf` files with overrides. systemd merges all drop-in
files with the base unit, and `/etc/systemd/system/` always takes precedence.

```text
/lib/systemd/system/otelcol.service     ← Package-managed. Never touch.
/etc/systemd/system/otelcol.service.d/  ← Your territory. Safe to modify.
  └── adm-group.conf                     ← Your drop-in.
```

### Creating the Drop-in

```bash
# Create the drop-in directory
sudo mkdir -p /etc/systemd/system/otelcol.service.d

# Write the drop-in
sudo tee /etc/systemd/system/otelcol.service.d/adm-group.conf <<'EOF'
[Service]
SupplementaryGroups=adm
EOF

# Reload systemd and restart the service
sudo systemctl daemon-reload
sudo systemctl restart otelcol
```

Or use the interactive editor, which handles the directory and file automatically:

```bash
sudo systemctl edit otelcol.service
```

Your editor opens a blank file pre-headed with the correct path. Add:

```ini
[Service]
SupplementaryGroups=adm
```

Save and close — systemd reloads automatically.

### What `SupplementaryGroups` Does

The `SupplementaryGroups=adm` directive adds `adm` to the list of supplementary groups of the process,
in addition to its primary group (`otel`). This happens at runtime, only for this service. The `otel`
user account itself is not modified. The `/etc/group` file is not modified.

If you run `id otel` in a shell, it will still show `groups=998(otel)` — because the group is added
by systemd when it starts the service, not by the OS user database.

---

## Verify the Fix

```bash
# Check that the drop-in is merged
sudo systemctl cat otelcol.service | grep -A1 "Drop-in"

# See the effective groups of the running process
PID=$(systemctl show -p MainPID --value otelcol)
echo "PID: $PID"
grep '^Groups:' /proc/$PID/status

# Functional test — check the service is reading logs without permission errors
# Note: sudo -u otel does NOT carry the supplementary group; it is only granted by systemd
sudo journalctl -u otelcol -n 20 | grep -i "filelog\|syslog\|error"
```

If there are no `permission denied` errors in the journal, the collector is reading `syslog`.
Restart the collector and watch log events begin to flow to your backend.

---

## Why Not the Simpler Alternatives?

You may wonder: why not just `sudo usermod -aG adm otel`? Or run the collector as root?

**Running as root** removes all the security guarantees the package's `User=otel` provides. A
vulnerability in the log parsing code could compromise the entire system. The whole point of a
dedicated non-privileged user is to limit that blast radius.

**`usermod -aG adm otel`** adds the group to the user account globally. This means any process
running as `otel` on the system — not just `otelcol` — could potentially read those log files.
It also does not survive a package reinstall that recreates the `otel` user. The drop-in is more
precise: it grants the permission to this specific service, and only when systemd launches it.

**Editing `/lib/systemd/system/otelcol.service`** directly works until you update the package, at
which point your changes are silently overwritten. Drop-ins were designed to solve exactly this
problem.

---

## Summary

| Concept | Key point |
|---------|-----------|
| `otel` user | Created by the package; non-privileged by design |
| `/var/log` permissions | `root:adm 640` — readable only by `adm` group members |
| `adm` group origin | Unix `/usr/adm` → System V `/var/adm` → Linux `/var/log`; convention preserved |
| The fix | systemd drop-in with `SupplementaryGroups=adm` |
| Why drop-in? | Package-safe, auditable, applies only to this service |

The `adm` group has been doing its quiet job for fifty years. Your OpenTelemetry Collector just needs
to be introduced.
