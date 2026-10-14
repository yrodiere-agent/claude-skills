---
name: incus-spawn-dev
description: >
  Working inside a tpl-incus-spawn VM — a pre-configured recursive
  development environment for incus-spawn (isx). Covers sudo access,
  missing packages (awk, etc.), build commands, and key constraints
  like HTTPS-only git.
---

# Incus-Spawn Development Environment

Use inside `tpl-incus-spawn`: check `$ISX_TEMPLATE`, or verify
`~/.local/bin/isx` and the Incus `cow` pool. Do not reinitialize existing setup.

## Environment

- `~/.local/bin/isx`; nested Incus with a 10GiB btrfs `cow` pool.
- Inner MITM proxy, firewalld, and incus-admin group configured.
- Placeholder credentials in `~/.config/incus-spawn/config.yaml` pass checks;
  the outer proxy injects real credentials. Do not replace placeholders.
- Repositories: `~/incus-spawn` (Java/Quarkus CLI),
  `~/incus-spawn-templates` (registered search path), `~/incus-spawn-images`.
- Passwordless sudo. Install missing packages, e.g. `sudo dnf install -y gawk`.
- Use HTTPS GitHub remotes; SSH is unavailable. After reboot, retry if Incus
  has not become ready yet.

## Build and Test

```bash
mvnd install -DskipTests
mvnd test                          # No Incus required
mvnd verify -DskipITs=false         # Requires running Incus
env -u JAVA_TOOL_OPTIONS ~/incus-spawn/install.sh
```

Unset `JAVA_TOOL_OPTIONS` for installation: its truststore settings interfere
with Java version detection.

```bash
isx build tpl-minimal
isx branch --from tpl-minimal mytest
isx destroy mytest
```
