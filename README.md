# ssh-remote-mount-workflow

A documented cross-platform workflow for SSH remote access, file transfer, and remote file-system mounting between Linux and Windows.

---

## Project Goal

This repository documents a practical workflow for:

- preparing a **target** machine for SSH access
- preparing a **client** machine to connect to that target
- manually starting and stopping the SSH service for better control
- transferring files over SSH
- mounting a remote machine as a file-system location
- using simple helper scripts instead of rewriting long commands every time

This workflow is designed for **on-demand access**, not for leaving services permanently exposed.

---

## Terminology

### Target
The machine you want to access remotely.

Examples:
- a Windows desktop you want to reach from Linux
- a Linux box you want to reach from Windows
- a remote machine you want to mount or use for file transfer

### Client
The machine you are using to connect to the target.

Examples:
- a Linux laptop connecting to a Windows desktop
- a Windows laptop connecting to a Linux target
- another workstation connecting to a remote target over SSH

---

## High-Level Workflow

1. prepare the **target**
2. install and verify the required SSH components on the target
3. configure the target so the SSH service can be started manually
4. prepare the **client**
5. install the client-side tools needed for SSH / file transfer / mounting
6. start the SSH service on the target
7. connect from the client
8. transfer files or mount the remote file-system
9. unmount / disconnect when done
10. stop the SSH service on the target

---

## Network Prerequisites

Before any of this works, the following conditions must be true:

- the target must be reachable from the client
- if the access is over the internet:
  - the target side must have a **public IP** or reachable public endpoint
  - router / NAT / ISP forwarding must be configured correctly
  - the forwarding must point to the **correct internal IP** of the target
- the target should ideally have:
  - a stable internal IP
  - or a DHCP reservation
- firewall rules must allow SSH traffic
- the SSH service must be running on the target when the client tries to connect

---

## Security Model

This repository uses a **manual service control** model.

That means:

- the SSH server is installed on the target
- the SSH service does **not** have to run permanently
- the service is started only when needed
- the service is stopped when the session is finished

This keeps the setup cleaner and reduces unnecessary exposure.

---

# SECTION 1 — TARGET SETUP

---

## A. Windows as Target

This section describes how to prepare a Windows target machine.

### What must be installed on the Windows target

The Windows target needs:

- **OpenSSH Server**

### What must be run as Administrator on the Windows target

The following must be done in **PowerShell as Administrator**:

- installing OpenSSH Server
- starting the `sshd` service
- stopping the `sshd` service
- changing service startup mode
- checking firewall rules
- modifying firewall rules if needed

### 1. Check whether OpenSSH Server is installed

Run in **PowerShell as Administrator**:

~~~powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH*'
~~~

This checks whether the OpenSSH server component is already available.

### 2. Install OpenSSH Server if needed

Run in **PowerShell as Administrator**:

~~~powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
~~~

This installs the SSH server component on Windows.

### 3. Start the SSH service manually

Run in **PowerShell as Administrator**:

~~~powershell
Start-Service sshd
~~~

This starts the SSH server service.

### 4. Verify that the service is running

Run in **PowerShell as Administrator**:

~~~powershell
Get-Service sshd
~~~

Expected idea:
- the service should show as **Running**

### 5. Verify that SSH is listening

Run in **PowerShell as Administrator**:

~~~powershell
netstat -ano | findstr :22
~~~

Expected idea:
- SSH should be listening on port 22

### 6. Set the startup mode to Manual

Run in **PowerShell as Administrator**:

~~~powershell
Set-Service sshd -StartupType Manual
~~~

This prevents the SSH service from automatically starting at boot.

### 7. Confirm startup mode

Run in **PowerShell as Administrator**:

~~~powershell
Get-Service sshd | Select StartType
~~~

Expected idea:
- startup mode should show as **Manual**

This means:

- SSH will **not** start automatically after reboot
- it must be started manually when needed

### 8. Stop the SSH service when finished

Run in **PowerShell as Administrator**:

~~~powershell
Stop-Service sshd
~~~

This stops the SSH service.

### 9. Verify that the service is stopped

Run in **PowerShell as Administrator**:

~~~powershell
Get-Service sshd
~~~

Expected idea:
- the service should show as **Stopped**

### 10. Check firewall rules

Run in **PowerShell as Administrator**:

~~~powershell
Get-NetFirewallRule -Name *ssh*
~~~

