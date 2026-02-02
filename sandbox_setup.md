# Local Testing Environment Setup

This guide covers setting up multiple Windows 11 VMs for local CUA agent testing using Hyper-V.

> **Note**: Windows Sandbox only allows one running instance at a time. For parallel testing, use Hyper-V VMs instead.

## Prerequisites

- Windows 11 Pro/Enterprise/Education (Hyper-V is not available on Home edition)
- 16GB+ RAM recommended (4GB per VM + host overhead)
- CPU with SLAT support (most modern CPUs)

## Step 1: Enable Hyper-V

Run in elevated PowerShell:

```powershell
Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
```

Reboot when prompted. Verify by opening "Hyper-V Manager" from Start menu.

## Step 2: Download Windows 11 ISO

1. Go to: https://www.microsoft.com/en-us/evalcenter/evaluate-windows-11-enterprise
2. Download the 64-bit ISO (90-day evaluation, free)
3. Save to `C:\ISOs\Win11Enterprise.iso`

## Step 3: Create Virtual Switch

Open Hyper-V Manager → Virtual Switch Manager → New virtual network switch:

| Setting | Value |
|---------|-------|
| Type | External |
| Name | `External-Internet` |
| External network | Select your physical NIC |

This gives VMs internet access and makes them reachable from host.

## Step 4: Create Base VM

In Hyper-V Manager → Action → New → Virtual Machine:

| Setting | Value |
|---------|-------|
| Name | `cua-base-template` |
| Generation | **2** (required for TPM/Windows 11) |
| Memory | 4096 MB (uncheck dynamic memory for consistent performance) |
| Network | `External-Internet` |
| Virtual Hard Disk | 60 GB, dynamically expanding |
| Installation | Mount the Windows 11 ISO |

After creation, configure security:
1. Right-click VM → Settings → Security
2. Enable "Enable Trusted Platform Module"
3. Secure Boot: Microsoft UEFI Certificate Authority

Start VM and install Windows 11.

## Step 5: Configure Base VM

After Windows installation:

### 5.1 Create Local Account

Create a local admin account (e.g., `cua-agent` with your password). Avoid Microsoft account for simpler automation.

### 5.2 Enable Remote Desktop

```
Settings → System → Remote Desktop → Enable
```

Allow connections from any version of Remote Desktop.

### 5.3 Set Static IP (Recommended)

```
Settings → Network & Internet → Ethernet → Edit IP settings
```

Example IPs for your VMs:
- VM1: `192.168.1.101`
- VM2: `192.168.1.102`
- VM3: `192.168.1.103`

### 5.4 Disable Windows Update (Optional)

Prevents surprise reboots during testing:

```powershell
# Run as admin
Stop-Service wuauserv
Set-Service wuauserv -StartupType Disabled
```

### 5.5 Install Python 3.12

> **Important**: Use Python 3.12, NOT 3.13+ (pywin32 compatibility issues)

Download from https://python.org and install with "Add to PATH" checked.

### 5.6 Install Driver Dependencies

```powershell
pip install pyautogui mss pillow rpaframework websockets
```

### 5.7 Clone and Setup Driver

```powershell
cd C:\
git clone <your-repo-url> granite
cd granite\driver
pip install -r requirements.txt
```

### 5.8 Configure Driver WebSocket URL

Update the driver configuration to point to your backend WebSocket endpoint.

### 5.9 Create Checkpoint

In Hyper-V Manager:
1. Right-click VM → Checkpoint
2. Name it `clean-base-with-driver`

This allows quick revert to clean state.

## Step 6: Clone VMs

### Export Base VM

1. Shut down the base VM
2. Right-click → Export
3. Save to `C:\Hyper-V\Exports\`

### Import as Copies

For each additional VM:

**Option A: GUI**
1. Action → Import Virtual Machine
2. Select the exported folder
3. Choose "Copy the virtual machine (create a new unique ID)"
4. Select new storage location

**Option B: PowerShell**
```powershell
Import-VM -Path "C:\Hyper-V\Exports\cua-base-template\Virtual Machines\*.vmcx" -Copy -GenerateNewId
```

### Configure Each Clone

For each cloned VM:
1. Rename in Hyper-V Manager: `cua-local-01`, `cua-local-02`, etc.
2. Start VM and change computer name (to avoid network conflicts)
3. Assign unique static IP
4. Update `client_machine_id` in driver config (must be unique per machine)

## Step 7: Start VMs and Register Drivers

1. Start all VMs in Hyper-V Manager
2. In each VM, run the driver application
3. Driver will auto-register with backend via WebSocket
4. Verify machines appear in MongoDB `driver_machines` collection

## Step 8: Add RDP Credentials

For each registered VM, update RDP credentials in MongoDB so the CUA agent can connect:

```python
from backend.utils.mongo.vm_keys_mongodb import update_rdp_credentials

# Get driver_machine_id from MongoDB after driver registers
update_rdp_credentials(
    driver_machine_id="<id-from-mongodb>",
    rdp_username="cua-agent",
    rdp_password="your-password",
    rdp_external_ip="192.168.1.101",  # VM's static IP
    rdp_port=3389
)
```

Or use MongoDB Compass to update the `driver_machines` collection directly:
```json
{
  "rdp_username": "cua-agent",
  "rdp_password": "your-password",
  "rdp_external_ip": "192.168.1.101",
  "rdp_port": 3389
}
```

## Quick Reference Commands

```powershell
# Start all CUA VMs
Get-VM -Name "cua-local-*" | Start-VM

# Stop all CUA VMs
Get-VM -Name "cua-local-*" | Stop-VM

# Check VM status
Get-VM -Name "cua-local-*" | Select Name, State, CPUUsage, MemoryAssigned

# Revert to clean checkpoint
Restore-VMCheckpoint -Name "clean-base-with-driver" -VMName "cua-local-01" -Confirm:$false

# Revert all VMs to checkpoint
Get-VM -Name "cua-local-*" | Restore-VMCheckpoint -Name "clean-base-with-driver" -Confirm:$false
```

## Resource Planning

| Configuration | RAM per VM | Total VMs | VM RAM Usage | Host Available |
|--------------|-----------|-----------|--------------|----------------|
| 16GB system  | 4GB       | 2         | 8GB          | ~8GB           |
| 32GB system  | 4GB       | 3         | 12GB         | ~20GB          |
| 32GB system  | 4GB       | 5         | 20GB         | ~12GB          |

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Can't RDP to VM | Check Windows Firewall allows RDP (port 3389) |
| VM not appearing in MongoDB | Verify driver is running and WebSocket URL is correct |
| RDP connection fails from backend | Verify `rdp_external_ip` matches VM's actual IP |
| VM runs slow | Disable dynamic memory, ensure host has RAM headroom |
| TPM error during Windows install | Enable TPM in VM Settings → Security |
| 90-day eval expires | Create fresh VM from ISO, re-run setup steps |
| Network conflicts | Ensure each VM has unique computer name and IP |

## Architecture Notes

The local Hyper-V setup integrates with the existing backend architecture:

1. **Driver Registration**: When you start the driver in a VM, it connects via WebSocket and calls `register_or_update_driver_machine()` automatically
2. **RDP Credentials**: Stored in MongoDB via `update_rdp_credentials()`, retrieved by CUA agent via `get_rdp_credentials()`
3. **Job Execution**: Backend dispatches jobs to VMs via the same job queue system used for cloud VMs
4. **No Backend Changes Required**: The architecture is VM-agnostic; local Hyper-V VMs work identically to cloud VMs
