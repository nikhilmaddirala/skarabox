# Detailed Guide: Getting facter.json and Installing Skarabox on Hetzner Cloud

## Prerequisites
1. **Hetzner Server Setup**: Your server should be accessible and you should know its IP address
2. **Local Skarabox Repository**: You have the Skarabox flake configured with your host settings
3. **hcloud CLI** (optional): For automated server management
4. **Platform Considerations**: Some commands may require running on the target server due to cross-platform build limitations

## Recovery Mode Setup (If Needed)

If your Hetzner server isn't accessible or needs a fresh start:

1. **Enable Recovery Mode**:
   ```bash
   # Using hcloud CLI
   hcloud server enable-rescue <server-name> --type linux64
   hcloud server reboot <server-name>
   
   # Or via Hetzner Console:
   # - Go to your server in the Hetzner Cloud Console
   # - Click "Enable rescue & power cycle"
   # - Wait for server to boot into rescue mode
   ```

2. **Access Recovery System**:
   - Recovery mode provides a minimal Linux environment
   - Default login: `root` with the rescue password from Hetzner
   - System has network access and basic tools installed

## Step-by-Step Installation Process

### 1. Server Preparation

#### Option A: Local Setup (Linux systems)

```bash
# Get server IP automatically
hcloud server ip <server-name> > ./myskarabox/ip

# Get server architecture 
hcloud server describe <server-name> --output json | jq -r '.server_type.architecture' > ./myskarabox/system

# Generate known hosts file
nix run .#myskarabox-gen-knownhosts-file
```

#### Option B: Server-Side Setup (for macOS/cross-platform issues)

If you encounter cross-platform build errors (e.g., on macOS), run the setup directly on the server:

```bash
# Setup IP and system files locally first
hcloud server ip <server-name> > ./myskarabox/ip
echo "x86_64-linux" > ./myskarabox/system

# SSH into recovery mode and run setup there
hcloud server enable-rescue <server-name> --type linux64
hcloud server reboot <server-name>

# SSH into the server
ssh root@$(cat ./myskarabox/ip)

# Install Nix on the server
curl -L https://nixos.org/nix/install | sh
source ~/.nix-profile/etc/profile.d/nix.sh

# Clone your repository (or upload files)
git clone <your-repo-url> /tmp/skarabox-setup
cd /tmp/skarabox-setup

# Generate known hosts file on the server
nix run .#myskarabox-gen-knownhosts-file

# Copy the generated known_hosts back to your local machine
# (or commit and pull it)
```

#### Option C: Manual Known Hosts (fallback)

If both above options fail, create the known_hosts file manually:

```bash
# Create known_hosts with both SSH ports
echo "[$(cat ./myskarabox/ip)]:22 $(cat ./myskarabox/host_key.pub | cut -d' ' -f1-2)" > ./myskarabox/known_hosts
echo "[$(cat ./myskarabox/ip)]:2222 $(cat ./myskarabox/host_key.pub | cut -d' ' -f1-2)" >> ./myskarabox/known_hosts
```

### 2. Hardware Detection (facter.json)

This step runs **locally** but connects to your server to gather hardware information:

```bash
# Option A: Using Nix command
nix run .#myskarabox-get-facter > ./myskarabox/facter.json

# Option B: Manual equivalent (for cross-platform issues)
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=./myskarabox/known_hosts -p 22 root@$(cat ./myskarabox/ip) sudo nixos-facter > ./myskarabox/facter.json
```

**What happens during this step**:
- Connects to server via SSH using the IP in `./myskarabox/ip`
- Runs `nixos-facter` on the remote server to detect:
  - CPU architecture and features
  - Memory configuration
  - Storage devices and layout
  - Network interfaces
  - Hardware-specific kernel modules needed
- Downloads the JSON output to your local machine

**Troubleshooting facter collection**:
- If SSH fails, ensure the server is in recovery mode
- For Hetzner Cloud, ensure the correct SSH key is configured
- Check that the IP in `./myskarabox/ip` is correct and accessible

### 3. Commit Hardware Configuration

