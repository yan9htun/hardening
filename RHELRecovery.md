# 🛠️ RHEL Recovery Guide After Failed OSCAP/PCI-DSS Hardening

##  Scenario

After applying **oscap PCI-DSS remediation** on RHEL, the system fails to boot. Errors such as:

`Failed to start OpenSSH server daemon: Permission denied Failed to execute command: Permission denied dracut-shutdown.service: Failed at step EXEC spawning dracut-initramfs-restore`

This is usually caused by incorrect **permissions**, **SELinux contexts**, or **disabled services** applied during hardening.

* * *

##  Recovery Procedure

### 1\. Boot into Rescue Mode

1.  Attach RHEL installation ISO and reboot VM.
    
2.  At boot menu, select:
    
    `Troubleshooting → Rescue a Red Hat Enterprise Linux system`
    
3.  Choose:
    
    -   **1) Continue** (mounts system under `/mnt/sysimage`).
        
4.  Enter chroot:
    
    `chroot /mnt/sysimage`
    

* * *

### 2\. Fix File Permissions

Reset permissions on critical executables:

```bash
chmod 755 /usr/sbin/sshd 
chmod 755 /usr/bin/pmlogctl
chmod 755 /usr/bin/pmiectl
chmod 755 /usr/lib/dracut/dracut-initramfs-restore
chown root:root /usr/sbin/sshd /usr/bin/pmlogctl /usr/bin/pmiectl /usr/lib/dracut/dracut-initramfs-restore
```

* * *

### 3\. Restore SELinux Contexts

```bash 
touch /.autorelabel restorecon -Rv /usr /bin /sbin /usr/lib /usr/sbin
```
OR
```bash
setenforce 0
vi /etc/selinux/config
```
change enforicng to disabled. You disabled SELinux, so step 3 (restorecon + autorelabel) can be skipped.
* * *

### 4\. Correct Local Repo File
The ISO must be mounted to /mnt before running dnf.

Ensure the subdirectories AppStream and BaseOS exist:
```bash
mount /dev/cdrom /mnt
ls /mnt
```
Make sure /etc/yum.repos.d/local.repo looks like this:
```bash
[AppStream]
name=AppStream
baseurl=file:///mnt/AppStream
enabled=1
gpgcheck=0

[BaseOS]
name=BaseOS
baseurl=file:///mnt/BaseOS
enabled=1
gpgcheck=0

```

* * *

### 5\. Rebuild Initramfs

`dracut -f`

* * *

### 6\. Exit & Reboot

`exit reboot`

* * *

##  Optional: Restore OSCAP Backup

If permissions/SELinux restore does not work, revert oscap changes:

`ls -d /var/tmp/oscap-backup-* oscap-restore /var/tmp/oscap-backup-<latest>`

* * *

##  Best Practices

-   **Scan first, remediate later:**
    
    `oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_pci-dss \ --report /root/pci-dss-report.html \ /usr/share/xml/scap/ssg/content/ssg-rhel8-ds.xml`
    
-   Apply only **needed rules** (not full PCI-DSS) to avoid breaking services.
    
-   Test remediation in a **lab VM** before applying to production.
