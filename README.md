# Trellis-WordFence-Cli

An Ansible role that installs, configures, and schedules [Wordfence CLI](https://github.com/wordfence/wordfence-cli) on [Roots Trellis](https://roots.io/trellis/)-managed WordPress servers.

- **Trellis** v1.31.0+
- **Wordfence CLI** v5.0.3+

## Features

- Installs Wordfence CLI from the precompiled binary (no Python/pip required)
- Writes a system-wide configuration file at `/etc/wordfence/wordfence-cli.ini`
- Schedules daily **malware scans** and **vulnerability scans** via cron with flock to prevent overlapping jobs
- Outputs scan results as CSV (configurable) to `/var/log/wordfence/`
- Optional email reporting via SMTP or sendmail
- Configures `logrotate` to manage scan log files

## Requirements

- Trellis ≥ 1.0 provisioned Ubuntu server (20.04, 22.04, or 24.04)
- Ansible ≥ 2.10
- A free or paid [Wordfence CLI license](https://www.wordfence.com/products/wordfence-cli/)

## Installation

Copy or symlink the `roles/wordfence` directory into your Trellis project:

```bash
cp -r roles/wordfence /path/to/your/trellis/roles/wordfence
```

Or reference this repository as a [Git submodule](https://git-scm.com/book/en/v2/Git-Tools-Submodules) and symlink the role.

## Configuration

### 1. Add your Wordfence license to Ansible Vault

In your Trellis project, add the license key to `group_vars/production/vault.yml` (encrypted with `ansible-vault`):

```yaml
vault_wordfence_license: "your-license-key-here"
```

### 2. Set role variables

Add any overrides to `group_vars/production/wordfence.yml` (or `group_vars/all/wordfence.yml`):

```yaml
# License is pulled from vault automatically:
wordfence_license: "{{ vault_wordfence_license }}"

# Paths to scan — defaults to Trellis www_root (/srv/www)
wordfence_malware_scan_paths:
  - /srv/www

wordfence_vuln_scan_paths:
  - /srv/www

# Cron schedule for malware scan (default: 02:00 daily)
wordfence_malware_scan_cron_hour: "2"
wordfence_malware_scan_cron_minute: "0"

# Cron schedule for vulnerability scan (default: 03:00 daily)
wordfence_vuln_scan_cron_hour: "3"
wordfence_vuln_scan_cron_minute: "0"

# Email reporting (optional)
wordfence_email: "admin@example.com"
wordfence_smtp_host: "smtp.example.com"
wordfence_smtp_user: "smtp-user@example.com"
# wordfence_smtp_password stored in vault as vault_wordfence_smtp_password
```

### 3. Run the playbook

```bash
# From within your Trellis directory:
ansible-playbook wordfence.yml -e env=production

# Install only:
ansible-playbook wordfence.yml -e env=production --tags wordfence-install

# Reconfigure only:
ansible-playbook wordfence.yml -e env=production --tags wordfence-configure

# Update cron schedule only:
ansible-playbook wordfence.yml -e env=production --tags wordfence-schedule
```

## Role Variables

All variables have sensible defaults defined in [`roles/wordfence/defaults/main.yml`](roles/wordfence/defaults/main.yml).

| Variable | Default | Description |
|---|---|---|
| `wordfence_version` | `5.0.3` | Wordfence CLI version to install |
| `wordfence_license` | `""` | License key (use vault) |
| `wordfence_user` | `root` | System user that runs scans |
| `wordfence_bin` | `/usr/local/bin/wordfence` | Path to the wordfence executable |
| `wordfence_config_dir` | `/etc/wordfence` | Configuration directory |
| `wordfence_config_file` | `/etc/wordfence/wordfence-cli.ini` | Configuration file path |
| `wordfence_cache_dir` | `/var/cache/wordfence` | Cache directory |
| `wordfence_lock_dir` | `/var/lock/wordfence` | flock lock file directory |
| `wordfence_log_dir` | `/var/log/wordfence` | Log/output directory |
| `wordfence_malware_scan_enabled` | `true` | Enable malware scan cron job |
| `wordfence_malware_scan_paths` | `[/srv/www]` | Paths to scan for malware |
| `wordfence_malware_scan_cron_minute` | `0` | Cron minute for malware scan |
| `wordfence_malware_scan_cron_hour` | `2` | Cron hour for malware scan |
| `wordfence_malware_scan_output_format` | `csv` | Output format: `csv`, `json`, `human` |
| `wordfence_malware_scan_output_path` | `/var/log/wordfence/malware-scan.csv` | Output file path |
| `wordfence_vuln_scan_enabled` | `true` | Enable vulnerability scan cron job |
| `wordfence_vuln_scan_paths` | `[/srv/www]` | Paths to scan for vulnerabilities |
| `wordfence_vuln_scan_cron_minute` | `0` | Cron minute for vuln scan |
| `wordfence_vuln_scan_cron_hour` | `3` | Cron hour for vuln scan |
| `wordfence_vuln_scan_output_format` | `csv` | Output format: `csv`, `json`, `human` |
| `wordfence_vuln_scan_output_path` | `/var/log/wordfence/vuln-scan.csv` | Output file path |
| `wordfence_email` | `""` | Email address(es) for reports |
| `wordfence_email_from` | `""` | From address for email reports |
| `wordfence_smtp_host` | `""` | SMTP hostname (blank = use sendmail) |
| `wordfence_smtp_port` | `587` | SMTP port |
| `wordfence_smtp_tls_mode` | `starttls` | TLS mode: `none`, `smtps`, `starttls` |
| `wordfence_smtp_user` | `""` | SMTP username |
| `wordfence_smtp_password` | `""` | SMTP password (use vault) |
| `wordfence_logrotate_enabled` | `true` | Enable logrotate configuration |
| `wordfence_logrotate_rotate` | `14` | Number of rotations to retain |

## Role Structure

```
roles/wordfence/
├── defaults/
│   └── main.yml              # Default variable values
├── handlers/
│   └── main.yml              # (empty — no daemons to restart)
├── meta/
│   └── main.yml              # Galaxy metadata
├── tasks/
│   ├── main.yml              # Entry point — imports sub-tasks
│   ├── install.yml           # Install Wordfence CLI and dependencies
│   ├── configure.yml         # Write config file and set up directories
│   └── schedule.yml          # Set up cron jobs
└── templates/
    ├── wordfence-cli.ini.j2  # Wordfence CLI INI configuration
    └── wordfence-logrotate.j2 # Logrotate configuration
```

## Cron Jobs

The role installs two cron jobs for the configured user (default: `root`):

**Malware scan** (default: daily at 02:00)
```
0 2 * * *  root  /usr/bin/flock -w 0 /var/lock/wordfence/malware-scan.lock \
  /usr/local/bin/wordfence malware-scan \
  --configuration /etc/wordfence/wordfence-cli.ini \
  --no-banner --no-color \
  /srv/www >> /var/log/wordfence/malware-scan.log 2>&1
```

**Vulnerability scan** (default: daily at 03:00)
```
0 3 * * *  root  /usr/bin/flock -w 0 /var/lock/wordfence/vuln-scan.lock \
  /usr/local/bin/wordfence vuln-scan \
  --configuration /etc/wordfence/wordfence-cli.ini \
  --no-banner --no-color \
  /srv/www >> /var/log/wordfence/vuln-scan.log 2>&1
```

`flock` prevents duplicate scans from running concurrently.

## Running a Scan Manually

After provisioning, you can trigger a scan on-demand:

```bash
# SSH into the server
sudo wordfence malware-scan --configuration /etc/wordfence/wordfence-cli.ini /srv/www

# Vulnerability scan
sudo wordfence vuln-scan --configuration /etc/wordfence/wordfence-cli.ini /srv/www
```

## License

MIT