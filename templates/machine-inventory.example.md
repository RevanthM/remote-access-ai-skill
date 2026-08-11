# Remote Machine Inventory Template

Keep real credentials and private keys OUT of this file.

| Friendly name | Hostname | OS | Remote user | Network address/name | CPU | RAM | GPU | VRAM | SSH | RDP |
|---|---|---|---|---|---|---|---|---|---|---|
| windows-main | EXAMPLE-WINDOWS | Windows 11 Pro | exampleuser | windows-main.example-tailnet.ts.net | Example CPU | 64 GB | NVIDIA Example | 16 GB | Working | Working |
| ubuntu-gpu | example-ubuntu | Ubuntu | exampleuser | ubuntu-gpu.example-tailnet.ts.net | Example CPU | 32 GB | NVIDIA Example | 12 GB | Working | N/A |
| macmini | Example-Mac-mini | macOS | exampleuser | macmini.example-tailnet.ts.net | Apple Silicon | 24 GB | Integrated | Unified | Working | N/A |

## Per-machine record

```text
Machine:
Hostname:
OS/version:
Remote username:
Private network hostname/IP:
LAN IP (optional):
CPU:
RAM:
GPU:
GPU VRAM:
SSH server status:
SSH alias:
RDP status:
Last verified:
Notes:
```

Do not record passwords or private-key contents here.
