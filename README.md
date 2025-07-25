I'll update the guide to use WSL2 for the entire build process with your specified paths:

## Complete Zorin OS to WSL2 Distribution Guide (Building from WSL2)

### Phase 1: Prepare Environment in WSL2

#### 1.1 Open WSL2 & Create Working Directory
```bash
# Create working directory
mkdir -p ~/nb-os
cd ~/nb-os
```

#### 1.2 Install Required Tools
```bash
# Install necessary packages
sudo apt update
sudo apt install -y squashfs-tools p7zip-full p7zip-rar wget curl jq imagemagick
```

#### 1.3 Create Swapfile VHDX (Run in PowerShell as Admin)
```powershell
# In Windows PowerShell (Admin)
New-VHD -Path "\\wsl.localhost\Ubuntu\home\example\nb-os\swap.vhdx" -SizeBytes 4GB -Dynamic
```

#### 1.4 Configure .wslconfig (In Windows)
```bash
# From WSL2, create the config file
cat > /mnt/c/Users/example/.wslconfig << 'EOF'
[wsl2]
memory=8GB
processors=4
kernel=C:\\Windows\\System32\\lxss\\tools\\kernel
kernelCommandLine=vsyscall=emulate
swapSize=4GB
swapfile=\\\\wsl.localhost\\Ubuntu\\home\\example\\nb-os\\swap.vhdx
pageReporting=true
localhostforwarding=true
nestedVirtualization=true
debugConsole=true

[experimental]
sparseVhd=true
EOF
```

### Phase 2: Extract Zorin OS Rootfs

#### 2.1 Mount & Extract ISO
```bash
# Download Zorin OS ISO
cd ~/nb-os
sudo wget https://distro.ibiblio.org/zorinos/17/Zorin-OS-17.3-Core-64-bit-r2.iso -O nb-os.iso

# Set ISO path
ISO_PATH="~/nb-os/nb-os.iso"

# Create mount point and mount ISO
sudo mkdir -p /mnt/nb-os
sudo mount -o loop "$ISO_PATH" /mnt/nb-os

# Copy filesystem.squashfs
cp /mnt/nb-os/casper/filesystem.squashfs ~/nb-os/

# Extract the squashfs
cd ~/nb-os
sudo unsquashfs -d rootfs filesystem.squashfs
```

### Phase 3: Prepare Rootfs for WSL

#### 3.1 Setup Chroot Environment
```bash
# Setup chroot mounts
sudo mount --bind /dev rootfs/dev
sudo mount --bind /dev/pts rootfs/dev/pts
sudo mount --bind /proc rootfs/proc
sudo mount --bind /sys rootfs/sys

# Copy resolv.conf for network access in chroot
sudo cp /etc/resolv.conf rootfs/etc/resolv.conf

# Enter chroot
sudo chroot rootfs /bin/bash
```

#### 3.2 Clean & Prepare System (Inside Chroot)
```bash
# Update package database
apt update

# Remove GUI and unnecessary packages
apt remove --purge -y \
    snapd \
    plymouth* \
    xserver-xorg* \
    xwayland \
    gdm3 \
    gnome-shell \
    gnome-session* \
    zorin-desktop-session \
    zorin-appearance \
    zorin-exec-guard \
    network-manager \
    modemmanager \
    linux-image-* \
    linux-headers-* \
    linux-modules-* \
    grub* \
    initramfs-tools* \
    ubuntu-desktop* \
    thunderbird* \
    libreoffice* \
    firefox* \
    cups* \
    bluetooth* \
    pulseaudio*

# Install WSL utilities
apt install -y \
    wslu \
    ubuntu-wsl \
    openssh-server \
    curl \
    wget \
    git \
    nano \
    vim \
    htop \
    build-essential

# Clean apt cache
apt autoremove -y
apt clean
```

#### 3.3 Configure WSL-specific Settings (Inside Chroot)
```bash
# Create wsl.conf
cat > /etc/wsl.conf << 'EOF'
[boot]
systemd=true

[automount]
enabled = true
root = /mnt/
options = "metadata,uid=1000,gid=1000,umask=022,fmask=111"
mountFsTab = true

[user]
default = nb-user

[network]
hostname = nb-os
generateHosts = true
generateResolvConf = true

[interop]
enabled = true
appendWindowsPath = true
EOF

# Create wsl-distribution.conf
cat > /etc/wsl-distribution.conf << 'EOF'
[oobe]
command = /etc/oobe.sh
defaultUid = 1000
defaultName = nb-user

[shortcut]
icon = /usr/share/icons/nb-os.ico

[windowsterminal]
ProfileTemplate = /usr/lib/wsl/terminal.json
EOF
```

