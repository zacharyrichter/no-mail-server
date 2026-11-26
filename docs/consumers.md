# What sends through it

## Through the shim

| Sender | How it calls the shim |
|---|---|
| unattended-upgrades | A full message on stdin, subject from the `Subject:` header. Configured by `51unattended-upgrades-mail` ([fleet-standard.md](fleet-standard.md#unattended-upgrades)) |
| cron | Output as a mail message on stdin, through `/usr/sbin/sendmail` |
| The monthly certificate expiry check | `ntfy-sendmail -s "Host certs: <result>"` with the report as the body. See [expire-on-purpose](https://github.com/zacharyrichter/expire-on-purpose) |
| The Traefik update timer | A one-line function, `printf '%s\n' "$2" \| ntfy-sendmail -s "$1" \|\| true`. It notifies only when something changed or failed, and the `\|\| true` means a notification problem can never fail the update |

The expiry check shows the pattern for a service that must stay unprivileged. The check runs as the admin user, so it can't read the root-only env file. Its systemd unit sends the notification from `ExecStopPost=+...`, where the `+` runs only that one step as root. It runs on success **and** failure, so a crashed check still produces a message.

## Proxmox and the backup server

Proxmox VE and Proxmox Backup Server don't call `sendmail` in the way the other hosts do. They have a native **webhook** notification target, which is simpler than SMTP and needs no listener, so they use it, configured the same way on each:

- POST to `https://ntfy.example.net/proxmox`
- header `Authorization: Bearer {{ secrets.token }}`, with the token in the target's secrets field instead of in plain text
- body `[{{ fields.hostname }}] {{ title }}: {{ message }}`

The existing notification matchers were repointed from the old sendmail target to the webhook. Each was confirmed with a real backup event, not the generic Test button, because the Test button proves the target exists and a real event proves the matcher reaches it.

## A sender that can only speak mail

The router's monitoring (Monit on OPNsense) can send mail but nothing else, and the router is managed through its GUI only: no custom scripts on it. So it uses the ntfy server's own SMTP listener, which listens on a non-standard port and turns the address into a topic with a prefix (`ntfy-opnsense@ntfy.example.net` publishes to the `opnsense` topic). That needed no new infrastructure: the listener had been built for Proxmox and was unused once Proxmox moved to webhooks.

Monit is the only place in this repo with a regex gotcha:

- **Monit's regular expressions are POSIX ERE, not PCRE.** A `\b` word boundary is a parse error, and a config parse error silently kills the whole Monit process, not just the new test. All monitoring was down until the error appeared in Monit's own log. The pattern that works matches case-insensitively with explicit character classes (`[Ee][Rr][Rr][Oo][Rr]`).
- **The GUI's Apply button does not reliably reload the running process.** The file on disk had the new pattern while Monit was still running the old one, confirmed by comparing the "Reinitializing Monit" time in its log with the edit time. After any change: restart the service and read Monit's log.
- **A short fragment matches too much.** A bare `emerg` alternative matched "Emerging Threats" in routine rule-update lines. Dropping the fragment and keeping the whole word `emergency` fixed it, with no boundary anchors needed.

Intrusion-prevention and CrowdSec alerts are out of scope on purpose: the goal was general system errors, and those tools have their own notification paths.
