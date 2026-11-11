# sysctl_hardening

Sole owner of kernel sysctl settings in this collection. Writes one drop-in, `/etc/sysctl.d/99-zz-linux-hardening.conf` (sorts after `99-sysctl.conf`), and sets the running values. Keys that do not exist on the running kernel are skipped and reported.

## What it enforces
Our own selection from common hardening guidance (not a verified CIS mapping): no ICMP/IP redirects or source routing, martian logging, SYN cookies, ASLR, `kptr_restrict`, `dmesg_restrict`, `yama.ptrace_scope`, `suid_dumpable`, protected hard/symlinks. `cis_level2` adds `sysrq`, protected FIFOs/regular files, `perf_event_paranoid`, `tcp_rfc1337`. Exact list: `vars/main.yml`.

Host-role dependent (defaults in `defaults/main.yml`): `ip_forward` (false), `rp_filter` (1), `ipv6_accept_ra` (unmanaged). Add or remove keys with `linux_hardening_sysctl_extra` / `linux_hardening_sysctl_overrides` (null removes a key).

## Modes
`audit` (default) is read-only and sets `linux_hardening_sysctl_findings` (`key, expected, actual, fixable`). `remediate` needs `linux_hardening_confirm_remediation: true`.

## Safety
- Switching IPv4 forwarding from on to off is refused unless `linux_hardening_sysctl_confirm_disable_forwarding: true` (it breaks Docker, Podman, Kubernetes). Set `linux_hardening_sysctl_ip_forward: true` on those hosts.
- A value the kernel refuses (containers, read-only `/proc/sys`) fails the run unless `linux_hardening_sysctl_ignore_errors: true`.

- Order matters: switching `net.ipv4.ip_forward` makes the kernel reset `accept_redirects` and related keys. The role therefore writes the forwarding keys first, re-reads the live values, then applies the rest, and lists forwarding first in the persistent file (so `sysctl --system` at boot behaves the same).

## Example
```yaml
- hosts: all
  roles:
    - role: iamenr0s.linux_hardening.sysctl_hardening
      vars:
        linux_hardening_mode: remediate
        linux_hardening_confirm_remediation: true
        linux_hardening_sysctl_ip_forward: true   # container host
```

## Verification
Molecule `default`, `strict` (cis_level2), `audit` (writes nothing), `refuse` (forwarding guard) in containers; `vagrant` on a real libvirt VM (writes the live kernel with no `ignore_errors`, checks `sysctl -n`). Tested: AlmaLinux 9 (containers and VM). CI matrix: AlmaLinux 8–10, Rocky 8–10, Fedora 42–44, Debian 12–13, Ubuntu 22.04/24.04.

## Rollback
Delete the drop-in (a backup of any earlier version is kept when `linux_hardening_backup` is true); running values revert at reboot or via `sysctl --system`.

## Known gaps
- Containers share the host kernel, so tests there cannot prove every key is settable.
- Keys set in a file sorting after `99-zz-` can still override these.
- CIS IDs not mapped; module blacklisting and boot parameters belong to a separate kernel role.

CI & Release (maintainers)
----------------------------
A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --repo iamenr0s/ansible-role-sysctl-hardening

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --repo iamenr0s/ansible-role-sysctl-hardening
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.ansible-role-sysctl-hardening`