This checks whether an SSH-related inbound firewall rule exists.

### 11. If needed, allow the SSH rule on all profiles

Run in **PowerShell as Administrator**:

~~~powershell
Set-NetFirewallRule -Name OpenSSH-Server-In-TCP -Profile Any
~~~

This allows the SSH rule for all network profiles.

### 12. Optional local self-test on the target

Run on the Windows target:

~~~powershell
ssh localhost
~~~

This is a quick local test to confirm the SSH server is responding.

### Suggested screenshots for Windows target setup

- `[PLACEHOLDER: screenshot of OpenSSH capability check]`
- `[PLACEHOLDER: screenshot of successful OpenSSH installation]`
- `[PLACEHOLDER: screenshot of Get-Service sshd showing Running]`
- `[PLACEHOLDER: screenshot of StartType showing Manual]`
- `[PLACEHOLDER: screenshot of firewall rule output]`
- `[PLACEHOLDER: screenshot of netstat showing port 22 listening]`

---

## B. Linux as Target

This section describes how to prepare a Linux target machine.

### What must be installed on the Linux target

The Linux target needs:

- **OpenSSH Server**

Typical package name:
- `openssh-server`

### What may require sudo / root privileges on the Linux target

The following usually require elevated privileges:

- installing `openssh-server`
- starting or stopping `ssh`
- enabling or disabling the service
- checking service status
- checking listening ports
- adjusting firewall rules if applicable

### 1. Install OpenSSH Server

On Debian / Ubuntu-based systems:

~~~bash
sudo apt update
sudo apt install openssh-server
~~~

### 2. Start the SSH service

On many systems the service name is `ssh`:

~~~bash
sudo systemctl start ssh
~~~

### 3. Verify that the service is running

~~~bash
sudo systemctl status ssh
~~~

Expected idea:
- the service should show as active / running

### 4. Check whether SSH is listening

~~~bash
ss -tulpn | grep :22
~~~

Expected idea:
- SSH should be listening on port 22

### 5. Configure the service for manual use if desired

If you want manual control instead of automatic boot start:

~~~bash
sudo systemctl disable ssh
~~~

This prevents automatic startup on boot.

### 6. Stop the SSH service when finished

~~~bash
sudo systemctl stop ssh
~~~

### 7. Verify service state again

~~~bash
sudo systemctl status ssh
~~~

Expected idea:
- the service should show as inactive / stopped

### 8. Optional local self-test on the target

~~~bash
ssh localhost
~~~

This tests whether the local SSH server responds correctly.

### Suggested screenshots for Linux target setup

- `[PLACEHOLDER: screenshot of openssh-server installation]`
- `[PLACEHOLDER: screenshot of systemctl start ssh]`
- `[PLACEHOLDER: screenshot of systemctl status ssh showing active]`
- `[PLACEHOLDER: screenshot of port 22 listening]`
- `[PLACEHOLDER: screenshot of service stopped state]`

---

# SECTION 2 — CLIENT SETUP

---

## A. Linux as Client

This section describes how to prepare a Linux client machine.

### What may need to be installed on the Linux client

Depending on what you want to do, the Linux client may need:

- **OpenSSH client** (often already installed)
- **sshfs** for file-system mounting

### What may require sudo on the Linux client

- installing `sshfs`

### 1. Verify SSH client availability

~~~bash
ssh -V
~~~

This checks whether the SSH client is available.

### 2. Install sshfs if remote mounting is needed

On Debian / Ubuntu-based systems:

~~~bash
sudo apt update
sudo apt install sshfs
~~~

### 3. Verify sshfs installation

~~~bash
sshfs --version
~~~

### 4. Create a mount point directory

~~~bash
mkdir ~/LOCAL_MOUNT_DIR
~~~

This creates a local directory where the remote target can be mounted.

Replace `LOCAL_MOUNT_DIR` with any name you want.

### Suggested screenshots for Linux client setup

- `[PLACEHOLDER: screenshot of ssh client version check]`
- `[PLACEHOLDER: screenshot of sshfs installation]`
- `[PLACEHOLDER: screenshot of sshfs version output]`
- `[PLACEHOLDER: screenshot of mount point directory creation]`

---

## B. Windows as Client

This section describes how to prepare a Windows client machine.

### What the Windows client may need

If the goal is only:
- SSH terminal access

