
# Nikhil's Skarabox README
This is a supplementary README for Nikhil's Skarabox setup, providing installation instructions and troubleshooting tips. It addresses places where the original Skarabox documentation was not working (often due to macos-specific issues) and includes additional context for setting up Skarabox on a Hetzner server.

## Prerequisites
- Hetzner server with known IP address
- Local Skarabox repository configured
- `hcloud` CLI (optional) for server management
- **Cross-platform note**: Some commands may need manual alternatives on macOS


## Recovery Mode (If Needed)
If your Hetzner server isn't accessible or needs a fresh start:

```bash
# Enable recovery mode
hcloud server enable-rescue <server-name> --type linux64
hcloud server reboot <server-name>
# Or use Hetzner Console: "Enable rescue & power cycle"
```

# Bootstrapping Skarabox
https://github.com/ibizaman/skarabox/blob/main/docs/installation.md

- In a new empty directory, run:
```bash
nix-shell --pure -p nix
nix run github:ibizaman/skarabox#init

# Use a different path - e.g. for a local fork
nix run github:ibizaman/skarabox#init -- --p /skarabox

# TODO: Not sure what `@VERSION@` should be replaced with.
# TODO: Need to create a pure nix shell to avoid path conflicts.
```
- Interactively set the password and save it.

- Add server configs:
```bash
# Option A: Automatic (with hcloud CLI)
hcloud server ip <server-name> > ./myskarabox/ip
hcloud server describe <server-name> --output json | jq -r '.server_type.architecture' > ./myskarabox/system

# Option B: Manual
echo <ip> > myskarabox/ip
echo x86_64-linux > myskarabox/system

# Optionally, adjust the ./myskarabox/ssh_port and ./myskarabox/ssh_boot_port if you want to.
```

- Generate known hosts:
```bash
# Option A: Using Nix (Linux systems)
nix run .#myskarabox-gen-knownhosts-file

# Option B: Using forked Skarabox (macOS-compatible)
nix run github:ibizaman/skarabox#myskarabox-gen-knownhosts-file -- -p /skarabox

# Option C: Manual (universal fallback)
echo "[$(cat ./myskarabox/ip)]:22 $(cat ./myskarabox/host_key.pub | cut -d' ' -f1-2)" > ./myskarabox/known_hosts
echo "[$(cat ./myskarabox/ip)]:2222 $(cat ./myskarabox/host_key.pub | cut -d' ' -f1-2)" >> ./myskarabox/known_hosts
```

## Hardware Detection
Detect server hardware configuration:

```bash
# Option A: Using Nix command
nix run .#myskarabox-get-facter > ./myskarabox/facter.json

# Option B: Using forked Skarabox (macOS-compatible)
nix run github:ibizaman/skarabox#myskarabox-get-facter -- -p /skarabox > ./myskarabox/facter.json

# Option C: Manual SSH (universal fallback)
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=./myskarabox/known_hosts -p 22 root@$(cat ./myskarabox/ip) sudo nixos-facter > ./myskarabox/facter.json
```

Commit the hardware configuration:
```bash
git add ./myskarabox/facter.json
git commit -m "Add hardware configuration for myskarabox"
```

## Installation

```bash
# Install NixOS on the server
nix run .#myskarabox-install-on-beacon

# After reboot, unlock the encrypted root partition
nix run .#myskarabox-unlock

# Verify installation
nix run .#myskarabox-ssh
zpool status
```

## Troubleshooting

**Cross-Platform Issues (macOS):**
- Error: `Cannot build '/nix/store/...-nixpkgs-patched.drv'`
- Solution: Use manual SSH commands or run problematic commands on the server

**SSH Connection Fails:**
- Ensure server is in recovery mode
- Verify IP address and SSH keys are correct

**Hardware Detection Fails:**
- Server might not be fully booted, wait and retry
- For Hetzner recovery mode: install Nix first, then run nixos-facter

## Archive
- Sync to git:

- Go to server
```bash
# Install nix
sh <(curl --proto '=https' --tlsv1.2 -L https://nixos.org/nix/install) --daemon
exec $SHELL #restart shell

# Clone the repository
git clone <repo>
cd <repo>

# Run the setup script
nix run .#myskarabox-gen-knownhosts-file --extra-experimental-features nix-command --extra-experimental-features flakes
```