# Instana Preparation Setup

## 1.1. Prepare the Server
```bash
#!/bin/bash
# Instana Storage Setup Script
# Compatible with RHEL/Ubuntu

set -e

echo "=== Instana Storage Configuration Script ==="

# Create directories
echo "[1/6] Creating mount directories..."
mkdir -p /mnt/instana/stanctl/data
mkdir -p /mnt/instana/stanctl/metrics
mkdir -p /mnt/instana/stanctl/analytics
mkdir -p /mnt/instana/stanctl/objects

# Format disks
echo "[2/6] Formatting disks..."
for disk in sd{c..f}; do
   echo "  → Formatting /dev/$disk..."
   mkfs.xfs -f -i size=1024 -L $disk /dev/$disk
done

# Generate fstab entries
echo "[3/6] Generating fstab entries..."
i=0
for disk in /dev/sd{c..f}; do 
  dirs=("/mnt/instana/stanctl/data" "/mnt/instana/stanctl/metrics" "/mnt/instana/stanctl/analytics" "/mnt/instana/stanctl/objects")
  uuid=$(blkid -s UUID -o value $disk)
  echo "UUID=$uuid ${dirs[$i]} xfs discard,defaults,nofail 0 0" >> result.txt
  ((i++))
done

# Backup and update fstab
echo "[4/6] Updating /etc/fstab..."
cp /etc/fstab /etc/fstab.backup.$(date +%Y%m%d_%H%M%S)
cat result.txt >> /etc/fstab
echo "  → Backup created: /etc/fstab.backup.*"
echo "  → Mounting filesystems..."
mount -a

# Disable firewall
echo "[5/6] Disabling firewall..."
if command -v firewall-cmd &> /dev/null; then
    # RHEL/CentOS
    systemctl stop firewalld
    systemctl disable firewalld
    echo "  → firewalld disabled"
elif command -v ufw &> /dev/null; then
    # Ubuntu/Debian
    ufw disable
    echo "  → ufw disabled"
else
    echo "  → No firewall detected (firewalld/ufw)"
fi

# System tuning
echo "[6/6] Applying system tuning..."
cat >> /etc/sysctl.d/99-stanctl.conf << EOF
vm.swappiness=0
fs.inotify.max_user_instances=8192
EOF
sysctl -p /etc/sysctl.d/99-stanctl.conf

# Update GRUB
echo "  → Updating GRUB configuration..."
sed -i "s/\(GRUB_CMDLINE_LINUX=\".*\)\"/\1 transparent_hugepage=never\"/" /etc/default/grub

if [ -f /etc/redhat-release ]; then
    # RHEL/CentOS
    grub2-mkconfig -o /boot/grub2/grub.cfg
    echo "  → GRUB updated (RHEL)"
elif [ -f /etc/debian_version ]; then
    # Ubuntu/Debian
    update-grub
    echo "  → GRUB updated (Ubuntu)"
fi

echo ""
echo "=== Configuration Complete ==="
echo "⚠️  REBOOT REQUIRED for GRUB changes to take effect"
echo ""
echo "Verification commands:"
echo "  df -h | grep instana"
echo "  cat /etc/fstab | grep instana"
echo ""
```

## 1.2. Install Stanctl 
```bash
#!/bin/bash
# Instana Repository Setup Script with Backup
# Compatible with Debian/Ubuntu

set -euo pipefail

#======================================
# CONFIGURATION VARIABLES
#======================================
DOWNLOAD_KEY="${DOWNLOAD_KEY:-}"           # Set via environment or edit here
BACKUP_DIR="/root/instana_backup_$(date +%Y%m%d_%H%M%S)"
REPO_URL="https://artifact-public.instana.io/artifactory"
REPO_FILE="/etc/apt/sources.list.d/instana-product.list"
AUTH_FILE="/etc/apt/auth.conf"
KEYRING_FILE="/usr/share/keyrings/instana-archive-keyring.gpg"

#======================================
# FUNCTIONS
#======================================
log() {
    echo "[$(date +'%Y-%m-%d %H:%M:%S')] $*"
}

error_exit() {
    log "ERROR: $1"
    exit 1
}

backup_file() {
    local file=$1
    if [ -f "$file" ]; then
        local backup_path="$BACKUP_DIR/$(basename $file)"
        cp -p "$file" "$backup_path"
        log "✓ Backed up: $file → $backup_path"
    fi
}

rollback() {
    log "⚠️  Rolling back changes..."
    if [ -d "$BACKUP_DIR" ]; then
        [ -f "$BACKUP_DIR/$(basename $REPO_FILE)" ] && cp -p "$BACKUP_DIR/$(basename $REPO_FILE)" "$REPO_FILE"
        [ -f "$BACKUP_DIR/$(basename $AUTH_FILE)" ] && cp -p "$BACKUP_DIR/$(basename $AUTH_FILE)" "$AUTH_FILE"
        [ -f "$BACKUP_DIR/$(basename $KEYRING_FILE)" ] && cp -p "$BACKUP_DIR/$(basename $KEYRING_FILE)" "$KEYRING_FILE"
        log "✓ Rollback completed"
    fi
    exit 1
}

#======================================
# MAIN SCRIPT
#======================================
trap rollback ERR

log "=== Instana Repository Setup ==="

# Validate root
if [ "$EUID" -ne 0 ]; then 
    error_exit "This script must be run as root"
fi

# Validate DOWNLOAD_KEY
if [ -z "$DOWNLOAD_KEY" ]; then
    read -sp "Enter Instana Download Key: " DOWNLOAD_KEY
    echo
    if [ -z "$DOWNLOAD_KEY" ]; then
        error_exit "DOWNLOAD_KEY is required"
    fi
fi

log "Download key: ${DOWNLOAD_KEY:0:10}...${DOWNLOAD_KEY: -10}"

# Create backup directory
log ""
log "[1/6] Creating backup directory..."
mkdir -p "$BACKUP_DIR"
log "✓ Backup directory: $BACKUP_DIR"

# Backup existing files
log ""
log "[2/6] Backing up existing files..."
backup_file "$REPO_FILE"
backup_file "$AUTH_FILE"
backup_file "$KEYRING_FILE"

# Configure repository
log ""
log "[3/6] Configuring Instana repository..."
echo "deb [signed-by=$KEYRING_FILE] $REPO_URL/rel-debian-public-virtual generic main" > "$REPO_FILE"
log "✓ Repository configured: $REPO_FILE"

# Configure authentication
log ""
log "[4/6] Configuring authentication..."
cat > "$AUTH_FILE" << EOF
machine artifact-public.instana.io
  login _
  password $DOWNLOAD_KEY
EOF
chmod 600 "$AUTH_FILE"
log "✓ Authentication configured: $AUTH_FILE"

# Download and install GPG key
log ""
log "[5/6] Installing GPG keyring..."
wget -nv -O- --user=_ --password="$DOWNLOAD_KEY" \
    "$REPO_URL/api/security/keypair/public/repositories/rel-debian-public-virtual" \
    | gpg --dearmor > "$KEYRING_FILE"
log "✓ GPG keyring installed: $KEYRING_FILE"

# Update and install
log ""
log "[6/6] Installing stanctl..."
apt update -y
apt install -y stanctl

log ""
log "=== Installation Complete ==="
log "Backup location: $BACKUP_DIR"
log "Installed packages:"
dpkg -l | grep stanctl || true
log ""
```