then Windows 11 often already includes an SSH client.

If the goal is:
- mounting the remote target as a drive

then the Windows client needs:

- **WinFsp**
- **SSHFS-Win**

### What may require Administrator privileges on the Windows client

- installing WinFsp
- installing SSHFS-Win
- running installation commands via PowerShell if needed

### 1. Install SSHFS-Win and dependencies

If needed, install SSHFS-Win with WinGet:

~~~powershell
winget install SSHFS-Win.SSHFS-Win
~~~

This also installs the required dependency stack when applicable.

### 2. Verify that the installation completed

After installation, the Windows client can use SSHFS-Win paths through drive mapping commands.

### Suggested screenshots for Windows client setup

- `[PLACEHOLDER: screenshot of winget installation]`
- `[PLACEHOLDER: screenshot showing successful installation of WinFsp and SSHFS-Win]`

---

# SECTION 3 — CONNECTING CLIENTS TO THE TARGET

---

## A. Linux Client → Windows Target (SSH shell)

Run from the Linux client:

~~~bash
ssh USERNAME@PUBLIC_IP
~~~

### Notes
- replace `USERNAME` with the target account name
- replace `PUBLIC_IP` with the target public IP or reachable hostname
- if this works without specifying a port, then the connection is already reachable on the default SSH port 22

---

## B. Linux Client → Linux Target (SSH shell)

Run from the Linux client:

~~~bash
ssh USERNAME@PUBLIC_IP
~~~

The same format applies.

---

## C. Linux Client → Remote Target (file transfer with SCP)

### Generic download pattern

~~~bash
scp USERNAME@PUBLIC_IP:"REMOTE_PATH_TO_FILE" .
~~~

### Notes
- `USERNAME` = remote account name
- `PUBLIC_IP` = reachable target address
- `REMOTE_PATH_TO_FILE` = full path to the file on the target
- `.` means: save it in the current local directory

### Generic download to a chosen local directory

~~~bash
scp USERNAME@PUBLIC_IP:"REMOTE_PATH_TO_FILE" ~/LOCAL_DIRECTORY/
~~~

---

## D. Linux Client → Remote Target (mount with sshfs)

### Generic mount pattern for a Windows target

~~~bash
sshfs USERNAME@PUBLIC_IP:/REMOTE_TARGET_PATH ~/LOCAL_MOUNT_DIR
~~~

### Generic mount pattern for a Linux target

~~~bash
sshfs USERNAME@PUBLIC_IP:/home/USERNAME ~/LOCAL_MOUNT_DIR
~~~

### Notes
- `USERNAME` = target account name
- `PUBLIC_IP` = target address
- `/REMOTE_TARGET_PATH` = path on the target you want to mount
- `~/LOCAL_MOUNT_DIR` = local mount point on the Linux client

---

## E. Linux Client → Manual unmount

~~~bash
fusermount -u ~/LOCAL_MOUNT_DIR
~~~

This cleanly disconnects the mount.

---

## F. Windows Client → Remote Target (mount as drive)

### Generic BAT-style connection

~~~bat
@echo off
net use X: \\sshfs\USERNAME@PUBLIC_IP /persistent:no
start X:
pause
~~~

### Notes
- `USERNAME` = target account name
- `PUBLIC_IP` = target public IP or hostname
- `X:` = chosen drive letter
- this maps the remote target as a Windows drive
- this does **not** open an SSH terminal
- it creates a mounted remote drive

---

## G. Windows Client → Manual disconnect

~~~bat
@echo off
net use X: /delete
pause
~~~

This removes the mounted drive mapping.

### Suggested screenshots for connection phase

- `[PLACEHOLDER: screenshot of Linux SSH connection to target]`
- `[PLACEHOLDER: screenshot of SCP transfer example]`
- `[PLACEHOLDER: screenshot of sshfs mount on Linux]`
- `[PLACEHOLDER: screenshot of mounted remote directory visible in Linux file manager]`
- `[PLACEHOLDER: screenshot of Windows drive mapping via SSHFS-Win]`
- `[PLACEHOLDER: screenshot of mapped drive visible in This PC]`

---

# SECTION 4 — HELPER SCRIPTS

---

## A. Windows Target Helper Scripts

### Windows target — start SSH service

Create:

`ssh-start.bat`

