
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
- Interactively set the password.

- gen server config
Retrieve the IP of the server, then:

```bash
echo <ip> > myskarabox/ip
echo x86_64-linux > myskarabox/system
nix run .#myskarabox-gen-knownhosts-file

# Optionally, adjust the ./myskarabox/ssh_port and ./myskarabox/ssh_boot_port if you want to.
```
