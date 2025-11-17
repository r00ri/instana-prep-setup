# instana-prep-setup

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