#### 3.4 Create OOBE Script (Inside Chroot)
```bash
cat > /etc/oobe.sh << 'EOF'
#!/bin/bash

set -ue

DEFAULT_GROUPS='adm,cdrom,sudo,dip,plugdev'
DEFAULT_UID='1000'
DEFAULT_USER='nb-user'

echo '╔═══════════════════════════════════════════╗'
echo '║      Welcome to NB-OS WSL Distribution    ║'
echo '║    NottyBoi OS - Powerful Yet Friendly    ║'
echo '╚═══════════════════════════════════════════╝'
echo

if getent passwd "$DEFAULT_UID" > /dev/null ; then
  echo '✓ Default user already exists, skipping creation'
  exit 0
fi

# Create default user
echo "Creating user: $DEFAULT_USER"
/usr/sbin/adduser --uid "$DEFAULT_UID" --disabled-password --gecos '' "$DEFAULT_USER"
/usr/sbin/usermod "$DEFAULT_USER" -aG "$DEFAULT_GROUPS"

# Set password
echo
echo "Please set a password for user $DEFAULT_USER:"
passwd "$DEFAULT_USER"

echo
echo "✓ NB-OS setup complete! You can now use 'wsl -d nb-os' to start."
EOF

chmod +x /etc/oobe.sh
```

#### 3.5 Create Terminal Profile (Inside Chroot)
```bash
mkdir -p /usr/lib/wsl
cat > /usr/lib/wsl/terminal.json << 'EOF'
{
  "profiles": [
    {
      "name": "NB-OS",
      "commandline": "wsl.exe -d nb-os",
      "startingDirectory": "~",
      "colorScheme": "Zorin Dark",
      "fontFace": "Cascadia Code",
      "fontSize": 11,
      "antialiasingMode": "cleartype",
      "cursorShape": "filledBox",
      "useAcrylic": true,
      "acrylicOpacity": 0.85
    }
  ],
  "schemes": [
    {
      "name": "Zorin Dark",
      "background": "#1E1E2E",
      "foreground": "#CDD6F4",
      "black": "#45475A",
      "red": "#F38BA8",
      "green": "#A6E3A1",
      "yellow": "#F9E2AF",
      "blue": "#89B4FA",
      "purple": "#F5C2E7",
      "cyan": "#94E2D5",
      "white": "#BAC2DE",
      "brightBlack": "#585B70",
      "brightRed": "#F38BA8",
      "brightGreen": "#A6E3A1",
      "brightYellow": "#F9E2AF",
      "brightBlue": "#89B4FA",
      "brightPurple": "#F5C2E7",
      "brightCyan": "#94E2D5",
      "brightWhite": "#A6ADC8",
      "selectionBackground": "#F5E0DC",
      "cursorColor": "#F5E0DC"
    }
  ]
}
EOF
```

#### 3.6 Configure Systemd for WSL (Inside Chroot)
```bash
# Disable problematic services
systemctl mask \
    systemd-resolved.service \
    systemd-networkd.service \
    NetworkManager.service \
    systemd-tmpfiles-setup.service \
    systemd-tmpfiles-clean.service \
    systemd-tmpfiles-clean.timer \
    systemd-tmpfiles-setup-dev-early.service \
    systemd-tmpfiles-setup-dev.service \
    tmp.mount

# Enable useful services
systemctl enable ssh.service || true
systemctl enable cron.service || true
```

#### 3.7 Clean Security Files (Inside Chroot)
```bash
# Clear password hashes
sed -i 's/:[^:]*:/:!:/' /etc/shadow

# Remove machine-id
rm -f /etc/machine-id
touch /etc/machine-id

# Clear SSH host keys
rm -f /etc/ssh/ssh_host_*

# Remove existing resolv.conf
rm -f /etc/resolv.conf

# Clean logs and temp files
find /var/log -type f -delete
find /tmp -type f -delete
find /var/tmp -type f -delete

# Remove apt lists
rm -rf /var/lib/apt/lists/*
```

