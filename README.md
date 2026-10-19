# fail2ban-jails

Custom [Fail2ban](https://github.com/fail2ban/fail2ban) jail configuration for a Debian/Ubuntu server running Apache, SSH and a Postfix/Dovecot mail stack. Bans are enforced through UFW, and repeat offenders can be reported to [AbuseIPDB](https://www.abuseipdb.com/).

## Contents

```
.
├── jail.local              # Global defaults (ban times, whitelist, ban action)
└── jail.d/
    ├── apache.conf         # Apache access-log jails for scanners and probes
    ├── defaults-debian.conf# Enables the sshd jail (Debian default)
    ├── recidive.conf       # Longer bans for repeat offenders + AbuseIPDB reporting
    └── services.conf       # sshd, dovecot and postfix jails
```

## Defaults (`jail.local`)

| Setting | Value | Notes |
| --- | --- | --- |
| `bantime` | `15m` | Default ban length, overridden per jail |
| `findtime` | `30m` | Window in which failures are counted |
| `maxretry` | `3` | Failures before a ban |
| `banaction` | `ufw` | Bans are applied as UFW rules |
| `ignoreip` | localhost + trusted IPs | Never banned; prevents accidental lockouts |

AbuseIPDB reporting is intentionally not part of the global `action`, to avoid API rate-limit errors. It is only used in the `recidive` jail.

## Jails

### Apache (`jail.d/apache.conf`)

All jails read `/var/log/apache2/access.log` and watch ports `http,https`.

| Jail | Filter | maxretry | findtime | bantime |
| --- | --- | --- | --- | --- |
| `apache-env` | `apache-env` | 1 | 30m | 24h |
| `apache-connect-probe` | `apache-connect-probe` | 1 | 1h | 24h |
| `apache-badstatus` | `apache-badstatus` | 2 | 1h | 24h |
| `apache-php-scan` | `apache-php-scan` | 1 | 60m | 24h |
| `apache-web-stratum` | `web-stratum` | 1 | 1h | 24h |
| `apache-scanner` | `apache-scanner` | 5 | 60s | 86400s (24h) |

### SSH, mail (`jail.d/services.conf`)

| Jail | Ports | Log | maxretry | findtime | bantime |
| --- | --- | --- | --- | --- | --- |
| `sshd` | default | default | 3 | 10m | 1h |
| `dovecot` | `pop3,pop3s,imap,imaps,submission,sieve` | `/var/log/mail.log` | 3 | 10m | 1h |
| `postfix` | `smtp,ssmtp,submission,submissions` | `/var/log/mail.log` | 3 | 45m | 24h (`mode = aggressive`) |

`jail.d/defaults-debian.conf` also enables `sshd`. The two files overlap, and `services.conf` supplies the tuned values.

### Repeat offenders (`jail.d/recidive.conf`)

The `recidive` jail watches `/var/log/fail2ban.log`. An IP banned twice within 3 days is banned for 7 days. These bans are also reported to AbuseIPDB (categories 18 and 22: brute-force and SSH).

## Requirements

- Fail2ban (0.10 or newer recommended)
- UFW, enabled and running
- Apache with logs at `/var/log/apache2/access.log`
- Postfix and Dovecot logging to `/var/log/mail.log` (if you use those jails)
- An [AbuseIPDB](https://www.abuseipdb.com/) API key (optional, for `recidive` reporting)

## Installation

1. Clone the repository:

   ```sh
   git clone https://github.com/sancliffe/fail2ban-jails.git
   cd fail2ban-jails
   ```

2. Back up your existing configuration:

   ```sh
   sudo cp -a /etc/fail2ban /etc/fail2ban.bak
   ```

3. Copy the files into place:

   ```sh
   sudo cp jail.local /etc/fail2ban/jail.local
   sudo cp jail.d/*.conf /etc/fail2ban/jail.d/
   ```

4. Edit the configuration (see below), then validate and reload:

   ```sh
   sudo fail2ban-client -t
   sudo systemctl restart fail2ban
   ```

5. Confirm the jails are running:

   ```sh
   sudo fail2ban-client status
   sudo fail2ban-client status apache-scanner
   ```

## Configuration you must change

- **`ignoreip` in `jail.local`:** replace the listed addresses with your own trusted IPs (home, office, mobile). Leaving someone else's IPs whitelisted exempts them from bans.
- **AbuseIPDB key in `jail.d/recidive.conf`:** replace `REMOVED-UPDATE-ME` with your API key, or remove the `action_abuseipdb` line to disable reporting. Don't commit a real key to a public repository.
- **Log paths:** adjust `logpath` if your Apache or mail logs live elsewhere (for example, vhost-specific logs).

## Custom filters

The Apache jails reference filters that are **not included in this repository**:

- `apache-env`
- `apache-connect-probe`
- `apache-badstatus`
- `apache-php-scan`
- `web-stratum`
- `apache-scanner`

Each jail needs a matching `/etc/fail2ban/filter.d/<name>.conf` file. Without it, Fail2ban will refuse to start those jails. Add your filter definitions to `filter.d/` and test them against your logs before enabling:

```sh
sudo fail2ban-regex /var/log/apache2/access.log /etc/fail2ban/filter.d/apache-scanner.conf
```

## Tips

- Unban an IP: `sudo fail2ban-client set <jail> unbanip <ip>`
- List banned IPs for a jail: `sudo fail2ban-client status <jail>`
- Watch activity: `sudo tail -f /var/log/fail2ban.log`
- Aggressive jails (`maxretry = 1`) can ban legitimate users and crawlers. Check your `ignoreip` list and test filters with `fail2ban-regex` before going live.

## License

Licensed under the [Apache License 2.0](LICENSE).