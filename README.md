# Trellis-WordFence-Cli

An Ansible role that installs, configures, and schedules [Wordfence CLI](https://github.com/wordfence/wordfence-cli) on [Roots Trellis](https://roots.io/trellis/)-managed WordPress servers.

## Features

- Installs Wordfence CLI from the precompiled binary (no Python/pip required)
- Verifies the official binary against its pinned SHA-256 checksum
- Installs a pinned, verified AWS CLI v2 release
- Writes a system-wide configuration file at `/etc/wordfence/wordfence-cli.ini`
- Schedules daily **malware scans** and **vulnerability scans** as systemd
  timers, one service per scan type so same-type runs cannot overlap
- Runs scans unprivileged as Trellis's `web_user` with reduced CPU and I/O
  priority and a hardened systemd sandbox
- Uploads CSV results to an existing S3 bucket; run output goes to journald
- Optionally reports each run to an Uptime Kuma push monitor
- Accepts the Wordfence CLI license terms for noninteractive scheduled scans

## Requirements

- Trellis-managed Ubuntu server with systemd
- Ansible ≥ 2.10
- A free or paid [Wordfence CLI license](https://www.wordfence.com/products/wordfence-cli/)
- An existing S3 bucket and an EC2 instance profile that can write objects
  under the configured prefix

Provisioning this role causes scheduled and manually invoked runner scans to
pass Wordfence CLI's `--accept-terms` option. Review the
[Wordfence CLI license terms](https://www.wordfence.com/wordfence-cli-license-terms-and-conditions/)
before deploying the role.

## Wire into Trellis

Trellis provisions custom roles from the main play in `server.yml`. Add this
role there so it runs automatically during normal staging and production
provisioning.

### 1. Add the role to `galaxy.yml`

Add the private Git repository under the existing `roles:` key in Trellis's
`galaxy.yml`:

```yaml
roles:
  # Existing Trellis roles...
  - name: wordfence
    src: git@github.com:spark451inc/Trellis-WordFence-Cli.git
    scm: git
    version: v2.0.0
```

Pin `version` to an existing release tag. The machine running Trellis or
Ansible must have SSH access to the private GitHub repository.
`trellis provision` installs entries from `galaxy.yml` automatically.

### 2. Add the role to `server.yml`

In the main `WordPress Server` play, append Wordfence to the existing `roles`
list:

```yaml
roles:
  # Existing Trellis roles...
  - { role: wordpress-setup, tags: [wordpress, wordpress-setup, letsencrypt] }
  - { role: wordfence, tags: [wordfence] }
```

Trellis upgrades may change `server.yml`; preserve this role entry when merging
upstream changes.

### 3. Configure each environment

Add the license, and optionally the Uptime Kuma push URLs, to each
environment's encrypted vault file:

```yaml
# group_vars/production/vault.yml
vault_wordfence_license: "your-license-key-here"

# Optional; one push monitor per scan type per environment
vault_wordfence_vuln_scan_uptime_kuma_push_url: "https://kuma.example.com/api/push/abc123"
vault_wordfence_malware_scan_uptime_kuma_push_url: "https://kuma.example.com/api/push/def456"
```

Repeat this in `group_vars/staging/vault.yml` with that environment's own
values. Then configure the existing S3 bucket in both
`group_vars/production/wordfence.yml` and `group_vars/staging/wordfence.yml`:

```yaml
# Existing destination bucket; do not include s3://
wordfence_s3_bucket: example-security-reports
wordfence_s3_prefix: wordfence

# systemd calendar expressions (defaults: 01:00 and 02:00 daily)
wordfence_vuln_scan_on_calendar: "*-*-* 01:00:00"
wordfence_malware_scan_on_calendar: "*-*-* 02:00:00"

# Optional Uptime Kuma push monitors; omit or leave empty to disable
wordfence_vuln_scan_uptime_kuma_push_url: "{{ vault_wordfence_vuln_scan_uptime_kuma_push_url }}"
wordfence_malware_scan_uptime_kuma_push_url: "{{ vault_wordfence_malware_scan_uptime_kuma_push_url }}"
```

Values shared by staging and production can instead live in
`group_vars/all/wordfence.yml`, but Uptime Kuma push URLs identify a single
monitor, so keep them per environment. Provisioning stops before configuration
if the resolved environment license is blank.

### 4. Provision Trellis

Normal Trellis provisioning now runs the Wordfence role automatically:

```bash
trellis provision staging
trellis provision production

# Run only this role:
trellis provision --tags wordfence staging
trellis provision --tags wordfence production
```

The role also tags its phases individually as `wordfence-install`,
`wordfence-configure`, and `wordfence-schedule`, so for example
`--tags wordfence-schedule` re-renders only the systemd units and timers.

Without Trellis CLI, run the same `server.yml` play directly:

```bash
ansible-galaxy install --force -r galaxy.yml
ansible-playbook server.yml -e env=staging
ansible-playbook server.yml -e env=production
```

The Trellis `env` value is embedded in the managed runner; it separates S3
objects by environment and is included in Uptime Kuma messages. Provision
staging with `-e env=staging` and production with `-e env=production`.

## Role Variables

Defaults are defined in [`defaults/main.yml`](defaults/main.yml).

| Variable | Default | Description |
|---|---|---|
| `wordfence_version` | `5.0.4` | Wordfence CLI version to install |
| `wordfence_checksums` | Architecture-specific SHA-256 values | Checksums published with the pinned release |
| `wordfence_aws_cli_version` | `2.36.30` | AWS CLI v2 version to install |
| `wordfence_aws_cli_checksums` | Architecture-specific SHA-256 values | Checksums verified against AWS-signed installers |
| `wordfence_license` | `""` | License key (use vault) |
| `wordfence_s3_bucket` | `""` | Existing destination bucket; required |
| `wordfence_s3_prefix` | `wordfence` | Object-key prefix; surrounding slashes are removed |
| `wordfence_vuln_scan_enabled` | `true` | Enable the vulnerability scan timer |
| `wordfence_vuln_scan_on_calendar` | `*-*-* 01:00:00` | systemd `OnCalendar` expression for the vulnerability scan |
| `wordfence_vuln_scan_uptime_kuma_push_url` | `""` | Optional Uptime Kuma push URL for the vulnerability scan |
| `wordfence_malware_scan_enabled` | `true` | Enable the malware scan timer |
| `wordfence_malware_scan_on_calendar` | `*-*-* 02:00:00` | systemd `OnCalendar` expression for the malware scan |
| `wordfence_malware_scan_uptime_kuma_push_url` | `""` | Optional Uptime Kuma push URL for the malware scan |

Schedules use [systemd calendar syntax](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html#Calendar%20Events);
check an expression with `systemd-analyze calendar '<expression>'`. An invalid
expression fails provisioning when the timer is started.

## Scheduled Scans

Scan paths are derived from Trellis's `wordpress_sites` inventory, including
each site's supported `current_path` and `public_path` overrides. With Trellis
defaults, malware scans cover `/srv/www/<site>/current/web`; vulnerability
scans target `/srv/www/<site>/current/web/wp`. Wordfence automatically
recognizes Bedrock's adjacent `web/app` content directory.

The runner skips inventory sites that do not yet have their active deployment
path and logs each skipped path. If no deployed sites remain, the run fails
with status 66 and uploads nothing.

Vulnerability scans run daily at 01:00 and malware scans at 02:00 by default.
Each scan type is its own systemd service and timer:

| Scan | Service | Timer |
|---|---|---|
| Vulnerability | `wordfence-vuln-scan.service` | `wordfence-vuln-scan.timer` |
| Malware | `wordfence-malware-scan.service` | `wordfence-malware-scan.timer` |

systemd never starts a second instance of a service that is still running, so
same-type scans cannot overlap; malware and vulnerability scans may run
concurrently. Timers use `Persistent=true`, so a run missed while the server
was off starts at the next boot. Each service has a 20 hour `TimeoutStartSec`
so a wedged scan is killed before the next day's run. Provisioning only
enables, starts, or restarts the timers and never starts a service. Disabling a
scan type stops and disables its timer but leaves the service installed for
manual runs.

Each service runs as `web_user:web_group` with `Nice=10`, best-effort I/O
priority 7, a private temporary directory, and a read-only system view except
for `/var/cache/wordfence`. The Wordfence license in
`/etc/wordfence/wordfence-cli.ini` is therefore readable by `web_group`, the
same group PHP-FPM runs as.

Each scan writes its CSV results to a temporary directory and uploads them as
the success marker. Run output, including Wordfence CLI's own messages, goes to
journald.

```text
wordfence/<environment>/<scan-type>/YYYY/MM/DD/<timestamp>.csv
```

This environment-only layout assumes one scanning host per environment.
Multiple hosts in the same environment can generate the same timestamped key.

Scheduled scan reports use CSV so every successful results object has a stable
format and `.csv` extension.

## Running a Scan Manually

Start the service rather than the runner script so the run is serialized and
logged like a scheduled one. `systemctl start` blocks until the scan finishes;
add `--no-block` to return immediately.

```bash
sudo systemctl start wordfence-malware-scan.service
sudo journalctl -fu wordfence-malware-scan.service
```

Inspect schedules and recent runs with:

```bash
sudo systemctl list-timers 'wordfence-*'
sudo journalctl -u wordfence-vuln-scan.service -u wordfence-malware-scan.service
```

## Monitoring with Uptime Kuma

When a push URL is configured, the runner reports `up` only after the scan
succeeded and its CSV was uploaded, and `down` with the exit status and reason
for any other outcome. Every push includes `ping=` with the run duration in
milliseconds, so the monitor's response-time graph tracks how long each scan
takes. Findings in the CSV do not affect the status; Kuma tracks whether the
scan ran, not what it found. The URL may be pasted exactly as Uptime Kuma
displays it; any query string is replaced. Pushes use `curl`, which Trellis
installs by default.

Create one push monitor per scan type per environment. Set each heartbeat
interval to 24 hours plus the observed maximum run time; vulnerability scans
usually finish in minutes, while a malware scan on a host with many sites can
take several hours. A missed heartbeat catches failures the runner cannot
report, such as a disabled timer or an unreachable host.

## Upgrading from 1.x

Version 2.0.0 replaces root cron jobs with systemd timers and is a breaking
change:

- `wordfence_*_cron_hour` and `wordfence_*_cron_minute` are replaced by
  `wordfence_*_on_calendar`.
- Scans run as `web_user` instead of root. `/var/cache/wordfence` changes
  owner, and `/etc/wordfence` and its INI become group-readable by
  `web_group`.
- Diagnostic logs are no longer uploaded to S3, and the `failed/` prefix is no
  longer written. Use journald instead.
- Run scans manually with `systemctl start`, not by invoking the runner script.
- Optional Uptime Kuma reporting is new; existing installs are unaffected
  unless a push URL is set.

Provisioning 2.x removes the 1.x cron entries and lock files and migrates
ownership of the cache contents. Provision every host with a 2.x release before
upgrading to 3.0.0, which drops that cleanup.

## License

[GPL-2.0-or-later](LICENSE)