#### 3.8 Exit Chroot
```bash
# Exit chroot
exit

# Back in WSL2, unmount everything
cd ~/nb-os
sudo umount rootfs/sys
sudo umount rootfs/proc
sudo umount rootfs/dev/pts
sudo umount rootfs/dev

# Unmount ISO
sudo umount /mnt/nb-os
```

### Phase 4: Create Distribution Icon

#### 4.1 Create Distribution Icon

**Option 1: Use Your Own Custom Icon**
```bash
cd ~/nb-os

# Place your custom icon file in one of these locations with one of these names:
# - /mnt/c/Users/example/Downloads/nb-os-icon.png
# - /mnt/c/Users/example/Downloads/nb-os-icon.jpg
# - /mnt/c/Users/example/Downloads/nb-os-icon.svg
# - ~/nb-os/custom-icon.png (or .jpg, .svg)

# Script will automatically detect and use your custom icon
CUSTOM_ICON=""

# Check for custom icon in Windows Downloads folder
for ext in png jpg jpeg svg; do
    if [ -f "/mnt/c/Users/example/Downloads/nb-os-icon.$ext" ]; then
        CUSTOM_ICON="/mnt/c/Users/example/Downloads/nb-os-icon.$ext"
        break
    fi
done

# Check for custom icon in current directory
if [ -z "$CUSTOM_ICON" ]; then
    for ext in png jpg jpeg svg; do
        if [ -f "custom-icon.$ext" ]; then
            CUSTOM_ICON="custom-icon.$ext"
            break
        fi
    done
fi

# Use custom icon if found
if [ ! -z "$CUSTOM_ICON" ]; then
    echo "✓ Using custom icon: $CUSTOM_ICON"
    convert "$CUSTOM_ICON" -resize 256x256 -define icon:auto-resize="256,128,64,48,32,16" nb-os.ico
else
    echo "No custom icon found, using default options..."
    
    # Try to download Zorin OS logo
    if wget -q -O zorin-logo.png "https://upload.wikimedia.org/wikipedia/commons/thumb/f/f2/Zorin_OS_logo.svg/256px-Zorin_OS_logo.svg.png"; then
        echo "✓ Downloaded Zorin OS logo"
        convert zorin-logo.png -resize 256x256 -define icon:auto-resize="256,128,64,48,32,16" nb-os.ico
        rm -f zorin-logo.png
    else
        # Create a simple branded icon as fallback
        echo "✓ Creating simple NB-OS branded icon"
        convert -size 256x256 xc:"#0084FF" \
                -fill white -gravity center \
                -pointsize 32 -annotate +0-20 "NB" \
                -pointsize 20 -annotate +0+20 "OS" \
                -stroke white -strokewidth 2 \
                -fill none -draw "roundrectangle 40,40 216,216 20,20" \
                nb-os.ico
    fi
fi

# Verify the icon was created
if [ -f "nb-os.ico" ]; then
    echo "✓ Icon created successfully: nb-os.ico"
    ls -lh nb-os.ico
else
    echo "✗ Failed to create icon"
    exit 1
fi

# Copy to rootfs
sudo cp nb-os.ico rootfs/usr/share/icons/nb-os.ico
```

**Option 2: Quick Custom Icon Setup**
```bash
# If you want to quickly add your own icon, you can also do:
# 1. Copy your icon file to the nb-os directory
# cp /path/to/your/icon.png ~/nb-os/custom-icon.png

# 2. Or download an icon directly
# wget -O custom-icon.png "https://your-icon-url.com/icon.png"

# 3. Then run the icon creation script above
```

### Phase 5: Package Distribution

#### 5.1 Create Tarball
```bash
cd ~/nb-os

# Create the tarball with proper exclusions
sudo tar --numeric-owner \
    --exclude='proc/*' \
    --exclude='sys/*' \
    --exclude='dev/*' \
    --exclude='run/*' \
    --exclude='tmp/*' \
    --exclude='var/cache/apt/*' \
    --exclude='var/lib/apt/lists/*' \
    --exclude='var/log/*' \
    --exclude='boot/*' \
    --exclude='media/*' \
    --exclude='mnt/*' \
    --exclude='home/*' \
    --exclude='root/*' \
    -czf nb-os.tar.gz -C rootfs .

# Verify tarball size
ls -lh nb-os.tar.gz
```

#### 5.2 Create WSL Extension File
```bash
# Copy as .wsl extension
cp nb-os.tar.gz nb-os.wsl
```

### Phase 6: Test Installation

