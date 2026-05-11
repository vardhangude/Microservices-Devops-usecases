# Resolving `/tmp` Disk Space and Jenkins Temp Space Issues on Linux/EC2

<img width="400" height="253" alt="UC-2" src="https://github.com/user-attachments/assets/bc162b77-32a4-4a10-afe4-60715562eae8" />

## Problem

Monitoring alert:

```text
Disk space is below threshold of 1.00 GiB.
Only 475.95 MiB out of 480.67 MiB left on /tmp
```

Common symptoms:
- Jenkins jobs failing
- Build interruptions
- `/tmp` running out of space
- No swap memory available
- System slowdown or instability

---

# Step 1 — Clean Package Cache

For RHEL/CentOS/Amazon Linux:

```bash
sudo yum clean all
```

---

# Step 2 — Restart Jenkins

```bash
sudo systemctl restart jenkins
```

---

# Step 3 — Create Swap Memory

## Create 2GB swap file

```bash
sudo fallocate -l 2G /swapfile
```

---

## Verify Disk Usage

```bash
df -h
```

---

## Set Proper Permissions

```bash
sudo chmod 600 /swapfile
```

---

## Create Swap Area

```bash
sudo mkswap /swapfile
```

---

## Enable Swap

```bash
sudo swapon /swapfile
```

---

## Verify Swap Memory

```bash
free -h
```

Expected output should show swap memory enabled.

---

# Step 4 — Make Swap Permanent

Edit `/etc/fstab`

```bash
sudo nano /etc/fstab
```

Add the following line at the bottom:

```text
/swapfile none swap sw 0 0
```

Save and exit.

---

# Step 5 — Restart Jenkins Again

```bash
sudo systemctl restart jenkins
```

---

# Step 6 — Verify `/tmp` Usage

```bash
df -h /tmp
```

---

# Step 7 — Verify Memory

```bash
free -h
```

---

# Step 8 — Check Jenkins Status

```bash
sudo systemctl status jenkins
```

---

# Step 9 — Check `/tmp` Mount Type

```bash
mount | grep /tmp
```

If `/tmp` is mounted as `tmpfs`, resize it.

---

# Step 10 — Increase `/tmp` Size Temporarily

```bash
sudo mount -o remount,size=2G /tmp
```

---

## Verify `/tmp` Size

```bash
df -h /tmp
```

---

# Step 11 — Make `/tmp` Resize Permanent

Edit `/etc/fstab`

```bash
sudo nano /etc/fstab
```

Add or update the `/tmp` entry:

```text
tmpfs /tmp tmpfs defaults,size=2G 0 0
```

Save and exit.

---

# Step 12 — Remount `/tmp`

```bash
sudo mount -o remount /tmp
```

---

# Step 13 — Restart Jenkins

```bash
sudo systemctl restart jenkins
```

---

# Verification Commands

## Check Disk Usage

```bash
df -h
```

---

## Check `/tmp`

```bash
df -h /tmp
```

---

## Check Swap

```bash
free -h
```

---

## Check Jenkins

```bash
sudo systemctl status jenkins
```

---

# Expected Result

After applying the above steps:

- `/tmp` free space increases
- Jenkins builds become stable
- Swap memory is enabled
- System performance improves
- Alerts related to temp space disappear

---

# Notes

- Swap memory helps when RAM usage is high.
- Increasing `/tmp` size prevents Jenkins build failures.
- Changes in `/etc/fstab` persist after reboot.
- Recommended minimum `/tmp` size for Jenkins servers: **2GB**

---

# Useful Troubleshooting Commands

## Find Large Files in `/tmp`

```bash
du -ah /tmp | sort -rh | head -20
```

---

## Clean `/tmp`

```bash
sudo rm -rf /tmp/*
```

---

## Check Large Jenkins Directories

```bash
sudo du -sh /var/lib/jenkins/*
```

---


