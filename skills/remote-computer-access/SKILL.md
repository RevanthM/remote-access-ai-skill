# Remote Computer Access Skill

Use this skill when the user wants to discover, inventory, configure, connect to, or troubleshoot computers over SSH or RDP.

## Core safety rules

1. Never print, request, commit, upload, or paste private SSH keys into chat, logs, source control, tickets, or documentation.
2. Public keys (`*.pub`) are safe to distribute to authorized target machines; private keys are not.
3. Never commit passwords, API tokens, recovery codes, RDP credentials, or credential-manager exports.
4. Prefer dedicated Ed25519 SSH keys and clear key names describing source and target.
5. Before changing networking or firewall settings, first verify the current state.
6. Prefer the smallest change needed to make the requested connection work.
7. Do not expose SSH or RDP directly to the public internet when a private network such as Tailscale/VPN is available.
8. Confirm the target machine and username before modifying `authorized_keys`.
9. When automating, use least privilege and avoid disabling authentication protections merely to make a connection work.

## Mental model

- SSH = remote command-line control.
- RDP = remote graphical desktop control, mainly for Windows.
- SSH config = address book / aliases.
- SSH private key = secret proof of identity; stays on the source/controller.
- SSH public key = permission record placed on the target.
- `authorized_keys` = list of public keys allowed to log into an account.

## Standard workflow

1. Identify the source/controller machine.
2. Identify the target machine.
3. Inventory the target.
4. Verify network reachability.
5. Verify SSH or RDP server availability on the target.
6. Verify the source has the required client.
7. Create a dedicated key pair if needed.
8. Install only the public key on the target.
9. Test an explicit connection using host/IP, username, and key.
10. Add a friendly alias after the explicit connection works.
11. Re-test using the alias.
12. Document the working connection without secrets.

## Machine inventory

Collect at minimum:

- Hostname
- Reachable IP address or DNS/Tailscale name
- Operating system and version
- Username used for remote access
- CPU
- System RAM
- GPU model
- GPU VRAM
- SSH/RDP status

### Linux inventory

```bash
hostname
hostname -I
cat /etc/os-release
lscpu | grep 'Model name'
free -h
nvidia-smi
```

If `nvidia-smi` is unavailable, determine whether the machine has a non-NVIDIA GPU using appropriate system tools before concluding there is no GPU.

### macOS inventory

```bash
hostname
whoami
ifconfig | grep 'inet '
system_profiler SPHardwareDataType
system_profiler SPDisplaysDataType
sw_vers
```

### Windows inventory — PowerShell

```powershell
hostname
whoami
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion, OsBuildNumber
Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer, Model, TotalPhysicalMemory
Get-CimInstance Win32_Processor | Select-Object Name
Get-NetIPAddress -AddressFamily IPv4
nvidia-smi
```

## SSH client checks

Windows, macOS, and Linux:

```text
ssh -V
```

## SSH server setup

### Ubuntu / Debian

Check first:

```bash
systemctl status ssh
```

If OpenSSH Server is missing and installation is authorized:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

### macOS

Use System Settings → General → Sharing → Remote Login.

Then verify:

```bash
sudo systemsetup -getremotelogin
```

### Windows

Run PowerShell as Administrator.

Check:

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
```

Install if missing:

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

Start and enable:

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
Get-Service sshd
```

## SSH key creation

Create a dedicated Ed25519 key on the source machine.

### macOS / Linux

```bash
ssh-keygen -t ed25519 -f ~/.ssh/<descriptive_name> -C '<source>-to-<target>'
```

### Windows PowerShell

```powershell
ssh-keygen -t ed25519 -f $HOME\.ssh\<descriptive_name> -C '<source>-to-<target>'
```

The file without `.pub` is private. Never share or commit it.

Display only the public key:

```bash
cat ~/.ssh/<descriptive_name>.pub
```

or on Windows:

```powershell
Get-Content $HOME\.ssh\<descriptive_name>.pub
```

