# ssh connections

A practical reference for everything around connecting to remote hosts: `ssh`, key
management with `ssh-keygen` and `ssh-add`, deploying keys with `ssh-copy-id`, port
forwarding, jump hosts, file transfer with `scp` / `sftp`, an example `~/.ssh/config`
block, and debugging recipes.

## 1. Basic connect

Plain connection with the default port and your default identity:

```bash
ssh alice@server.example.com
```

Non-default port — `-p` for `ssh`, capital `-P` for `scp`/`sftp`:

```bash
ssh -p 22 alice@server.example.com
```

Use a specific private key (skip the agent and any `~/.ssh/config` `IdentityFile`):

```bash
ssh -i ~/.ssh/id_ed25519 alice@server.example.com
```

## 2. Generate a key pair

Modern Ed25519 — small, fast, secure. Empty passphrase is fine for short-lived agent
keys, but use a passphrase for keys that live on disk:

```bash
ssh-keygen -t ed25519 -C "alice@laptop" -f ~/.ssh/id_ed25519
```

Legacy RSA for systems that don't yet accept Ed25519 (e.g. older Cisco / network gear).
Use at least 4096 bits:

```bash
ssh-keygen -t rsa -b 4096 -C "alice@laptop" -f ~/.ssh/id_rsa
```

## 3. Inspect / fingerprint a key

Show fingerprint and comment of a public key:

```bash
ssh-keygen -lf ~/.ssh/id_ed25519.pub
```

Force SHA-256 fingerprint (the format GitHub etc. display):

```bash
ssh-keygen -E sha256 -lf ~/.ssh/id_ed25519.pub
```

Re-derive a public key from a private key (handy when the `.pub` was lost):

```bash
ssh-keygen -y -f ~/.ssh/id_ed25519
```

## 4. Add a key to ssh-agent

Start an agent in the current shell:

```bash
eval "$(ssh-agent -s)"
```

Add a key (you'll be prompted for the passphrase once):

```bash
ssh-add ~/.ssh/id_ed25519
```

List loaded keys by fingerprint:

```bash
ssh-add -l
```

macOS only — store the passphrase in the login keychain so the key is loaded
automatically on reboot:

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

## 5. Copy your public key to a server

`ssh-copy-id` appends your public key to `~/.ssh/authorized_keys` on the remote,
creating the file with the right permissions if needed:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub alice@server.example.com
```

Manual fallback when `ssh-copy-id` isn't available (e.g. minimal busybox hosts):

```bash
cat ~/.ssh/id_ed25519.pub | ssh alice@server.example.com 'mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

## 6. Local port forwarding

Bring a remote service to a local port — for example, talk to a database that only
listens on the bastion's private network:

```bash
ssh -L 8080:db.internal:5432 alice@server.example.com
```

After connecting, `localhost:8080` on your machine reaches `db.internal:5432` from the
server's perspective.

## 7. Remote port forwarding

Expose a local service to the remote host — useful for sharing a dev server through a
public bastion:

```bash
ssh -R 5432:localhost:8080 alice@server.example.com
```

Anyone on the remote box can now hit `localhost:5432` and reach your laptop's port
8080. To bind on all remote interfaces (not just loopback) the server needs
`GatewayPorts yes` in `sshd_config`.

## 8. Dynamic SOCKS proxy

Turn the SSH connection into a SOCKS5 proxy your browser (or any SOCKS-aware client)
can use. `-N` says "don't run a remote command", `-f` backgrounds the process after
authentication:

```bash
ssh -D 1080 -N -f alice@server.example.com
```

Point your browser's SOCKS5 proxy at `localhost:1080` to tunnel all traffic through
the host.

## 9. Jump host (ProxyJump)

Reach a private host through a bastion in one command:

```bash
ssh -J alice@bastion.example.com alice@server.example.com
```

`-J` can be chained (`-J host1,host2`) for multi-hop. Prefer a `ProxyJump` line in
`~/.ssh/config` once the path is stable.

## 10. File transfer

Copy a single file up:

```bash
scp ./deploy.tar.gz alice@server.example.com:/var/www/app
```

Recursively copy a directory:

```bash
scp -r ./deploy.tar.gz alice@server.example.com:/var/www/app
```

Non-default port (note the capital `-P`):

```bash
scp -P 22 ./deploy.tar.gz alice@server.example.com:/var/www/app
```

Interactive sftp session for browsing / pulling several files:

```bash
sftp alice@server.example.com
```

## 11. SSH config file

Drop this block in `~/.ssh/config` and `ssh app` is enough — every flag becomes a
default for that host alias:

```sshconfig
Host app
    HostName server.example.com
    User alice
    Port 22
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump alice@bastion.example.com
    ServerAliveInterval 60
    ServerAliveCountMax 3
    ForwardAgent no
```

## 12. Diagnose & debug

One `-v` shows the negotiation overview; `-vvv` is full protocol trace — the first
thing to reach for when auth or connection mysteries strike:

```bash
ssh -v alice@server.example.com
```

```bash
ssh -vvv alice@server.example.com
```

Test git auth against GitHub (or any host that returns a banner instead of a shell).
`-T` disables pseudo-terminal allocation:

```bash
ssh -T git@github.com
```

Print the effective config that would apply for a host — including everything merged
from `~/.ssh/config`, `/etc/ssh/ssh_config`, and command-line options. No connection
is made:

```bash
ssh -G server.example.com
```
