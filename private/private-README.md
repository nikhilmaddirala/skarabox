
# Nikhil's Skarabox README
This is a supplementary README for Nikhil's Skarabox setup, providing installation instructions and troubleshooting tips. It addresses places where the original Skarabox documentation was not working (often due to macos-specific issues) and includes additional context for setting up Skarabox on a Hetzner server.


# Bootstrapping Skarabox
https://github.com/ibizaman/skarabox/blob/main/docs/installation.md

- In a new empty directory, run:
```bash
nix-shell --pure -p nix
nix run github:ibizaman/skarabox#init

# Use a different path - e.g. for a local fork
nix run github:ibizaman/skarabox#init -- --path /path/to/skarabox

# TODO: Not sure what `@VERSION@` should be replaced with.
# TODO: Need to create a pure nix shell to avoid path conflicts.
```
- Interactively set the password and save it.

- Add server configs:
```bash
echo <ip> > myskarabox/ip
echo x86_64-linux > myskarabox/system

# Optionally, adjust the ./myskarabox/ssh_port and ./myskarabox/ssh_boot_port if you want to.
```

- Generate known hosts manually:
```bash
# Create known_hosts with both SSH ports
echo "[$(cat ./myskarabox/ip)]:22 $(cat ./myskarabox/host_key.pub | cut -d' ' -f1-2)" > ./myskarabox/known_hosts
echo "[$(cat ./myskarabox/ip)]:2222 $(cat ./myskarabox/host_key.pub | cut -d' ' -f1-2)" >> ./myskarabox/known_hosts
```


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