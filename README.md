## REMnux + MCP + Ollama: Fully Isolated AI-Assisted Malware Analysis Lab

A fully offline malware analysis lab built on [REMnux](https://remnux.org) with AI-assisted
analysis via [OpenCode](https://opencode.ai) and the [REMnux MCP Server](https://github.com/REMnux/remnux-mcp-server),
powered by a local LLM running on [Ollama](https://ollama.com).

No data leaves your machine. No internet connection required after initial setup.

> **TL;DR** — Import the REMnux OVA into VMware, give it two network adapters
> (one NAT for setup, one host-only for isolation), install Ollama on the host,
> point OpenCode at it, disable the NAT adapter, and you have a fully offline
> AI malware analysis lab. ~30 minutes total.

---

### Features

- Fully isolated REMnux VM with **zero external connectivity**
- AI-assisted malware analysis via OpenCode + MCP (Model Context Protocol)
- Local LLM inference — no cloud API, no data exfiltration risk
- Pre-configured MCP servers:
  - `remnux` — core tools (file info, IOC extraction, tool runner)
  - `radare2` — static/dynamic binary analysis
  - `ghidra` — decompilation and function analysis
  - `procmon` — Windows Process Monitor log parsing
  - `remnux-docs` — tool documentation search
- One-command sample analysis: `Analyze sample.exe`
- Snapshot-based restore for clean re-analysis

---

### Architecture

| Component | Location | Role |
|-----------|----------|------|
| Ollama + local LLM | Host (Linux Mint) | Local LLM inference (GPU-accelerated) |
| REMnux VM | VMware Workstation 17 | Malware analysis environment |
| OpenCode | Inside REMnux | AI agent / MCP client |
| REMnux MCP Server | Inside REMnux | Exposes analysis tools to the AI |
| vmnet1 (host-only) | VMware | Isolated network between host and VM |

```
┌─────────────────────────────────────────────────────────┐
│  Linux Mint Host                                         │
│                                                          │
│  ┌──────────────────────┐       ┌────────────────────┐  │
│  │  Ollama (host)       │◄──────│  REMnux VM         │  │
│  │  local LLM           │ vmnet1│  172.16.204.100    │  │
│  │  172.16.204.1:11434  │ host- │  ens33 = UP        │  │
│  │  GPU (4+ GB VRAM)    │ only  │  ens34 = DOWN      │  │
│  └──────────────────────┘       │  OpenCode + MCP    │  │
│                                 └────────────────────┘  │
│                                                          │
│  No external network connectivity from REMnux.           │
└─────────────────────────────────────────────────────────┘
```

---

### Prerequisites

| Requirement | Notes |
|-------------|-------|
| Linux Mint (or any Debian/Ubuntu) | Host OS |
| VMware Workstation 17 Pro | Hypervisor |
| NVIDIA GPU with 4+ GB VRAM | For Ollama inference (8 GB recommended) |
| 8+ GB RAM (16 GB recommended) | Split between host and VM |
| ~25 GB disk space | REMnux OVA (~9 GB) + Ollama model (~2.5 GB) + headroom |

---

### Installation

> **Note:** Replace the example IPs below with your actual values from Step 3 before copying.

---

#### Step 1: Install VMware Workstation 17 *(host)*

Download the `.bundle` from the VMware/Broadcom site, then:

```bash
sudo sh VMware-Workstation-Full-*.bundle
```

---

#### Step 2: Start VMware Network Services *(host)*

```bash
sudo /usr/bin/vmware-networks --start
ps aux | grep vmnet
```

You should see `vmnet-natd`, `vmnet-dhcpd`, and `vmnet-netifup` processes.

Make it persist across reboots:

```bash
echo "sudo /usr/bin/vmware-networks --start" | sudo tee -a /etc/rc.local
sudo chmod +x /etc/rc.local
```

---

#### Step 3: Identify Your Network Subnets *(host)*

```bash
ip addr show vmnet8
ip addr show vmnet1
```

Note both IPs:

- **vmnet8** (NAT) — e.g. `192.168.45.1/24` → used in Step 7
- **vmnet1** (host-only) — e.g. `172.16.204.1/24` → used in Step 10

---

#### Step 4: Create an Isolated Host-Only Network *(host, GUI)*

1. **Edit → Virtual Network Editor → Change Settings**
2. **Add Network** → select **VMnet2** (or next available)
3. Set type to **Host-only**
4. **Uncheck** "Connect a host virtual adapter to this network"
5. Enable DHCP within the assigned subnet
6. **Apply**

This network will later host a Windows victim VM for dynamic analysis.

---

#### Step 5: Download and Import the REMnux OVA *(host, GUI)*

1. Download the **General OVA** from [docs.remnux.org](https://docs.remnux.org/install-distro/get-virtual-appliance)
2. Verify the SHA-256 hash
3. In VMware: **File → Open** → select the `.ova` → **Import**
4. Choose **"I copied it"** if prompted

---

#### Step 6: Configure REMnux VM Networking *(host, GUI)*

With the VM **powered off**, go to **VM → Settings → Network**:

| Adapter | Type | Purpose |
|---------|------|---------|
| Adapter 1 | **Custom → VMnet2** (isolated host-only) | Lab traffic (no internet) |
| Adapter 2 | **NAT** (vmnet8) | Internet for initial setup only |

Also set:

- RAM: **2048 MB** minimum
- CPUs: **2**
- Disable **Shared Folders** and **Drag and Drop** under Options

> Inside the VM: Adapter 1 = `ens33`, Adapter 2 = `ens34`.

---

#### Step 7: Boot and Fix NAT Networking *(REMnux)*

Power on the VM. It **auto-logs in** as `remnux` (no password prompt).

Replace `192.168.45.100` and `192.168.45.1` with your actual vmnet8 values, then copy-paste the entire block:

```bash
sudo ip addr flush dev ens34
sudo ip link set ens34 up
sudo ip addr add 192.168.45.100/24 dev ens34
sudo ip route add default via 192.168.45.1 dev ens34
echo -e "[Resolve]\nDNS=8.8.8.8" | sudo tee /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
ping -c 3 192.168.45.1
ping -c 3 8.8.8.8
getent hosts api.github.com
```

**If `ping 8.8.8.8` fails** (gateway responds but internet doesn't), on the **host**:

```bash
ip route | grep default
sudo iptables -t nat -A POSTROUTING -s 192.168.45.0/24 -o wlp2s0 -j MASQUERADE
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

> Replace `wlp2s0` with your actual uplink and `192.168.45.0/24` with your vmnet8 subnet.

---

#### Step 8: Make REMnux Networking Persist *(REMnux)*

Replace IPs with your actual values, then copy-paste the entire block:

```bash
sudo tee /etc/netplan/01-netcfg.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
    ens34:
      dhcp4: false
      addresses:
        - 192.168.45.100/24
      routes:
        - to: default
          via: 192.168.45.1
EOF
sudo chmod 600 /etc/netplan/01-netcfg.yaml
sudo netplan apply
```

---

#### Step 9: Update REMnux and Enable SSH *(REMnux)*

```bash
remnux install
sudo systemctl enable ssh
sudo systemctl start ssh
sudo reboot
ping -c 3 8.8.8.8
```

> ⚠️ Do **not** use `update-remnux` — that command is from REMnux v6 (2015) and no longer exists.

---

#### Step 10: Install Ollama on the Host *(host)*

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama pull qwen3:4b
```

Model selection by VRAM:

| VRAM | Model | Size |
|------|-------|------|
| 4 GB | `qwen3:4b` | ~2.5 GB |
| 8 GB | `qwen3:8b` | ~5 GB |
| 16 GB | `qwen3:14b` | ~9 GB |
| 24 GB+ | `qwen3:32b` | ~20 GB |

Configure Ollama to listen on the vmnet1 IP. Replace `172.16.204.1` with your actual vmnet1 IP, then copy-paste the entire block:

```bash
sudo mkdir -p /etc/systemd/system/ollama.service.d
sudo tee /etc/systemd/system/ollama.service.d/override.conf << 'EOF'
[Service]
Environment="OLLAMA_HOST=172.16.204.1:11434"
Environment="OLLAMA_CONTEXT_LENGTH=16384"
EOF
sudo systemctl daemon-reload
sudo systemctl restart ollama
ss -tlnp | grep 11434
```

Should show `172.16.204.1:11434`.

---

#### Step 11: Connect REMnux to Ollama *(REMnux)*

Replace IPs with your actual vmnet1 values, then copy-paste the entire block:

```bash
sudo ip link set ens33 up
sudo ip addr add 172.16.204.100/24 dev ens33
curl http://172.16.204.1:11434/api/tags
sudo tee /etc/netplan/01-netcfg.yaml << 'EOF'
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses:
        - 172.16.204.100/24
    ens34:
      dhcp4: false
EOF
sudo chmod 600 /etc/netplan/01-netcfg.yaml
sudo netplan apply
```

The `curl` line should return JSON listing `qwen3:4b`.

---

#### Step 12: Disable Internet (Full Isolation) *(REMnux)*

```bash
sudo ip link set ens34 down
```

REMnux now has **zero external connectivity**. The only network path is to Ollama.

---

#### Step 13: Configure OpenCode *(REMnux)*

Replace `172.16.204.1` with your actual vmnet1 IP, then copy-paste the entire block:

```bash
mkdir -p ~/.config/opencode
cat > ~/.config/opencode/opencode.json << 'EOF'
{
  "$schema": "https://opencode.ai/config.json",
  "model": "ollama/qwen3:4b",
  "provider": {
    "ollama": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "Ollama",
      "options": {
        "baseURL": "http://172.16.204.1:11434/v1"
      },
      "models": {
        "qwen3:4b": {
          "name": "Qwen3 4B (local)"
        }
      }
    }
  },
  "mcp": {
    "servers": {
      "remnux": {
        "type": "local",
        "command": ["remnux-mcp-server"]
      }
    }
  }
}
EOF
```

> **Note:** The `radare2`, `ghidra`, `procmon`, and `remnux-docs` MCP servers are pre-configured by REMnux's salt-states and will connect automatically. Only the `remnux` core server needs to be added manually. If you've already configured additional MCP servers, merge this config into your existing file rather than replacing it.

---

#### Step 14: Verify *(REMnux)*

```bash
opencode
```

Inside OpenCode, type:

```
/mcps
```

You should see:

| MCP Server | Status | Purpose |
|------------|--------|---------|
| `remnux` | ✅ connected | Core tools (file info, IOC extraction, tool runner) |
| `radare2` | ✅ connected | Static/dynamic binary analysis |
| `ghidra` | ✅ connected | Decompilation and function analysis |
| `procmon` | ✅ connected | Windows Process Monitor log parsing |
| `remnux-docs` | ✅ connected | Tool documentation search |
| `x64dbg` | ⛔ disabled | Windows-only debugger (expected) |

---

#### Step 15: Take a Snapshot *(host, GUI)*

In VMware: **VM → Take Snapshot** → name it `clean-remnux`

This is your restore point. Revert to this snapshot before analyzing each new sample.

---

### Usage

#### Transfer a sample into REMnux *(host)*

```bash
scp /path/to/sample remnux@172.16.204.100:~/files/samples/
```

#### Analyze *(REMnux, inside OpenCode)*

```
Analyze the file in my samples directory
```

```
Extract IOCs from sample.exe
```

```
What tools do you recommend for this PE file?
```

```
Decompile the main function of sample.dll
```

OpenCode will automatically select and run the appropriate MCP tools.

#### Retrieve results *(host)*

```bash
scp remnux@172.16.204.100:~/files/output/ ~/Desktop/results/
```

#### Workflow per sample

| Step | Action |
|------|--------|
| 1 | Revert to `clean-remnux` snapshot |
| 2 | Power on REMnux |
| 3 | `scp` sample into `~/files/samples/` |
| 4 | Run `opencode` and analyze |
| 5 | Pull results from `~/files/output/` |
| 6 | Power off, take new snapshot if needed |

---

### Troubleshooting

| Symptom | Fix |
|---------|-----|
| `remnux install` fails with DNS error | Re-run the `echo` + `systemctl restart systemd-resolved` lines from Step 7 |
| `ping 8.8.8.8` fails but gateway responds | Add MASQUERADE rule on host (Step 7) |
| `ping 192.168.45.1` fails | Run `sudo /usr/bin/vmware-networks --start` on host |
| `dhclient: command not found` | Use `ip addr` / `ip route` manually (no dhclient on REMnux) |
| `nslookup` / `dig` not found | Use `getent hosts <domain>` instead |
| OpenCode: "Configuration is invalid" | Ensure `"model"` is a **string** (`"ollama/qwen3:4b"`), not an object |
| Ollama not reachable from REMnux | Verify `OLLAMA_HOST` matches vmnet1 IP and `ens33` is UP with correct subnet |
| `vmware-networks` not found after reboot | Re-run `sudo /usr/bin/vmware-networks --start` (or check `/etc/rc.local`) |
| `scp` fails: "Connection refused" | Run `sudo systemctl start ssh` inside REMnux |
| Netplan permission warning | Run `sudo chmod 600 /etc/netplan/01-netcfg.yaml` |

---

### Security Notes

- **No external connectivity** — `ens34` (NAT) is down. The only network path is REMnux → Ollama on the isolated host-only network.
- **No data leaves your machine** — all inference is local via Ollama.
- **Prompt injection risk** — if a sample contains strings designed to manipulate the AI, the VM cannot make outbound requests to external servers.
- **Snapshot isolation** — revert to a clean snapshot before each new sample to prevent cross-contamination.
- **For dynamic analysis** — attach a Windows victim VM to VMnet2 (the isolated host-only network) and point its gateway at REMnux.

---

### References

- [REMnux Documentation](https://docs.remnux.org)
- [REMnux MCP Server](https://github.com/REMnux/remnux-mcp-server)
- [OpenCode](https://opencode.ai)
- [Ollama](https://ollama.com)
- [Zeltser's Malware Analysis Lab Guide](https://zeltser.com/malware-analysis-lab-remnux)

---

#### Step 16: Build the Windows Victim VM *(host, GUI + Windows)*

This VM is where you run malware samples for dynamic analysis. It connects to the isolated VMnet2 host-only network (from Step 4) with REMnux as its gateway.

**Create the VM in VMware:**

1. **File → New Virtual Machine**
2. Choose **Typical** → **Next**
3. Select **Installer disc image file (ISO)** → browse to your Windows ISO → **Next**
4. Enter edition (e.g. Windows 11 Pro) → **Next**
5. Set: **2 CPUs**, **4096 MB RAM**, **80 GB disk**
6. **Customize Settings** before finishing:
   - **Network Adapter** → **Custom: VMnet2** (the isolated host-only from Step 4)
   - **Disable** Shared Folders and Drag and Drop
7. **Finish** and power on

**Install Windows** (standard OOBE — create a username with no spaces, e.g. `analyst`).

After Windows boots, run this single block in **PowerShell as Administrator** to disable security features and Windows Update:

```powershell
# Disable Windows Defender real-time protection
Set-MpPreference -DisableRealtimeMonitoring $true
Set-MpPreference -DisableBehaviorMonitoring $true
Set-MpPreference -DisableIOAVProtection $true
Set-MpPreference -DisableArchiving $true

# Disable Tamper Protection (requires registry)
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender" /v "DisableAntiSpyware" /t REG_DWORD /d 1 /f
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows Defender\Real-Time Protection" /v "DisableBehaviorMonitoring" /t REG_DWORD /d 1 /f

# Disable Windows Update
Set-Service -Name wuauserv -StartupType Disabled
Stop-Service -Name wuauserv -Force
Set-Service -Name bits -StartupType Disabled
Stop-Service -Name bits -Force

# Disable SmartScreen
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Explorer" /v "SmartScreenEnabled" /t REG_SZ /d "Off" /f

# Disable firewall
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled False

# Disable hibernation (prevents hiberfil.sys from interfering)
powercfg /hibernate off
```