```bash
# Add the hardware specification to version control
git add ./myskarabox/facter.json
git commit -m "Add hardware configuration for myskarabox"
```

### 4. Preview Configuration (Optional)

```bash
# See what NixOS configuration will be generated
nix run .#myskarabox-debug-facter-nix-diff

# Alternative view of configuration changes
nix run .#myskarabox-debug-facter-nvd
```

These commands show you exactly what NixOS modules and settings will be applied based on the detected hardware.

### 5. Run the Installation

```bash
# This is the main installation command
nix run .#myskarabox-install-on-beacon
```

**What happens during installation**:
- Uses `nixos-anywhere` to perform remote installation
- Connects to your server (in recovery mode if applicable)
- Partitions disks according to Skarabox's ZFS layout:
  - Creates encrypted root pool (`root`)
  - Creates encrypted data pool (`zdata`) if multiple disks
  - Sets up boot partition
- Installs NixOS with your configuration
- Configures ZFS encryption with your passphrase
- Sets up SSH keys and network configuration
- Installs bootloader
- Automatically reboots into the new NixOS system

### 6. Post-Installation: Root Decryption

After installation, the server reboots but needs the encrypted root partition unlocked:

```bash
# Connect to boot SSH to decrypt root partition
nix run .#myskarabox-boot-ssh

# Or use the unlock command specifically
nix run .#myskarabox-unlock
```

**The unlock process**:
- Connects to the initrd SSH server (runs during boot)
- Prompts for your ZFS encryption passphrase
- Unlocks the root pool, allowing boot to continue
- Data pool auto-unlocks using keys stored in `/persist`

### 7. Verify Installation

```bash
# SSH into the running system
nix run .#myskarabox-ssh

# Check ZFS pools are mounted
zpool status
zfs list
```

## Recovery Mode Specific Considerations

When using Hetzner recovery mode:

1. **Network Configuration**: Recovery mode provides DHCP networking by default
2. **SSH Access**: Uses root login with rescue password or SSH keys
3. **Disk Access**: Full access to all disks for partitioning and installation
4. **Time Limit**: Recovery mode expires after 60 minutes of inactivity

## Troubleshooting Common Issues

### Cross-Platform Build Issues

1. **macOS Build Error**: `Cannot build '/nix/store/...-nixpkgs-patched.drv'. Reason: required system or feature not available`
   - **Cause**: Commands like `gen-knownhosts-file` and `get-facter` try to build x86_64-linux packages on aarch64-darwin
   - **Solution**: Use Option B (Server-Side Setup), Option C (Manual Known Hosts), or manual SSH commands from Steps 1-2

2. **Cross-Compilation Failures**:
   - Some Nix commands require building for the target system architecture
   - Run problematic commands directly on the target server in recovery mode
   - Alternative: Configure remote builders or use Linux VM

### Installation Issues

3. **SSH Connection Fails**:
   - Verify server is in recovery mode
   - Check IP address is correct
   - Ensure SSH keys are properly configured

4. **Hardware Detection Fails**:
   - Server might not be fully booted
   - Try running facter command again after a few minutes

5. **Installation Hangs**:
   - Check network connectivity
   - Verify disk space and layout
   - Monitor installation logs

6. **Boot Decryption Fails**:
   - Ensure you're using the correct passphrase
   - Check that initrd SSH is accessible
   - Verify network configuration allows connections to boot SSH port

## Key Points

- **Flexible execution**: Commands can run locally (Linux) or on the server (for cross-platform issues)
- **nixos-anywhere handles remote installation**: No need to manually run commands on the server during installation
- **Recovery mode provides clean slate**: Useful when server isn't accessible or needs fresh install
- **Cross-platform considerations**: macOS users may need to run some commands on the target server
- **ZFS encryption requires manual unlock**: After installation, you must decrypt the root pool to complete boot
- **hcloud CLI integration**: Automates server IP and architecture detection

This process leverages Skarabox's automation to handle the complex NixOS installation with ZFS encryption, while hcloud CLI provides convenient server management integration.