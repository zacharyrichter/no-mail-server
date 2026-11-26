# Design

## The problem

Software on a Linux host reports problems by mail. Unattended-upgrades, cron and integrity checkers all expect a local `sendmail`. A lab that wants those reports on a phone has two options: run a mail server and bridge it, or give the software a `sendmail` that isn't one.

## Why no MTA

One host in this lab carried a copy-pasted mail setup: a local mail transfer agent, plus wrapper scripts that relayed through a hosted mail API. It had been cloned from another machine and nobody had chosen it. That is the case against an MTA in one line: it was a service, with credentials, that nobody was responsible for.

Replacing it with the shim removed:

- a daemon that has to be patched and watched,
- an outbound relay account and its credentials on every host,
- an open question, on every host, of whether something is listening on port 25.

What remains is one script that does one thing and has one failure mode: it can't reach the server.

## Three choices in the script

**Two calling styles.** Mail clients disagree about how to pass a subject. Some pipe a full message with a `Subject:` header. Others call `sendmail -s "subject"` and pipe only the body. The shim accepts both, because the first style is what unattended-upgrades sends and the second is what integrity checkers and the lab's own scripts use. The second style was added after the first version only parsed the header; the fleet then had two different scripts, which is its own lesson (see [rollout-and-testing.md](rollout-and-testing.md#faults-the-rollout-found)).

**It tolerates sendmail's own flags.** Tools call `sendmail -oi -t` and similar. The script's option string accepts `-o`, `-i`, `-t`, `-f` and `-F` and ignores them, because the shim has no use for any of them. Without that, `getopts` prints `illegal option` for each one, which turns every cron mail into noise on stderr.

**The token is not in the script.** The URL and token are read from `/etc/ntfy-sendmail.env`, owned by root, mode `0600`, never in the repository and never managed by the playbook that installs the script. The script can therefore be world-readable and identical on every host, and a token can be rotated by changing one file per host.

**It fails loudly and quickly.** A missing env file is an error on stderr and exit status 1. A failed publish returns curl's non-zero status, so cron or a calling script can see it. The 20-second timeout means a dead server delays the caller by seconds, not indefinitely.

## What it deliberately doesn't do

- No retry and no queue. A queue is a mail server by another name.
- No per-recipient routing. The topic is in the URL, so every message from a host lands in one place. That is enough for host-level reports, and it keeps the shim from growing into the thing it replaced.
- No attachments and no HTML.

## The server side

ntfy runs on its own host, not on the application host that gets backed up and rebooted, so notifications don't go down at the moments when most of them are sent. That host has a snapshot-mode backup job of its own, so even its own backup doesn't stop it.

The server's default access is deny-all, with login enabled. Hosts publish with a bearer token; people read with their own accounts. The server also has an SMTP listener on a non-standard port, for the one kind of sender that can only speak mail (see [consumers.md](consumers.md#a-sender-that-can-only-speak-mail)).

The internet-facing VPS is deliberately **not** pointed at this server. It runs its own independent ntfy instance and the same script against that instance, so a host exposed to the internet never needs a path into the home network.
