# ansible-role-sshd

Configure and harden SSH daemon.

## Requirements

- Debian/Ubuntu

## Supported Distributions

- Debian (bullseye, bookworm)
- Ubuntu (focal, jammy, noble, resolute)

Other distributions will fail with an error message.

## Role Variables

### Basic

| Variable | Default | Description |
|----------|---------|-------------|
| `sshd_port` | `22` | SSH port |
| `sshd_address_family` | `inet` | Address family (inet, inet6, any) |
| `sshd_clean_config_d` | `false` | Remove all files from /etc/ssh/sshd_config.d/ |
| `sshd_manage_socket` | `true` | Manage systemd `ssh.socket` port override (Ubuntu 24.04+) |

### Authentication

| Variable | Default | Description |
|----------|---------|-------------|
| `sshd_permit_root_login` | `no` | Allow root login |
| `sshd_password_authentication` | `no` | Allow password auth |
| `sshd_permit_empty_passwords` | `no` | Allow empty passwords |
| `sshd_pubkey_authentication` | `yes` | Allow pubkey auth |
| `sshd_kbd_interactive_authentication` | `no` | Allow keyboard-interactive auth |
| `sshd_hostbased_authentication` | `no` | Allow host-based auth |
| `sshd_ignore_rhosts` | `yes` | Ignore .rhosts files |
| `sshd_permit_user_environment` | `no` | Allow user environment |
| `sshd_strict_modes` | `yes` | Check file permissions |
| `sshd_login_grace_time` | `30` | Login grace time (seconds) |
| `sshd_max_auth_tries` | `3` | Max auth attempts |
| `sshd_max_sessions` | `10` | Max sessions |
| `sshd_max_startups` | `10:30:60` | Max concurrent connections |
| `sshd_allow_users` | `[]` | List of allowed users |
| `sshd_allow_groups` | `[]` | List of allowed groups |

### Forwarding

| Variable | Default | Description |
|----------|---------|-------------|
| `sshd_x11_forwarding` | `no` | Allow X11 forwarding |
| `sshd_allow_tcp_forwarding` | `no` | Allow TCP forwarding |
| `sshd_allow_agent_forwarding` | `no` | Allow agent forwarding |

### Connection

| Variable | Default | Description |
|----------|---------|-------------|
| `sshd_tcp_keep_alive` | `yes` | TCP keep-alive |
| `sshd_client_alive_interval` | `600` | Keep-alive interval (seconds) |
| `sshd_client_alive_count_max` | `3` | Keep-alive count |
| `sshd_use_dns` | `no` | Use DNS lookup |

### Crypto

| Variable | Default | Description |
|----------|---------|-------------|
| `sshd_ciphers` | modern ciphers | Allowed ciphers |
| `sshd_macs` | modern MACs | Allowed MACs |
| `sshd_kex_algorithms` | modern KEX | Allowed key exchange algorithms |

The crypto lists are **filtered per host** at runtime: the role queries the host's
OpenSSH (`ssh -Q kex/cipher/mac`) and drops any algorithm the local OpenSSH does not
support, preserving the desired order. This lets the same default list include
post-quantum KEX (`mlkem768x25519-sha256`, OpenSSH 9.9+) without breaking `sshd -t`
validation on older hosts (e.g. OpenSSH 9.6 on Ubuntu 24.04), where the unknown
algorithm is silently removed instead of rejecting the whole config.

### Other

| Variable | Default | Description |
|----------|---------|-------------|
| `sshd_log_level` | `VERBOSE` | Log level |
| `sshd_use_pam` | `yes` | Use PAM |
| `sshd_print_motd` | `no` | Print MOTD |

### Distro-specific (auto-detected)

| Variable | Debian/Ubuntu | RHEL/Rocky |
|----------|---------------|------------|
| `sshd_service_name` | `ssh` | `sshd` |

## Backup

- Original configuration is backed up to `/etc/ssh/sshd_config.orig` on first run
- Original sshd_config.d files are backed up to `/etc/ssh/sshd_config.d.orig/` when `sshd_clean_config_d: true`

## systemd socket activation (Ubuntu 24.04+/26.04)

On Ubuntu 24.04 noble and 26.04 resolute SSH is started via `ssh.socket`
(systemd socket activation). In this mode:

- The listening port is owned by `ssh.socket` (`ListenStream=`), **not** by `Port`
  in `sshd_config` — changing `sshd_port` alone has no effect.
- Each connection spawns a fresh `sshd`, so `sshd_config` changes (crypto, auth)
  are applied to new connections automatically.

When `sshd_manage_socket: true` (default) and `sshd_port != 22`, the role deploys
`/etc/systemd/system/ssh.socket.d/override.conf` to set the listening port and
restarts `ssh.socket` (not `ssh.service`, which would conflict). On hosts without
socket activation (most Debian installs) the role restarts `ssh.service` as before.

## Cleaning sshd_config.d

Some cloud providers (AWS, DigitalOcean, Hetzner, etc.) add files to `/etc/ssh/sshd_config.d/`
that may override your settings (e.g., `PasswordAuthentication yes`).

To remove these overrides:

```yaml
sshd_clean_config_d: true
```

Original files will be backed up to `/etc/ssh/sshd_config.d.orig/` on first run.

## Examples

### Basic usage

```yaml
sshd_allow_users:
  - deploy
```

### Clean cloud provider overrides

```yaml
sshd_clean_config_d: true
sshd_allow_users:
  - deploy
```

### Custom port with group access

```yaml
sshd_port: 2222
sshd_allow_groups:
  - sudo
```

### Allow TCP forwarding for tunnels

```yaml
sshd_allow_users:
  - deploy
sshd_allow_tcp_forwarding: "yes"
```

## Usage

```yaml
- hosts: all
  become: true
  roles:
    - sshd
```

## Warning

Make sure you have a user with SSH key access and sudo rights
before applying this role, otherwise you will be locked out!

Checklist before running:

1. User created (ansible-role-users)
2. SSH key added to user
3. User has sudo access (ansible-role-sudo)
4. User is in `sshd_allow_users` or `sshd_allow_groups`

## Adding new distribution support

1. Create `vars/<DistributionName>.yml` with `sshd_service_name`
2. Add distribution to supported list in `tasks/main.yml`
3. Update `meta/main.yml` platforms

## License

MIT
