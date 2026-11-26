# Fleet standard

Every Debian VM and LXC is held to the same setup. The Proxmox nodes and the backup server are the exception: they use their own webhook notification target (see [consumers.md](consumers.md#proxmox-and-the-backup-server)) and the base audit skips them for this rule.

## The three requirements

| Requirement | Enforced by |
|---|---|
| `/usr/sbin/sendmail` is never a regular file, so a real mail server can't appear unnoticed | an assertion in the base standards playbook |
| `/usr/sbin/sendmail` is a symlink to `/usr/local/bin/ntfy-sendmail` | a task in the same playbook, which reports drift in `--check --diff` |
| `/usr/local/bin/ntfy-sendmail` is the shim in [`files/ntfy-sendmail`](../files/ntfy-sendmail), `root:root`, mode `0755`, and `/etc/ntfy-sendmail.env` exists | the hardening playbook that builds a new host |

The first two, from the base standards playbook:

```yaml
- name: Check sendmail
  stat:
    path: /usr/sbin/sendmail
    follow: false
  register: sendmail_path
  when: not is_hypervisor

- name: sendmail must not be a real MTA (the ntfy shim is the only mailer)
  assert:
    that: not (sendmail_path.stat.exists and sendmail_path.stat.isreg)
    fail_msg: "/usr/sbin/sendmail is a regular file on {{ inventory_hostname }}; check what installed an MTA"
    quiet: true
  when: not is_hypervisor

- name: sendmail points at the ntfy shim
  file:
    src: /usr/local/bin/ntfy-sendmail
    dest: /usr/sbin/sendmail
    state: link
  when: not is_hypervisor
```

The assertion is a rule that can't be a change. If something installs Postfix or Exim, deleting the symlink and putting a real binary there is not something the playbook should quietly undo, so the audit stops and names the host. The reasoning is in [the audit model](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/docs/the-audit-model.md).

## Installing on a new host

The hardening playbook installs `curl` (the shim needs it, and a minimal base image doesn't have it), installs the script, and checks for the env file:

```yaml
- name: Install curl
  apt:
    name: curl
    state: present

- name: Check for ntfy-sendmail config
  stat:
    path: /etc/ntfy-sendmail.env
  register: ntfy_env

- name: Warn if ntfy-sendmail config is missing
  debug:
    msg: "/etc/ntfy-sendmail.env is missing on {{ inventory_hostname }} - notifications will fail until it exists (root:root 0600, NTFY_URL and NTFY_TOKEN)"
  when: not ntfy_env.stat.exists
```

The script is installed with the `copy` module (`/usr/local/bin/ntfy-sendmail`, `root:root`, `0755`, `diff: false`) with the content of [`files/ntfy-sendmail`](../files/ntfy-sendmail).

## The env file

The playbook does not manage `/etc/ntfy-sendmail.env`, because it holds the token. It is created by hand on each host:

```
NTFY_URL=https://ntfy.example.net/host-updates
NTFY_TOKEN=<token>
```

Owner `root:root`, mode `0600`. Because the playbook only warns when it is missing, a new host is never blocked on the token, and the secret never passes through the repository.

## Unattended-upgrades

Unattended-upgrades gets its reporting from a separate file, so the stock `50unattended-upgrades` keeps its own mail lines commented out:

```
# /etc/apt/apt.conf.d/51unattended-upgrades-mail
Unattended-Upgrade::Mail "ntfy";
Unattended-Upgrade::MailReport "on-change";
```

The recipient is a placeholder, because the shim ignores it. `on-change` means a report only when something was upgraded or went wrong.

## Checking that hosts agree

The same script runs on every host, byte for byte. Comparing is one command per host:

```
sha256sum /usr/local/bin/ntfy-sendmail
```

Checking this by hand is how the fleet's two diverging copies were found (see [rollout-and-testing.md](rollout-and-testing.md#faults-the-rollout-found)).
