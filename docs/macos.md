# Bootstrapping on MacOS host
This doc describes workarounds for bootstrapping Skarabox on a macOS host where the official installation instructions don't work: https://github.com/ibizaman/skarabox/blob/main/docs/installation.md

## Prerequisites
- Nix installed on the macOS host
- Cloud server provisioned with any Linux distribution (e.g., Ubuntu, Debian, etc.)

## Initializing the repo

### Original instructions
```bash
mkdir myskarabox
cd myskarabox
nix run github:ibizaman/skarabox?ref=@VERSION@#init
```

### Problems with original instructions
- Not sure what `@VERSION@` should be replaced with
- Conflicting packages on macOS host path (e.g. sed) cause failures.

### Recommended workaround
```bash
mkdir myskarabox
cd myskarabox
nix-shell --pure -p nix
nix run github:ibizaman/skarabox#init
```

## Generating known hosts
### Original instructions
```bash
echo <ip> > myskarabox/ip
echo x86_64-linux > myskarabox/system
nix run .#myskarabox-gen-knownhosts-file

nix run .#packages.aarch64-darwin.myskarabox-gen-knownhosts-file

# Optionally, adjust the ./myskarabox/ssh_port and ./myskarabox/ssh_boot_port if you want to.
```

### Problems with original instructions
- `nix run .#myskarabox-gen-knownhosts-file` fails on macOS due to cross-platform issues.
- Not sure when/why `myskarabox/ssh_port` and `myskarabox/ssh_boot_port` need to be adjusted. 

### Recommended workaround
```bash
echo <ip> > myskarabox/ip 
echo x86_64-linux > myskarabox/system

echo "[$(cat ./myskarabox/ip)]:22 $(cat ./myskarabox/host_key.pub | cut -d' ' -f1-2)" > ./myskarabox/known_hosts
echo "[$(cat ./myskarabox/ip)]:2222 $(cat ./myskarabox/host_key.pub | cut -d' ' -f1-2)" >> ./myskarabox/known_hosts
```

## Facter
### Original instructions
```bash
nix run .#myskarabox-get-facter > ./myskarabox/facter.json
```

### Problems with original instructions
- `nix run .#myskarabox-get-facter` fails on macOS due to cross-platform issues.

### Recommended workaround
```bash
# SSH into the server and install nix
sh <(curl --proto '=https' --tlsv1.2 -L https://nixos.org/nix/install) --daemon

# Reset the shell to apply Nix environment
exec $SHELL 

# Install nixos-facter on the server
nix-env -iA nixpkgs.nixos-facter

# Go back to the macOS host and run:
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=./myskarabox/known_hosts -p 22 root@$(cat ./myskarabox/ip) "sudo bash -c 'source ~/.nix-profile/etc/profile.d/nix.sh && nixos-facter'" > ./myskarabox/facter.json
```

## Installation
### Original instructions
```bash
nix run .#myskarabox-install-on-beacon
```
### Problems with original instructions
- `nix run .#myskarabox-install-on-beacon` fails on macOS due to cross-platform issues.

### Recommended workaround
```bash
# TODO
```

## Long term plan
The long-term plan is to fix the Skarabox fork to work on macOS hosts, so that we can use the original instructions without workarounds. This will involve addressing cross-platform compatibility issues and ensuring that all necessary tools are available on macOS.

For developing a local fork, you need to replace the commands in the instructions with the following:

```bash
nix run github:ibizaman/skarabox#myskarabox-get-facter -- -p /skarabox 
```