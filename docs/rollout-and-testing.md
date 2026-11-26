# Rollout and testing

## Testing a host

Three steps, in this order, on every host:

1. **Reach the server.** `curl -s https://ntfy.example.net/v1/health` from the host.
2. **Publish with the token.** An authenticated publish as root, to prove the token and the topic ACL.
3. **Run the real consumer.** A real unattended-upgrades run that reports through the shim, not only a synthetic message.

And three calling styles, as root, each of which should arrive on the topic:

```
echo "body" | sendmail -s "test" root
printf 'To: root\nSubject: test\n\nbody\n' | sendmail root
printf 'To: root\nSubject: test\n\nbody\n' | sendmail -oi -t
```

These are three different paths through the script, so a test of one proves nothing about the others. The third one also has to finish with no `illegal option` lines on stderr.

Non-root callers fail, and that is expected: the env file is `0600`.

## Faults the rollout found

| Fault | What it was | Fix |
|---|---|---|
| A cloned mail stack | One host ran a local MTA and wrapper scripts relaying through a hosted mail API, copied from another machine | Removed, and replaced by the shim. A custom daily "reboot required" mail script on the same host was removed too, because unattended-upgrades flags a pending reboot itself |
| Two versions of the script | Hashing the script on each host showed two different files. One parsed only the `Subject:` header; the other also handled `-s` | Standardized on one script that handles both, copied to every host and checked with `sha256sum` |
| Sendmail's own flags | A later comparison found one more difference: the internet-facing host's copy accepted flags such as `-oi` and `-t`, and the rest of the fleet's did not. On those hosts a caller passing them got `illegal option` on stderr, though the message still sent | One option string that accepts and ignores the usual flags, installed on every host and checked by hash, then tested in all three calling styles |
| A missing dependency | A minimal base image had no `curl`, and the shim needs it | `curl` is now installed by the hardening playbook |
