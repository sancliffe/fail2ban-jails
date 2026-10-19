# fail2ban-jails

Custom [Fail2ban](https://github.com/fail2ban/fail2ban) jail configuration for a Debian/Ubuntu server running Apache, SSH and a Postfix/Dovecot mail stack. Bans are enforced through UFW, and repeat offenders can be reported to [AbuseIPDB](https://www.abuseipdb.com/).

## Contents

```
.
├── jail.local                        # Global defaults (ban times, whitelist, ban action)
├── jail.d/
│   ├── apache.conf                   # Apache access-log jails for scanners and probes
│   ├── defaults-debian.conf          # Enables the sshd jail (Debian default)
│   ├── recidive.conf                 # Longer bans for repeat offenders + AbuseIPDB reporting
│   └── services.conf                 # sshd, dovecot and postfix jails
└── filter.d/
    ├── apache-badstatus.conf         # Custom: 400/408 responses
    ├── apache-connect-probe.conf     # Custom: CONNECT proxy probes
    ├── apache-env.conf               # Custom: requests for .env files
    ├── apache-php-scan.conf          # Custom: EMPTY, needs a failregex
    ├── apache-scanner.conf           # Custom: repeated 404s
    ├── apache-web-stratum.conf       # Custom: EMPTY, and name doesn't match the jail (see below)
    ├── dovecot.conf                  # Copy of the stock Fail2ban filter
    ├── postfix.conf                  # Copy of the stock Fail2ban filter
    ├── recidive.conf                 # Copy of the stock Fail2ban filter
    └── sshd.conf                     # Copy of the stock Fail2ban filter
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
   sudo cp filter.d/*.conf /etc/fail2ban/filter.d/
   ```

   The copies of `dovecot`, `postfix`, `recidive` and `sshd` in `filter.d/` will override the ones shipped with your Fail2ban package. If you'd rather keep the package versions, copy only the `apache-*` filters.

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

Each jail needs a matching `/etc/fail2ban/filter.d/<name>.conf` file. The Apache filters in `filter.d/` match the access log as follows:

| Filter | What it matches |
| --- | --- |
| `apache-env` | `GET`/`POST`/`HEAD` requests for paths containing `.env` that return 400, 403 or 404 |
| `apache-connect-probe` | `CONNECT` requests (open-proxy probing) |
| `apache-badstatus` | Responses with status 400 or 408 |
| `apache-scanner` | Any `GET`/`POST`/`HEAD` that returns 404 |

The `dovecot`, `postfix`, `recidive` and `sshd` filters are copies of the ones shipped with Fail2ban.

### Known issues

- **`apache-php-scan` is empty.** `filter.d/apache-php-scan.conf` has no content, so it has no `failregex` and the `apache-php-scan` jail will fail to start. Add a `[Definition]` section with a `failregex`, or set `enabled = false` for that jail.
- **`apache-web-stratum` points at the wrong filter name.** The jail sets `filter = web-stratum`, but the file is `filter.d/apache-web-stratum.conf` (which is also empty). Rename the file or change the jail's `filter` line so they match, and add a `failregex`.
- **`apache-scanner` is strict.** It bans after 5 requests that return 404 within 60 seconds, so a legitimate client hitting a few broken links can be banned for 24 hours.

Because a jail with a broken filter can stop Fail2ban from loading, run `sudo fail2ban-client -t` after installing. Test filters against your logs before enabling:

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