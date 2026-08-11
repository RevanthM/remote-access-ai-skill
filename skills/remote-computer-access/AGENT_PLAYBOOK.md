# Agent Playbook: Autonomous SSH/RDP Setup

This playbook turns the core skill into an agentic workflow. The agent should inspect first, decide second, change third, and verify last.

## Operating policy

The agent should not assume the operating system, username, network path, SSH service state, RDP availability, key location, or firewall state. Discover these facts before modifying the machine.

The agent may automate routine configuration on machines the user owns or administers, but should avoid destructive changes and should never weaken authentication merely to get a connection working.

## Phase 1 — Identify the current machine

Determine:

- OS family: Windows, macOS, or Linux
- hostname
- current username
- local IP addresses
- whether SSH client exists
- whether SSH server exists/runs
- whether RDP host capability exists
- whether Tailscale or another private-network client exists
- existing SSH keys and config aliases

### Windows discovery

```powershell
hostname
whoami
ssh -V
Get-Service sshd -ErrorAction SilentlyContinue
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.IPAddress -notlike '127.*'}
Get-ChildItem $HOME\.ssh -Force -ErrorAction SilentlyContinue
Get-Content $HOME\.ssh\config -ErrorAction SilentlyContinue
Get-Command tailscale -ErrorAction SilentlyContinue
```

### macOS discovery

```bash
hostname
whoami
ssh -V
sudo systemsetup -getremotelogin 2>/dev/null || true
ifconfig | grep 'inet '
ls -la ~/.ssh 2>/dev/null
cat ~/.ssh/config 2>/dev/null
command -v tailscale || true
```

### Linux discovery

```bash
hostname
whoami
ssh -V
systemctl status ssh --no-pager 2>/dev/null || systemctl status sshd --no-pager 2>/dev/null || true
hostname -I
ls -la ~/.ssh 2>/dev/null
cat ~/.ssh/config 2>/dev/null
command -v tailscale || true
```

## Phase 2 — Identify the target and desired control mode

Choose SSH when the user wants:

- command execution
- file management
- software installation
- scripts
- AI workloads
- server administration
- automation

Choose RDP when the user specifically needs:

- a Windows graphical desktop
- GUI-only applications
- visual interaction with the remote Windows session

If both are useful, configure SSH first because it is easier to automate and diagnose.

## Phase 3 — Decide the network path

Preferred order:

1. Existing private overlay network hostname/IP such as Tailscale
2. Stable LAN hostname/IP when all machines are on the same trusted network
3. VPN/private routed network
4. Public exposure only when explicitly required and securely designed

Do not automatically open ports 22 or 3389 on an internet-facing router.

## Phase 4 — Build the SSH trust relationship

For controller A to connect to target B:

1. Find or create a dedicated private/public key pair on A.
2. Keep the private key on A.
3. Add A's public key to B's authorized-key store.
4. Verify permissions.
5. Test using the full explicit SSH command.
6. Confirm `hostname` and `whoami` after login.
7. Add a friendly alias to A's SSH config.
8. Re-test using the alias.

A machine being able to connect outward does not imply it accepts incoming SSH. Configure each desired target separately.

## Phase 5 — Multi-machine mesh

If the user wants any authorized machine to control every other machine, model this as multiple trust edges.

For three machines A, B, C, full bidirectional SSH requires:

- A -> B
- A -> C
- B -> A
- B -> C
- C -> A
- C -> B

Do not solve this by copying the same private key to every machine. Prefer a separate key identity per controller machine and distribute only public keys.

## Phase 6 — Friendly aliases

Use aliases only after explicit connections work.

Example:

```text
Host windows-main
    HostName windows-main.example-tailnet.ts.net
    User exampleuser
    IdentityFile ~/.ssh/controller_to_windows
    IdentitiesOnly yes

Host ubuntu-gpu
    HostName ubuntu-gpu.example-tailnet.ts.net
    User exampleuser
    IdentityFile ~/.ssh/controller_to_ubuntu
    IdentitiesOnly yes

Host macmini
    HostName macmini.example-tailnet.ts.net
    User exampleuser
    IdentityFile ~/.ssh/controller_to_macmini
    IdentitiesOnly yes
```

Use neutral examples in public repositories. Keep actual private hostnames, usernames, and network addresses in private configuration, not the reusable public skill.

## Phase 7 — RDP setup logic

Before enabling RDP on Windows, determine Windows edition and whether it supports inbound Remote Desktop hosting.

Check current state before changing anything.

Typical Windows discovery:

```powershell
Get-ComputerInfo | Select-Object WindowsProductName, WindowsVersion
Get-ItemProperty 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections
Get-NetFirewallRule -DisplayGroup 'Remote Desktop' -ErrorAction SilentlyContinue
```

If inbound RDP is supported and the user authorizes enabling it, enable the feature and the corresponding firewall rules on the intended private network profile. Do not expose RDP directly to the public internet by default.

After configuration, test from an authorized client and confirm the expected Windows machine/account.

## Phase 8 — Troubleshooting decision tree

### Timeout

Investigate network path first:

- wrong address
- target offline
- different subnet without routing
- private-network client disconnected
- firewall dropping traffic

### Connection refused

Investigate target service:

- SSH/RDP server not running
- wrong port
- service bound to another interface

### Permission denied

Investigate authentication:

- wrong username
- wrong key
- public key absent
- permissions/ACL problem
- administrator Windows key stored in wrong authorized-keys file

### Host identification warning

Verify the target identity. If the machine was rebuilt or intentionally changed, remove only the stale entry for that host after verification.

## Phase 9 — Verification report

After changes, the agent should produce a short report for every machine:

```text
Machine: <friendly name>
Hostname: <hostname>
OS: <OS/version>
User: <remote username>
Network path: <LAN/VPN/Tailscale>
SSH server: working/not configured
SSH alias: <alias or none>
RDP: working/not configured/not supported
GPU: <model>
RAM: <amount>
Verification: hostname + whoami confirmed
```

Never include private-key material or passwords in the report.

## Phase 10 — Stop conditions

Stop and ask the user before proceeding when:

- ownership/authorization of a target is unclear
- a step would expose SSH/RDP to the public internet
- a destructive firewall/network change is required
- disk encryption or credential-store secrets would need to be bypassed
- the only apparent fix is disabling authentication or host verification
- the target identity cannot be verified

Otherwise, continue autonomously through inspection, minimal configuration, connection test, alias setup, and final verification.