#### 6.1 Create Test Manifest Script
```bash
# Create PowerShell script for testing
cat > ~/nb-os/test-manifest.ps1 << 'EOF'
#Requires -RunAsAdministrator

$TarPath = "\\wsl.localhost\Ubuntu\home\example\nb-os\nb-os.tar.gz"
$hash = (Get-FileHash $TarPath -Algorithm SHA256).Hash

$manifest = @{
    ModernDistributions = @{
        "nb-os" = @(
            @{
                "Name" = "nb-os-v1"
                Default = $true
                FriendlyName = "NB-OS (Zorin-based)"
                Amd64Url = @{
                    Url = "file:///$TarPath"
                    Sha256 = "0x$hash"
                }
            }
        )
    }
}

$manifestFile = "\\wsl.localhost\Ubuntu\home\example\nb-os\manifest.json"
$manifest | ConvertTo-Json -Depth 5 | Out-File -Encoding ASCII $manifestFile

Set-ItemProperty -Path "HKLM:SOFTWARE\Microsoft\Windows\CurrentVersion\Lxss" `
    -Name DistributionListUrl -Value "file:///$manifestFile" -Type String -Force

Write-Host "Manifest created successfully!"
Write-Host "Run 'wsl --list --online' to verify"
EOF
```

#### 6.2 Install Distribution (From PowerShell Admin)
```powershell
# Method 1: Direct import
wsl --import nb-os "C:\Users\example\nb-os-instance" "\\wsl.localhost\Ubuntu\home\example\nb-os\nb-os.tar.gz" --version 2

# Method 2: Double-click the .wsl file in File Explorer

# Method 3: Using test manifest
cd \\wsl.localhost\Ubuntu\home\example\nb-os\
.\test-manifest.ps1
wsl --list --online
wsl --install nb-os-v1
```

### Phase 7: Post-Installation Setup

#### 7.1 Configure Distribution
```powershell
# Ensure it's WSL2
wsl --set-version nb-os 2

# Mount swap file
Mount-VHD -Path "\\wsl.localhost\Ubuntu\home\example\nb-os\swap.vhdx"

# Set as default (optional)
wsl --set-default nb-os
```

#### 7.2 First Run & Verification
```bash
# Start the distribution
wsl -d nb-os

# Verify system info
cat /etc/os-release
uname -a
systemctl status

# Test Windows interop
explorer.exe .
```

### Phase 8: Cleanup & Additional Formats

#### 8.1 Create ISO for Desktop Installation (Optional)
```bash
# If you want to create a bootable ISO with GUI for desktop installation
cd ~/nb-os

# Create a new rootfs copy for desktop version
sudo cp -r rootfs rootfs-desktop

# Re-enter chroot for desktop version
sudo mount --bind /dev rootfs-desktop/dev
sudo mount --bind /dev/pts rootfs-desktop/dev/pts
sudo mount --bind /proc rootfs-desktop/proc
sudo mount --bind /sys rootfs-desktop/sys
sudo cp /etc/resolv.conf rootfs-desktop/etc/resolv.conf

sudo chroot rootfs-desktop /bin/bash

# Inside chroot - reinstall desktop components
apt update
apt install -y \
    zorin-desktop-session \
    zorin-appearance \
    gnome-shell \
    gdm3 \
    xserver-xorg \
    ubuntu-desktop-minimal \
    linux-image-generic \
    grub-pc \
    casper \
    lupin-casper \
    discover1 \
    laptop-detect \
    os-prober

# Exit chroot
exit

# Unmount desktop rootfs
sudo umount rootfs-desktop/sys
sudo umount rootfs-desktop/proc
sudo umount rootfs-desktop/dev/pts
sudo umount rootfs-desktop/dev

# Create new squashfs for desktop
sudo mksquashfs rootfs-desktop filesystem-desktop.squashfs -comp xz

# Create ISO structure (requires additional ISO creation tools)
# This is a simplified example - full ISO creation requires more steps
sudo mkdir -p iso-build/{casper,isolinux,install}
sudo cp filesystem-desktop.squashfs iso-build/casper/filesystem.squashfs
# ... additional ISO creation steps would go here
```

#### 8.2 Cleanup Working Files
```bash
# Remove working files to save space
cd ~/nb-os
rm -f filesystem.squashfs
rm -f nb-os.iso
sudo rm -rf rootfs/
sudo rm -rf rootfs-desktop/ # if you created the desktop version
```