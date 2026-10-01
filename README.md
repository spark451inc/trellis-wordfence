# trellis-wordfence

> **Wordfence CLI is end-of-life.** Version 3.0.0 is the final release of this
> role. It no longer installs or schedules anything; it removes everything
> earlier releases put on a server. Run it once on every host, then remove the
> role from Trellis.

An Ansible role for [Roots Trellis](https://roots.io/trellis/)-managed
WordPress servers. Releases 1.x and 2.x installed
[Wordfence CLI](https://github.com/wordfence/wordfence-cli), scheduled scans,
and uploaded reports to S3. This independent integration by
[Spark451](https://www.spark451.com/) is not an official Roots or Wordfence
project.

## What 3.0.0 Removes

Every task is idempotent and a no-op where nothing is installed, so it is safe
on any host, including ones upgrading directly from 1.x or pre-release builds.

- The `wordfence-vuln-scan` and `wordfence-malware-scan` systemd timers and
  services: stopped, disabled, deleted, failed state cleared, and their
  `Persistent=true` timer stamps removed. A scan that is running is stopped.
- The 1.x root cron jobs `Wordfence CLI malware scan` and
  `Wordfence CLI vulnerability scan`, plus any scan they started that is still
  running (sent `SIGTERM`, then `SIGKILL` after 30 seconds)
- `/usr/local/bin/wordfence` and the `/usr/local/sbin/wordfence-scan-to-s3`
  runner
- `/etc/wordfence` (including the license key), `/var/cache/wordfence`, and the
  pre-1.0 `/var/log/wordfence`, `/var/lock/wordfence`, and
  `/etc/logrotate.d/wordfence`
- `~/.config/wordfence` and `~/.cache/wordfence` for root and every `/home`
  user, which manual `wordfence` runs can create with the license key inside
- Interrupted-install leftovers in `/usr/local/bin` and `/tmp`, and a pre-1.0
  `wordfence` APT package or `pip3`-installed `wordfence` if present (its
  Python dependencies are left in place because other software may share them)
- **AWS CLI v2** (`/usr/local/bin/aws`, `/usr/local/bin/aws_completer`,
  `/usr/local/aws-cli`), unconditionally. If anything else on the server uses
  it, install it separately after decommissioning.
- `libpcre3`, only if APT confirms nothing else depends on it; otherwise it is
  marked automatically installed so a later `apt autoremove` can clean it up

The role takes no variables and no longer needs a license, S3 bucket, or any
Trellis variables.

### Not Removed

- `unzip`, which 2.x also installed but Composer uses during Bedrock deploys
- Past scan output in journald, which ages out under the journal's retention
  policy
- Anything off the server; see [Clean Up Elsewhere](#4-clean-up-elsewhere)

## Decommissioning

### 1. Pin 3.0.0 in `galaxy.yml`

Keep the existing source and local alias `name: wordfence`; change only the
version:

```yaml
roles:
  - name: wordfence
    src: spark451inc.trellis_wordfence
    version: v3.0.0
```

or, from GitHub:

```yaml
roles:
  - name: wordfence
    src: https://github.com/spark451inc/trellis-wordfence.git
    scm: git
    version: v3.0.0
```

Leave `{ role: wordfence, tags: [wordfence] }` in `server.yml` for now.

### 2. Provision every environment

```bash
trellis provision --tags wordfence staging
trellis provision --tags wordfence production
```

Without Trellis CLI:

```bash
ansible-galaxy install --force -r galaxy.yml
ansible-playbook server.yml -e env=staging --tags wordfence
ansible-playbook server.yml -e env=production --tags wordfence
```

The 2.x phase tags (`wordfence-install`, `wordfence-configure`,
`wordfence-schedule`) no longer exist; use `wordfence`. Include every host
that ever ran this role, including development or retired-but-running hosts.

To confirm, these should print nothing:

```bash
systemctl list-units --all 'wordfence*' --no-legend
sudo crontab -l 2>/dev/null | grep -i wordfence
sudo find / -xdev -iname '*wordfence*' 2>/dev/null
```

### 3. Remove the role from Trellis

After every host has been provisioned with 3.0.0:

- Delete the `wordfence` entry from `server.yml` and `galaxy.yml`.
- Delete `group_vars/*/wordfence.yml` and the `vault_wordfence_license` and
  `vault_wordfence_*_uptime_kuma_push_url` entries from each environment's
  `vault.yml`.

### 4. Clean up elsewhere

The role cannot reach these:

- **Uptime Kuma:** delete or pause the Wordfence push monitors, or they will
  start alerting once heartbeats stop.
- **S3 and IAM:** decide whether to keep or delete the reports under the
  `wordfence/` prefix, and remove the instance-profile permission to write
  there.
- **Wordfence license:** cancel or revoke it in your Wordfence account.

## License

Copyright (C) 2026 JenSpark, Inc. d/b/a Spark451

This role is free software: you can redistribute it and/or modify it under
the terms of the GNU General Public License as published by the Free Software
Foundation, either version 2 of the License, or (at your option) any later
version. See [LICENSE](LICENSE). SPDX identifier: `GPL-2.0-or-later`.

This role is distributed in the hope that it will be useful, but WITHOUT ANY
WARRANTY; without even the implied warranty of MERCHANTABILITY or FITNESS FOR
A PARTICULAR PURPOSE. See the GNU General Public License for more details.

This license covers the role's source code and documentation. Wordfence CLI,
AWS CLI, and Trellis retain their own licenses; their code and binaries are
not bundled in this repository.
