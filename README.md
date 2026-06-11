#!/bin/bash
# Mint Workstation Bootstrap v1.4 (Idempotent Edition)

set -euo pipefail

REPORT="$HOME/mint-bootstrap-report.txt"
touch "$REPORT"

log() {
  echo "[$(date '+%F %T')] $1"
}

install_if_missing() {
  local pkg="$1"
  dpkg -s "$pkg" >/dev/null 2>&1 || sudo apt install -y "$pkg"
}

log "Starting Mint Bootstrap v1.4"

sudo apt update
sudo apt upgrade -y

# Package lists
CORE_PKGS=(curl wget git vim nano tree htop btop unzip p7zip-full net-tools openssh-server openssh-client ca-certificates software-properties-common)
SAMBA_PKGS=(samba smbclient cifs-utils)
VIRT_PKGS=(qemu-kvm libvirt-daemon-system libvirt-clients virt-manager podman podman-compose docker.io freerdp3-x11)
WINE_PKGS=(wine64 wine32 winetricks winbind)
VULKAN_PKGS=(mesa-vulkan-drivers vulkan-tools mesa-utils)
OTHER_PKGS=(steam-installer virtualbox vlc copyq)

for pkg in "${CORE_PKGS[@]}" "${SAMBA_PKGS[@]}" "${VIRT_PKGS[@]}" "${WINE_PKGS[@]}" "${VULKAN_PKGS[@]}" "${OTHER_PKGS[@]}"; do
  install_if_missing "$pkg"
done

# VS Code repo (idempotent)
if [ ! -f /etc/apt/sources.list.d/vscode.list ]; then
  wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > /tmp/packages.microsoft.gpg
  sudo install -D -o root -g root -m 644 /tmp/packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
  echo "deb [arch=amd64 signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" | sudo tee /etc/apt/sources.list.d/vscode.list >/dev/null
  sudo apt update
fi

dpkg -s code >/dev/null 2>&1 || sudo apt install -y code

# Services
for svc in docker smbd ssh; do
  sudo systemctl enable "$svc" >/dev/null 2>&1 || true
  sudo systemctl start "$svc" >/dev/null 2>&1 || true
done

# Groups
for grp in docker libvirt kvm vboxusers sambashare; do
  getent group "$grp" >/dev/null || sudo groupadd "$grp"
  groups "$USER" | grep -qw "$grp" || sudo usermod -aG "$grp" "$USER"
done

# Samba Share
mkdir -p "$HOME/WinShare"
chmod 777 "$HOME/WinShare"

if ! grep -q "^\[WinShare\]" /etc/samba/smb.conf; then
sudo tee -a /etc/samba/smb.conf >/dev/null <<EOF

[WinShare]
path = /home/$USER/WinShare
browseable = yes
read only = no
guest ok = yes
force user = $USER
create mask = 0666
directory mask = 0777
EOF
fi

sudo systemctl restart smbd

# CopyQ autostart
mkdir -p "$HOME/.config/autostart"
if [ -f /usr/share/applications/com.github.hluk.copyq.desktop ] && [ ! -f "$HOME/.config/autostart/com.github.hluk.copyq.desktop" ]; then
  cp /usr/share/applications/com.github.hluk.copyq.desktop "$HOME/.config/autostart/"
fi

# Wine init
if [ ! -d "$HOME/.wine" ]; then
  wineboot -u || true
fi

# DXVK/VKD3D markers
mkdir -p "$HOME/.local/share/mint-bootstrap"
if [ ! -f "$HOME/.local/share/mint-bootstrap/dxvk.done" ]; then
  winetricks -q dxvk || true
  touch "$HOME/.local/share/mint-bootstrap/dxvk.done"
fi

if [ ! -f "$HOME/.local/share/mint-bootstrap/vkd3d.done" ]; then
  winetricks -q vkd3d || true
  touch "$HOME/.local/share/mint-bootstrap/vkd3d.done"
fi

# Hibernation
SWAP_UUID=$(sudo blkid | awk -F'"' '/TYPE="swap"/ {print $2; exit}')

if [ -n "$SWAP_UUID" ]; then
  sudo mkdir -p /etc/initramfs-tools/conf.d

  CURRENT_RESUME=""
  [ -f /etc/initramfs-tools/conf.d/resume ] && CURRENT_RESUME=$(grep "^RESUME=" /etc/initramfs-tools/conf.d/resume || true)

  if [ "$CURRENT_RESUME" != "RESUME=UUID=$SWAP_UUID" ]; then
    echo "RESUME=UUID=$SWAP_UUID" | sudo tee /etc/initramfs-tools/conf.d/resume >/dev/null
    sudo update-initramfs -u
  fi
else
  if [ ! -f /swapfile ]; then
    sudo fallocate -l 20G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    grep -q "^/swapfile" /etc/fstab || echo "/swapfile none swap sw 0 0" | sudo tee -a /etc/fstab >/dev/null
  fi
fi

# Aliases (deduplicated)
touch "$HOME/.bash_aliases"

add_alias() {
  local alias_line="$1"
  grep -Fqx "$alias_line" "$HOME/.bash_aliases" || echo "$alias_line" >> "$HOME/.bash_aliases"
}

add_alias "alias ll='ls -alF'"
add_alias "alias la='ls -A'"
add_alias "alias l='ls -CF'"
add_alias "alias update='sudo apt update && sudo apt upgrade -y'"
add_alias "alias pod='podman'"
add_alias "alias dps='docker ps -a'"
add_alias "alias weather='curl wttr.in'"
add_alias "alias hibernate-now='systemctl hibernate'"
add_alias "alias myip='hostname -I'"
add_alias "alias ports='ss -tulpn'"
add_alias "alias mem='free -h'"
add_alias "alias disks='df -h'"

# Backup configs once
mkdir -p "$HOME/ConfigBackups"
[ -f "$HOME/.bashrc" ] && [ ! -f "$HOME/ConfigBackups/bashrc.bak" ] && cp "$HOME/.bashrc" "$HOME/ConfigBackups/bashrc.bak"
[ -f "$HOME/.bash_aliases" ] && [ ! -f "$HOME/ConfigBackups/bash_aliases.bak" ] && cp "$HOME/.bash_aliases" "$HOME/ConfigBackups/bash_aliases.bak"

# Health Report
{
echo "===== Mint Bootstrap v1.4 ====="
date
echo
echo "Wine:"
wine --version 2>/dev/null || true
echo
echo "Winetricks:"
winetricks --version 2>/dev/null || true
echo
echo "Docker:"
docker --version 2>/dev/null || true
echo
echo "Podman:"
podman --version 2>/dev/null || true
echo
echo "KVM:"
[ -e /dev/kvm ] && echo "Present" || echo "Missing"
echo
echo "Vulkan:"
vulkaninfo --summary | head -20 2>/dev/null || true
echo
echo "Swap:"
swapon --show
echo
echo "Samba:"
systemctl is-active smbd || true
} > "$REPORT"

echo
echo "======================================"
echo "Mint Bootstrap v1.4 Complete"
echo "======================================"
echo "Health report: $REPORT"
echo
echo "Optional next step:"
echo "sudo smbpasswd -a $USER"
echo
echo "Reboot recommended if first run."
