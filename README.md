# Trellis-WordFence-Cli

An Ansible role that installs, configures, and schedules [Wordfence CLI](https://github.com/wordfence/wordfence-cli) on [Roots Trellis](https://roots.io/trellis/)-managed WordPress servers.

## Features

- Installs Wordfence CLI from the precompiled binary (no Python/pip required)
- Verifies the official binary against its pinned SHA-256 checksum
- Installs a pinned, verified AWS CLI v2 release
- Writes a system-wide configuration file at `/etc/wordfence/wordfence-cli.ini`
- Schedules daily **malware scans** and **vulnerability scans** with a shared lock
- Uploads CSV results and diagnostic logs to an existing S3 bucket
- Accepts the Wordfence CLI license terms for noninteractive scheduled scans

## Requirements

- Trellis-managed Ubuntu server
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
    version: v1.0.0
```

The example assumes the first release is published as `v1.0.0`; pin `version`
to an existing release tag. The machine running Trellis or Ansible must have
SSH access to the private GitHub repository. `trellis provision` installs
entries from `galaxy.yml` automatically.

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

Add the license to each environment's encrypted vault file:

```yaml
# group_vars/production/vault.yml
vault_wordfence_license: "your-license-key-here"
```

Repeat this in `group_vars/staging/vault.yml`. Then configure the existing S3
bucket in both `group_vars/production/wordfence.yml` and
`group_vars/staging/wordfence.yml`:

```yaml
# Existing destination bucket; do not include s3://
wordfence_s3_bucket: example-security-reports
wordfence_s3_prefix: wordfence

# Cron schedule for malware scan (default: 02:00 daily)
wordfence_malware_scan_cron_hour: "2"
wordfence_malware_scan_cron_minute: "0"

# Cron schedule for vulnerability scan (default: 03:00 daily)
wordfence_vuln_scan_cron_hour: "3"
wordfence_vuln_scan_cron_minute: "0"
```

Values shared by staging and production can instead live in
`group_vars/all/wordfence.yml`. Provisioning stops before configuration if the
resolved environment license is blank.

### 4. Provision Trellis

Normal Trellis provisioning now runs the Wordfence role automatically:

```bash
trellis provision staging
trellis provision production

# Run only this role:
trellis provision --tags wordfence staging
trellis provision --tags wordfence production
```

Without Trellis CLI, run the same `server.yml` play directly:

```bash
ansible-galaxy install --force -r galaxy.yml
ansible-playbook server.yml -e env=staging
ansible-playbook server.yml -e env=production
```

The Trellis `env` value is embedded in the managed runner and separates S3
objects by environment. Provision staging with `-e env=staging` and production
with `-e env=production`.

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
| `wordfence_malware_scan_enabled` | `true` | Enable malware scan cron job |
| `wordfence_malware_scan_cron_minute` | `0` | Cron minute for malware scan |
| `wordfence_malware_scan_cron_hour` | `2` | Cron hour for malware scan |
| `wordfence_vuln_scan_enabled` | `true` | Enable vulnerability scan cron job |
| `wordfence_vuln_scan_cron_minute` | `0` | Cron minute for vuln scan |
| `wordfence_vuln_scan_cron_hour` | `3` | Cron hour for vuln scan |

## Scheduled Scans

Scan paths are derived from Trellis's `wordpress_sites` inventory, including
each site's supported `current_path` and `public_path` overrides. With Trellis
defaults, malware scans cover `/srv/www/<site>/current/web`; vulnerability
scans target `/srv/www/<site>/current/web/wp`. Wordfence automatically
recognizes Bedrock's adjacent `web/app` content directory and fails incomplete
scans into the S3 diagnostic path.

The runner skips inventory sites that do not yet have their active deployment
path and records each skipped path in the diagnostic log. If no deployed sites
remain, the run fails and uploads only a failure diagnostic.

Malware scans run daily at 02:00 and vulnerability scans at 03:00 by default.
A shared nonblocking lock protects scheduled and manual scans. If another scan
holds the lock, the invocation fails immediately with status 75 and uploads a
failure diagnostic instead of queuing.

Each scan writes results and diagnostics to a temporary directory, uploads the
diagnostic log, then uploads the CSV as the success marker. Failed scans upload
only their diagnostic log under `failed/`.

```text
wordfence/<environment>/<scan-type>/YYYY/MM/DD/<timestamp>.csv
wordfence/<environment>/<scan-type>/YYYY/MM/DD/<timestamp>.log
wordfence/<environment>/failed/<scan-type>/YYYY/MM/DD/<timestamp>-scan-<status>.log
```

This environment-only layout assumes one scanning host per environment.
Multiple hosts in the same environment can generate the same timestamped key.

Scheduled scan reports use CSV so every successful results object has a stable
format and `.csv` extension.

## Running a Scan Manually

After provisioning, you can trigger a scan on-demand:

```bash
sudo /usr/local/sbin/wordfence-scan-to-s3 malware-scan
sudo /usr/local/sbin/wordfence-scan-to-s3 vuln-scan
```

## License

[GPL-2.0-or-later](LICENSE)
