# no-mail-server

**Every Debian host in the lab reports through one 39-line script and one notification server. None of them runs a mail server.**

Tools on Linux hosts still expect `/usr/sbin/sendmail`: unattended-upgrades reports through it, cron sends its output through it, and half the scripts you write call it. The usual answer is to install an MTA, which means a daemon to patch, a relay to configure and credentials to keep, all to deliver a few lines of text.

This repo is the other answer. `/usr/sbin/sendmail` is a symlink to [`ntfy-sendmail`](files/ntfy-sendmail), a sendmail-compatible shim that turns the message into one authenticated HTTPS request to a self-hosted [ntfy](https://ntfy.sh) server. Nothing listens on port 25, nothing queues, and the only secret is a token in a root-only file.

```
unattended-upgrades ─┐
cron                 ├─▶ /usr/sbin/sendmail ─▶ ntfy-sendmail ─▶ HTTPS ─▶ ntfy ─▶ phone
my own scripts       ┘     (symlink)            (reads token from
                                                /etc/ntfy-sendmail.env)
```

## What the shim does

| Caller writes | How the shim reads it |
|---|---|
| `sendmail -s "subject"` with the body on stdin | Subject from `-s`, the whole of stdin is the body |
| A complete mail message on stdin (what unattended-upgrades and cron send) | Subject from the `Subject:` header, body is everything after the first blank line |

- No subject at all becomes `Notification from <hostname>`. An empty body falls back to the whole input.
- Recipient arguments are ignored. All mail from a host goes to one topic, set by `NTFY_URL`.
- The usual sendmail flags (`-oi`, `-t`, `-f`, `-F`) are accepted and ignored, so a tool that passes them works without error output.
- The body is capped at 3500 bytes, under ntfy's default 4 KB message limit.
- `curl` runs with `--max-time 20`, fails on an HTTP error (`-f`) and discards the response, so a slow or failing server can't hang the caller, and the script's exit status tells the caller whether the publish worked.
- The URL and token come from `/etc/ntfy-sendmail.env` (`root:root`, mode `0600`). If the file can't be read the script exits 1 and says so on stderr. The script itself is world-readable and contains no secret.

## What's here

| Path | What it is |
|---|---|
| [`files/ntfy-sendmail`](files/ntfy-sendmail) | The shim, byte for byte what runs on the hosts |
| [`docs/design.md`](docs/design.md) | Why no mail server, what the shim guarantees and what it doesn't |
| [`docs/fleet-standard.md`](docs/fleet-standard.md) | How every host is held to the same setup, with the Ansible tasks |
| [`docs/consumers.md`](docs/consumers.md) | Everything that sends through it, and the paths that can't use the shim |
| [`docs/rollout-and-testing.md`](docs/rollout-and-testing.md) | How a host is tested, and the faults the rollout turned up |

## Limits

- **No delivery guarantee.** The shim makes one attempt. If the notification server is down, that message is lost. The server runs on its own host, away from the application host that gets backed up, to keep that window small.
- **Root only.** The token file is `0600`, so a non-root caller of `sendmail` fails by design.
- **It is not a general mailer.** It carries text to one topic. Attachments, multiple recipients and HTML are out of scope on purpose.

## Related

- [diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit): the base standard asserts that `/usr/sbin/sendmail` is never a real mail server and that it links to this shim.
- [expire-on-purpose](https://github.com/zacharyrichter/expire-on-purpose): the certificate expiry check sends its monthly report through the shim.
- [trust-no-cache](https://github.com/zacharyrichter/trust-no-cache): the internet-facing VPS, which runs its own independent ntfy server and the same shim.

Everything here is sanitized: domains are `example.net` and the human account is `admin`.

MIT licensed. See [`LICENSE`](LICENSE).
