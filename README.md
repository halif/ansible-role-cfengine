# ansible-role-cfengine

Ansible role for installing [CFEngine](https://cfengine.com/) Community
Edition and bootstrapping it to a policy server.

Yes — this installs a *different* configuration management system using
Ansible. That's a legitimate real-world pattern: Ansible (push-based) is
often used to bootstrap servers, including installing a long-running
pull-based agent (CFEngine, Puppet, Chef) for continuous, unattended
configuration enforcement afterward. The two tools solve different
problems and commonly coexist.

## What it does

1. Adds the official CFEngine repository (apt/yum/zypper, with a modern
   non-deprecated GPG key setup for apt).
2. Installs the `cfengine-community` package.
3. Bootstraps the agent to a policy server you specify — this is what
   actually starts CFEngine's daemons (`cf-execd`, `cf-serverd`,
   `cf-monitord`) and establishes trust.

## Requirements

- Ansible >= 2.14
- Target OS: Ubuntu 22.04/24.04, Debian 11/12, RHEL/Rocky/Alma 8/9,
  openSUSE (systemd)
- Internet access on the target host (to install the package)
- Required collection: `community.general` (for openSUSE/zypper support)
  — install with:
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

## Topology

You must set `cfengine_policy_server` explicitly — there is no sensible
default. Two common patterns:

- **Self-bootstrap** (single host acting as both hub and client, common
  for learning/testing): set it to the host's own IP.
- **Hub + clients** (real deployment): bootstrap the hub to itself
  first, then bootstrap every client host to the hub's IP.

```yaml
- hosts: cfengine_hub
  become: true
  roles:
    - role: halif.cfengine
      vars:
        cfengine_policy_server: "{{ ansible_default_ipv4.address }}"

- hosts: cfengine_clients
  become: true
  roles:
    - role: halif.cfengine
      vars:
        cfengine_policy_server: "10.0.1.1"   # the hub's IP
```

## Role Variables

See the full list with defaults in [`defaults/main.yml`](defaults/main.yml).

| Variable                   | Default    | Description                                        |
|------------------------------|-------------|-------------------------------------------------------|
| `cfengine_policy_server`     | `""` (**required**) | IP/hostname of the policy server to bootstrap to |
| `cfengine_package_name`      | `"cfengine-community"` | Package name to install                     |

## Known upstream issue (RHEL family)

CFEngine's official yum repository (hosted on S3) bundles builds for
multiple EL major versions (el8, el9, el10, ...) together in one flat
path with no version-specific subdirectory. Without pinning the EL
suffix explicitly, `dnf` may resolve to a newer build than your host's
actual OS version supports (e.g. picking an el10 build on an el9 host,
which then fails to resolve `selinux-policy` and other OS-version-tied
dependencies). `tasks/install.yml` works around this by pinning the
package spec to `cfengine-community-*.el{{ ansible_facts['distribution_major_version'] }}`.

Separately, the repository has also shown transient reliability issues
in the past — see [CFE-2671](https://tracker.mender.io/browse/CFE-2671)
(incorrect HTTP status codes that can confuse metadata resolution).
`tasks/install.yml` retries the RHEL install step a few times to ride
out such transient failures.

## Testing

CI runs a **single-host smoke test**: the role bootstraps a container to
itself (`127.0.0.1`), on Ubuntu 22.04 and Rocky Linux 9. This validates
the automation (repository setup, package install, bootstrap command,
idempotency) but does **not** exercise a real multi-host hub/client
topology, since that requires genuinely separate reachable machines.

The openSUSE branch of `tasks/prereqs.yml` follows CFEngine's official
zypper instructions but is **not currently exercised in CI** — there was
no verified, init-capable Docker test image available at the time this
role was written. If you use this role on openSUSE, please validate it
manually and report back.

```bash
pip install ansible molecule "molecule-plugins[docker]" docker
ansible-galaxy collection install -r requirements.yml
molecule test
```

## License

MIT

## Author

[halif](https://github.com/halif)
