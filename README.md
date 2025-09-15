# SN_TejasSawant
# Personal Home Cloud with Secure Multi-Level Access

A comprehensive guide to building a secure, self-hosted cloud server for personal and family use. This system provides role-based access control, guest portals, and is securely exposed to the internet for global accessibility.

![Network Architecture](https://via.placeholder.com/800x400.png?text=Network+Diagram+Placeholder) 
*Note: Replace with your actual network diagram*

## 📋 Project Overview

This project implements a secure home-based cloud server that provides role-based access to personal and family data. Different users (yourself, parents, siblings, visiting relatives) get customized levels of access through an internet-exposed system with secure remote access.

## 🛠️ Hardware Requirements

- **Beelink Mini-PC with N100 CPU** (or similar)
- **USB Drive**: 8GB+ for Proxmox installation
- **Storage**: NVMe/SSD for Proxmox OS, additional HDDs for data storage
- **Client PC**: For preparing boot media and accessing web UI

## 🔧 Installation Guide

### Phase 1: Proxmox VE Installation

1. **Prepare Installation Media**
   - Download Proxmox VE ISO from official site
   - Flash to USB drive using BalenaEtcher or similar tool

2. **BIOS Configuration**
   - Enter BIOS (Del or F7 for Beelink devices)
   - Disable Secure Boot
   - Enable Virtualization (VT-x/AMD-V, IOMMU)
   - Set USB as first boot device

3. **Install Proxmox VE**
   - Boot from USB and select "Install Proxmox VE"
   - Choose target disk (NVMe/SSD recommended)
   - Configure:
     - Filesystem: ext4 or ZFS (ZFS requires more RAM)
     - Country/keyboard/timezone settings
     - Management interface: Select Ethernet NIC
     - Set static IP (e.g., 192.168.1.100)
     - Root password and email

4. **Post-Installation**
   - Reboot and remove USB drive
   - Access web interface at `https://<Proxmox-IP>:8006`
   - Login with root credentials

### Phase 2: Network Security & Segmentation

#### VLAN Configuration
Create the following VLAN structure on your router (OPNsense/pfSense recommended):

- **VLAN 10 - Trusted LAN**: Personal computers and phones
- **VLAN 20 - Server VLAN**: Proxmox host and services (isolated)
- **VLAN 30 - Guest IoT VLAN**: Visitors and IoT devices (restricted)

#### Firewall Rules
Implement strict firewall policies:

```bash
# Trusted LAN → Server VLAN: Allow HTTPS only
allow VLAN10 → VLAN20 TCP/443

# Guest VLAN: Internet access only, block internal traffic
allow VLAN30 → Internet TCP/80,443, UDP/53
block VLAN30 → VLAN10, VLAN20
```

#### Secure Remote Access
Set up WireGuard VPN on your firewall for secure remote access:
- Users connect to VPN first
- Then access services as if on local network
- Only expose VPN port to internet

### Phase 3: Service Deployment & Access Control

#### Nextcloud Installation in LXC Container

1. **Create LXC Container**
   - Use Debian 12 template
   - Allocate 2+ CPU cores, 4+ GB RAM
   - Attach to VLAN 20 (Server VLAN)

2. **Install Dependencies**
   ```bash
   apt update && apt upgrade -y
   apt install nginx mariadb-server php-fpm php-mysql \
   php-xml php-zip php-gd php-curl php-mbstring php-intl unzip -y
   ```

3. **Database Setup**
   ```sql
   CREATE DATABASE nextcloud;
   CREATE USER 'ncuser'@'localhost' IDENTIFIED BY 'StrongPassw0rd!';
   GRANT ALL PRIVILEGES ON nextcloud.* TO 'ncuser'@'localhost';
   FLUSH PRIVILEGES;
   ```

4. **Install Nextcloud**
   ```bash
   wget https://download.nextcloud.com/server/releases/latest.zip
   unzip latest.zip
   mv nextcloud /var/www/
   chown -R www-data:www-data /var/www/nextcloud
   ```

5. **Nginx Configuration**
   Create `/etc/nginx/sites-available/nextcloud.conf` with proper server block configuration

6. **Secure with HTTPS**
   - Use Let's Encrypt for SSL certificates
   - Force HTTPS redirects

#### Access Control Implementation

1. **User and Group Management**
   - Create users: `alice`, `bob`, `mom`, `dad`, `guest_jane`
   - Create groups: `Family`, `Adults`, `Kids`, `Guests`

2. **Folder Permissions**
   ```
   /Alice → RW for alice only
   /Bob → RW for bob only
   /Family-Vacation → RW for Adults, RO for Kids
   /Public → RO for Guests, RW for Family
   ```

3. **Guest Access Portal**
   - Create guest accounts in `Guests` group
   - Place shared files in `/Public` folder
   - Implement time-based access (manual disable or cron job)

### Phase 4: Encryption & Monitoring

#### Disk Encryption
**Option A: Proxmox with LUKS (Recommended)**
- Install Debian first with encrypted LVM
- Then install Proxmox on top
- Entire storage is encrypted at rest

**Option B: Encrypt Additional Volume**
```bash
# Format disk with LUKS
cryptsetup luksFormat /dev/sdb

# Open encrypted device
cryptsetup open /dev/sdb securedata

# Create filesystem and mount
mkfs.ext4 /dev/mapper/securedata
mkdir /mnt/securedata
mount /dev/mapper/securedata /mnt/securedata
```

#### Monitoring Setup
- Install Fail2Ban for intrusion prevention
- Configure firewall logging
- Set up regular security updates

## 🔒 Security Features

- **Network Segmentation**: Isolated VLANs for different trust levels
- **Role-Based Access Control**: Granular permissions for users and groups
- **Encryption**: Data encrypted at rest (LUKS) and in transit (HTTPS/VPN)
- **Secure Remote Access**: WireGuard VPN instead of port forwarding
- **Monitoring**: Logging and intrusion detection systems

## 🌐 Remote Access

1. Connect to home WireGuard VPN
2. Access Nextcloud at `https://<server-local-ip>`
3. All traffic encrypted through VPN tunnel

## 📝 Maintenance

- Regular security updates
- Backup encryption keys securely
- Monitor logs for suspicious activity
- Review user access permissions periodically

## 🆘 Troubleshooting

Common issues and solutions:

1. **Can't access Proxmox web interface**
   - Check firewall rules between VLANs
   - Verify IP configuration

2. **Nextcloud installation errors**
   - Verify PHP modules are installed
   - Check database permissions

3. **VPN connection issues**
   - Verify port forwarding on router
   - Check client configuration

## 📚 Resources

- [Proxmox VE Documentation](https://pve.proxmox.com/wiki/Main_Page)
- [Nextcloud Administration Manual](https://docs.nextcloud.com/server/latest/admin_manual/)
- [WireGuard Documentation](https://www.wireguard.com/)

## 👥 Contributing

This project is open to improvements and suggestions. Please feel free to submit issues or pull requests.

## 📄 License

This project is provided for educational purposes. Use at your own risk in production environments.

---

*Disclaimer: This setup requires technical knowledge. Proper security implementation is essential when exposing services to the internet.*