~~~bat
@echo off
echo Starting SSH service...
powershell Start-Service sshd
echo.
echo Current status:
powershell Get-Service sshd
pause
~~~

### Windows target — stop SSH service

Create:

`ssh-stop.bat`

~~~bat
@echo off
echo Current status:
powershell Get-Service sshd
echo.
echo Stopping SSH service...
powershell Stop-Service sshd
echo.
echo New status:
powershell Get-Service sshd
pause
~~~

These should be run with Administrator privileges.

---

## B. Linux Target Helper Scripts

### Linux target — start SSH service

Create:

`ssh-start.sh`

~~~bash
#!/bin/bash
sudo systemctl start ssh
sudo systemctl status ssh
~~~

### Linux target — stop SSH service

Create:

`ssh-stop.sh`

~~~bash
#!/bin/bash
sudo systemctl status ssh
sudo systemctl stop ssh
echo
echo "New status:"
sudo systemctl status ssh
~~~

Then make them executable:

~~~bash
chmod +x ssh-start.sh
chmod +x ssh-stop.sh
~~~

---

## C. Linux Client Helper Scripts

### Linux client — mount remote target

Create:

`mount-remote.sh`

~~~bash
#!/bin/bash
sshfs USERNAME@PUBLIC_IP:/REMOTE_TARGET_PATH ~/LOCAL_MOUNT_DIR
~~~

### Linux client — unmount remote target

Create:

`unmount-remote.sh`

~~~bash
#!/bin/bash
fusermount -u ~/LOCAL_MOUNT_DIR
~~~

Then make them executable:

~~~bash
chmod +x mount-remote.sh
chmod +x unmount-remote.sh
~~~

---

## D. Windows Client Helper Scripts

### Windows client — mount remote target as drive

Create:

`mount-remote.bat`

~~~bat
@echo off
net use X: \\sshfs\USERNAME@PUBLIC_IP /persistent:no
start X:
pause
~~~

### Windows client — unmount remote target

Create:

`unmount-remote.bat`

~~~bat
@echo off
net use X: /delete
pause
~~~

---

# SECTION 5 — WHY MANUAL START / STOP IS RECOMMENDED

Using SSH on demand is cleaner than leaving it active permanently.

Reasons:

- reduces unnecessary exposure
- gives better operational control
- avoids leaving services open when not needed
- fits home lab and controlled workflow use cases
- makes troubleshooting simpler

Recommended flow:

1. start SSH server on target
2. connect from client
3. transfer or mount
4. unmount / disconnect
5. stop SSH server on target

---

# SECTION 6 — REPOSITORY PLACEHOLDERS

Use this repository for:

- sanitized setup notes
- screenshots
- command examples with placeholders
- helper script examples
- troubleshooting notes

Do **not** commit:

- real usernames
- real public IPs
- real passwords
- real internal network details
- real router screenshots with sensitive data
- real credentials of any kind

---

## Suggested Screenshot Blocks

### Target Setup — Windows
- `[PLACEHOLDER: Windows OpenSSH installation]`
- `[PLACEHOLDER: Windows service running]`
- `[PLACEHOLDER: Windows firewall rule]`
- `[PLACEHOLDER: Windows service startup mode set to Manual]`

### Target Setup — Linux
- `[PLACEHOLDER: Linux OpenSSH installation]`
- `[PLACEHOLDER: Linux systemctl start ssh]`
- `[PLACEHOLDER: Linux service active state]`

### Client Setup — Linux
- `[PLACEHOLDER: Linux sshfs installation]`
- `[PLACEHOLDER: Linux mount directory creation]`

### Client Setup — Windows
- `[PLACEHOLDER: SSHFS-Win installation]`
- `[PLACEHOLDER: WinFsp installation]`

### Connection Examples
- `[PLACEHOLDER: Linux SSH shell connection]`
- `[PLACEHOLDER: Linux SCP file transfer]`
- `[PLACEHOLDER: Linux sshfs mount visible in file manager]`
- `[PLACEHOLDER: Windows mapped drive visible in This PC]`

---

# SECTION 7 — DISCLAIMER

This repository is intended for:

- home lab use
- administrative workflow practice
- SSH-based file workflow documentation
- learning and automation examples

Use only on systems you own or are explicitly authorized to manage.

Always understand:
- firewall configuration
- routing / NAT / port forwarding
- SSH service exposure
- credential handling
- the security implications of remote access

---