## Installing a public key on Linux/macOS target

On the target account:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
printf '%s\n' '<PUBLIC_KEY_LINE>' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

Avoid adding the same key repeatedly. Check first:

```bash
cat ~/.ssh/authorized_keys
```

## Installing a public key on Windows target

For a normal non-administrator Windows account, OpenSSH commonly uses:

```text
C:\Users\<username>\.ssh\authorized_keys
```

For an administrator account, Windows OpenSSH commonly uses:

```text
C:\ProgramData\ssh\administrators_authorized_keys
```

Create/edit the appropriate file as Administrator and apply restrictive ACLs. Typical administrator-key ACL command:

```powershell
icacls 'C:\ProgramData\ssh\administrators_authorized_keys' /inheritance:r /grant 'Administrators:F' /grant 'SYSTEM:F'
Restart-Service sshd
```

If access is denied, verify the editor/PowerShell process is elevated before changing ACLs.

## Testing SSH

Always test explicitly before creating aliases.

```bash
ssh -i ~/.ssh/<private_key> <username>@<host>
```

Windows PowerShell example:

```powershell
ssh -i $HOME\.ssh\<private_key> <username>@<host>
```

Useful verification after login:

```bash
hostname
whoami
```

Exit:

```bash
exit
```

## SSH aliases

After explicit access works, simplify with `~/.ssh/config`.

Example:

```text
Host ubuntu
    HostName 192.168.1.20
    User exampleuser
    IdentityFile ~/.ssh/example_key
    IdentitiesOnly yes

Host macmini
    HostName 192.168.1.30
    User exampleuser
    IdentityFile ~/.ssh/example_key
    IdentitiesOnly yes
```

Then connect with:

```bash
ssh ubuntu
ssh macmini
```

Do not put actual private-key contents in the config file.

## RDP workflow

Use RDP when graphical desktop control of a Windows target is actually required.

1. Confirm the target Windows edition supports acting as an RDP host.
2. Confirm Remote Desktop is enabled on the intended target.
3. Confirm the intended account is allowed to sign in remotely.
4. Verify the target firewall permits Remote Desktop on the intended private network.
5. Determine the target's reachable private address or private-network hostname.
6. From Windows, launch `mstsc` and connect to that target.
7. From macOS/Linux, use an authorized RDP client.
8. Prefer VPN/Tailscale/private networking for remote-site access rather than forwarding TCP 3389 from the public internet.
9. Never store RDP passwords in the repository or skill.

SSH and RDP are complementary: SSH is preferred for automation and command execution; RDP is for tasks that genuinely require the GUI.

## Troubleshooting SSH

### `Permission denied (publickey)`

Check:

- Correct username?
- Correct private key selected?
- Matching public key present in target `authorized_keys`?
- Correct permissions/ACLs?
- Correct target machine?

Use verbose mode when necessary:

```bash
ssh -vvv -i ~/.ssh/<key> <user>@<host>
```

### `Connection refused`

Usually means host is reachable but SSH server is not listening. Verify SSH service and port.

### Timeout / no route

Usually networking, routing, VPN, firewall, or incorrect IP. Verify target IP and reachability before changing SSH configuration.

### Host key warning

On first connection, verify the target is the intended machine before accepting the fingerprint.

If a machine was rebuilt and the host key genuinely changed, remove only the stale host entry after verifying the rebuild.

## Stable networking

Local addresses such as `192.168.x.x` may change and usually work only on the local network.

For reliable private access across locations, prefer a VPN/private overlay such as Tailscale rather than exposing TCP 22 or TCP 3389 directly to the public internet.

## Completion criteria

Do not call the setup complete merely because configuration commands succeeded. A connection is complete only after:

1. The target is reachable.
2. Authentication succeeds.
3. `hostname`/`whoami` confirm the expected target and account.
4. The user can disconnect cleanly.
5. Any requested alias is tested.
6. Documentation contains no private keys, passwords, or other secrets.
