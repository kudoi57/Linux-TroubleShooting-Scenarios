# Linux-TroubleShooting-Scenarios
Linux Troubleshooting Scenarios based question with solution approach
Here’s a **detailed hierarchical troubleshooting guide** 

---

### **1. Server is Not Reachable or Cannot Connect**
```plaintext
├── Ping the server by Hostname and IP Address  
│   ├── **Hostname/IP is pingable**  
│   │   ├── Check if the **service/port** is accessible:  
│   │   │   ├── `telnet <IP> <port>` or `nc -zv <IP> <port>`.  
│   │   │   ├── If blocked:  
│   │   │   │   ├── Check server firewall:  
│   │   │   │   │   ├── `iptables -L -n` (legacy).  
│   │   │   │   │   ├── `firewall-cmd --list-all` (firewalld).  
│   │   │   │   ├── Check SELinux/AppArmor:  
│   │   │   │   │   ├── `ausearch -m avc` (SELinux denials).  
│   │   │   │   │   ├── `audit2allow` to generate policies.  
│   │   ├── **Client-side checks**:  
│   │   │   ├── Test from another client.  
│   │   │   ├── Check client firewall (`ufw status`, `iptables -L`).  
│   ├── **Hostname not pingable, IP is pingable**  
│   │   ├── **DNS Resolution**:  
│   │   │   ├── Check `/etc/hosts` for static overrides.  
│   │   │   ├── Verify `/etc/resolv.conf` (nameserver entries).  
│   │   │   ├── Check `/etc/nsswitch.conf` (order: `files dns`).  
│   │   │   ├── Test DNS with `dig <hostname> +short` or `nslookup <hostname>`.  
│   │   │   ├── For NetworkManager-managed DNS:  
│   │   │   │   ├── `nmcli dev show <interface>` (check DNS settings).  
│   │   │   │   ├── Edit `/etc/sysconfig/network-scripts/ifcfg-eth0` (RHEL).  
│   ├── **Hostname/IP both not pingable**  
│   │   ├── **Network-wide check**:  
│   │   │   ├── Ping gateway (`ip route show default`).  
│   │   │   ├── Ping another server on the same subnet.  
│   │   │   │   ├── If others are unreachable: Network outage (contact network team).  
│   │   │   │   ├── If others work: Isolate to the target server.  
│   │   ├── **Physical access via virtual console**:  
│   │   │   ├── Check uptime (`uptime`) to confirm reboots.  
│   │   │   ├── Verify NIC status (`ip link show`).  
│   │   │   │   ├── Bring interface up: `ip link set dev eth0 up`.  
│   │   │   ├── Check IP assignment (`ip addr show eth0`).  
│   │   │   │   ├── If missing:  
│   │   │   │   │   ├── Static IP: Verify `/etc/netplan/*.yaml` (Ubuntu) or `/etc/sysconfig/network-scripts/ifcfg-eth0` (RHEL).  
│   │   │   │   │   ├── DHCP: Restart `NetworkManager` or `systemctl restart network`.  
│   │   │   ├── Check ARP table: `arp -n`.  
│   │   │   ├── Test physical connectivity:  
│   │   │   │   ├── `ethtool eth0` (check link status).  
│   │   │   │   ├── NIC LEDs (physical link activity).  
```

---

### **2. Cannot Connect to a Website/Application**
```plaintext
├── **Service/Port Check**:  
│   ├── If server is reachable:  
│   │   ├── Test port connectivity: `telnet <IP> 80` or `curl -Iv http://<IP>`.  
│   │   ├── If port is closed:  
│   │   │   ├── Check service status: `systemctl status httpd` (Apache) or `systemctl status nginx`.  
│   │   │   ├── Check listening ports: `ss -tuln | grep :80`.  
│   │   │   ├── Verify firewall rules (`firewall-cmd --list-services`).  
│   │   │   ├── Check SELinux: `semanage port -l | grep http_port_t`.  
│   │   │   ├── Review logs: `journalctl -u httpd` or `/var/log/nginx/error.log`.  
│   │   ├── If port is open but no response:  
│   │   │   ├── Check application configuration (e.g., `nginx -t`).  
│   │   │   ├── Test backend services (e.g., database connectivity).  
```

---

### **3. Cannot SSH as Root/User**
```plaintext
├── **SSH Service Check**:  
│   ├── If server is reachable:  
│   │   ├── Verify SSH daemon status: `systemctl status sshd`.  
│   │   ├── Check SSH port: `ss -tln | grep :22`.  
│   │   ├── Test local SSH: `ssh localhost` (from the server itself).  
│   │   ├── Check `sshd_config`:  
│   │   │   ├── `PermitRootLogin yes` (for root access).  
│   │   │   ├── `AllowUsers <username>` (whitelist).  
│   │   ├── Verify user shell: `grep <user> /etc/passwd` (e.g., `/sbin/nologin`).  
│   │   ├── Check SSH logs: `journalctl -u sshd | grep "Failed password"`.  
│   │   ├── Test key authentication:  
│   │   │   ├── Validate `~/.ssh/authorized_keys` permissions (must be `600`).  
```

---

### **4. Disk Space Full/Extend Disk**
```plaintext
├── **Diagnose Space Usage**:  
│   ├── `df -Th` → Identify full filesystem.  
│   ├── `du -shx /* | sort -h` → Find large directories.  
│   ├── Check for deleted-but-open files: `lsof +L1`.  
│   ├── Clear space:  
│   │   ├── Delete old logs: `journalctl --vacuum-size=100M`.  
│   │   ├── Remove unused kernels (Debian: `apt autoremove`; RHEL: `package-cleanup --oldkernels`).  
├── **Extend Disk (LVM)**:  
│   ├── Add disk → `lsblk` to confirm (`/dev/sdb`).  
│   ├── Create PV: `pvcreate /dev/sdb`.  
│   ├── Extend VG: `vgextend vg_name /dev/sdb`.  
│   ├── Extend LV: `lvextend -l +100%FREE /dev/vg_name/lv_name`.  
│   ├── Resize filesystem:  
│   │   ├── Ext4: `resize2fs /dev/vg_name/lv_name`.  
│   │   ├── XFS: `xfs_growfs /mountpoint`.  
```

---

### **5. Filesystem Corruption**
```plaintext
├── **Repair Steps**:  
│   ├── Unmount filesystem: `umount /dev/sda1`.  
│   ├── Run `fsck`: `fsck -y /dev/sda1`.  
│   ├── If root FS is corrupted:  
│   │   ├── Reboot into rescue mode (e.g., GRUB → `init=/bin/bash`).  
│   │   ├── Remount as read-write: `mount -o remount,rw /`.  
│   │   ├── Run `fsck`.  
│   ├── Check disk health: `smartctl -a /dev/sda`.  
```

---

### **6. Fstab File Missing/Corrupt**
```plaintext
├── **Recovery**:  
│   ├── Reboot into rescue mode.  
│   ├── Mount root FS: `mount /dev/sda1 /mnt/sysimage`.  
│   ├── Edit fstab: `vi /mnt/sysimage/etc/fstab`.  
│   ├── Regenerate UUIDs: `blkid` → Update entries.  
│   ├── Rebuild initramfs: `dracut -fv /boot/initramfs-$(uname -r).img $(uname -r)`.  
```

---

### **7. Can’t `cd` to Directory**
```plaintext
├── **Permission Checks**:  
│   ├── Verify directory exists: `ls -ld /path`.  
│   ├── Check execute permission on parent directories:  
│   │   ├── `ls -l /parent/path` (e.g., `drwxr-xr-x`).  
│   ├── Check SELinux context: `ls -Z /path`.  
│   ├── Hidden directory: `ls -a`.  
```

---

### **8. Can’t Open/Run File**
```plaintext
├── **Troubleshooting**:  
│   ├── Check permissions: `ls -l /path/file` (e.g., `-rwxr-xr-x`).  
│   ├── Verify shebang line: `head -1 /path/script` (e.g., `#!/bin/bash`).  
│   ├── Check file type: `file /path/file`.  
│   ├── Test with absolute path: `/path/to/script.sh`.  
```

---

### **9. Can’t Create Links**
```plaintext
├── **Common Fixes**:  
│   ├── Use absolute paths: `ln -s /source/abs/path /dest/path`.  
│   ├── Check write permissions on the destination directory.  
│   ├── Verify source file existence: `ls -l /source/path`.  
```

---

### **10. Running Out of Memory**
```plaintext
├── **Diagnose**:  
│   ├── `free -h` → Check `available` memory.  
│   ├── `top` → Sort by `%MEM` (Shift+M).  
│   ├── Check OOM killer logs: `dmesg | grep -i oom`.  
├── **Add Swap**:  
│   ├── `fallocate -l 2G /swapfile`.  
│   ├── `chmod 600 /swapfile`.  
│   ├── `mkswap /swapfile && swapon /swapfile`.  
│   ├── Add to `/etc/fstab`: `/swapfile swap swap defaults 0 0`.  
```

---

### **11. Can’t Run Certain Commands**
```plaintext
├── **Troubleshooting**:  
│   ├── Check `$PATH`: `echo $PATH` → Ensure command’s directory is included.  
│   ├── Verify command exists: `which <command>`.  
│   ├── Check permissions: `ls -l $(which <command>)`.  
│   ├── Test with full path: `/usr/sbin/command`.  
│   ├── Reinstall package: `yum install <package>` or `apt install <package>`.  
```

---

### **12. Unexpected Reboots/Process Restarts**
```plaintext
├── **Diagnose**:  
│   ├── Check `last reboot` → Verify reboot times.  
│   ├── Review kernel logs: `dmesg | grep -i "error\|panic"`.  
│   ├── Check hardware logs (IPMI/iDRAC).  
│   ├── Monitor system resources: `sar -u -r 1 10`.  
```

---

### **13. Unable to Get IP Address**
```plaintext
├── **DHCP Issues**:  
│   ├── Check DHCP client: `journalctl -u NetworkManager | grep DHCP`.  
│   ├── Renew lease: `dhclient -v eth0`.  
├── **Static IP Issues**:  
│   ├── Verify config in `/etc/netplan/*.yaml` (Ubuntu) or `/etc/sysconfig/network-scripts/ifcfg-eth0` (RHEL).  
```

---

### **14. IP Assigned but Not Reachable**
```plaintext
├── **Checks**:  
│   ├── Test ARP resolution: `arping -I eth0 <IP>`.  
│   ├── Check subnet mask: `ip addr show eth0`.  
│   ├── Verify routing: `ip route get <destination>`.  
│   ├── Check for IP conflicts: `arp-scan --localnet`.  
```

---

### **15. Backup/Restore File Permissions**
```plaintext
├── **ACL Backup**:  
│   ├── Backup: `getfacl -R /path > permissions.acl`.  
│   ├── Restore: `setfacl --restore=permissions.acl`.  
```

---

### **16. HTTP 403 Forbidden in Yum**
```plaintext
├── **Fix**:  
│   ├── Check proxy in `/etc/yum.conf`:  
│   │   ├── `proxy=http://proxy:port`.  
│   ├── Verify repo URLs: `cat /etc/yum.repos.d/*.repo`.  
│   ├── Clear cache: `yum clean all`.  
│   ├── Temporarily disable SELinux: `setenforce 0`.  
```

---

### **17. Disk Partitioning Tips**
```plaintext
├── **Best Practices**:  
│   ├── Use LVM for flexibility.  
│   ├── Align partitions to 1MB boundaries (`-a optimal` in `parted`).  
│   ├── Prefer XFS for large files, Ext4 for general use.  
│   ├── Backup partition table: `sfdisk -d /dev/sda > sda-partition-table.bak`.  
```

---

### **18. General Troubleshooting Tips**
- **Logs**: Always check `/var/log/messages`, `journalctl`, and application-specific logs.  
- **Documentation**: Use `man`, `tldr`, or `--help` for command syntax.  
- **Simplicity**: Test changes in a staging environment first.  

