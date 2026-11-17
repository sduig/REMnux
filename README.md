## REMnux + MCP + Ollama: Fully Isolated AI-Assisted Malware Analysis Lab

A fully offline malware analysis lab built on [REMnux](https://remnux.org) with AI-assisted
analysis via [OpenCode](https://opencode.ai) and the [REMnux MCP Server](https://github.com/REMnux/remnux-mcp-server),
powered by a local LLM running on [Ollama](https://ollama.com).

No data leaves your machine. No internet connection required after initial setup.

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
| Ollama + qwen3:4b | Host (Linux Mint) | Local LLM inference (GPU-accelerated) |
| REMnux VM | VMware Workstation 17 | Malware analysis environment |
| OpenCode | Inside REMnux | AI agent / MCP client |
| REMnux MCP Server | Inside REMnux | Exposes analysis tools to the AI |
| vmnet1 (host-only) | VMware | Isolated network between host and VM |

┌─────────────────────────────────────────────────────────┐
│ Linux Mint Host │
│ │
│ ┌──────────────────────┐ ┌────────────────────┐ │
│ │ Ollama (host) │◄──────│ REMnux VM │ │
│ │ qwen3:4b │ vmnet1│ 172.16.204.100 │ │
│ │ 172.16.204.1:11434 │ host- │ ens33 = UP │ │
│ │ GPU: GTX 980M 4GB │ only │ ens34 = DOWN │ │
│ └──────────────────────┘ │ OpenCode + MCP │ │
│ └────────────────────┘ │
│ │
│ No external network connectivity from REMnux. │
└─────────────────────────────────────────────────────────┘
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

bash
sudo sh VMware-Workstation-Full-*.bundle

---

### Step 2: Start VMware Network Services (host)

sudo /usr/bin/vmware-networks --start
ps aux | grep vmnet   

You should see vmnet-natd, vmnet-dhcpd, and vmnet-netifup processes.

Make it persist across reboots:

echo "sudo /usr/bin/vmware-networks --start" | sudo tee -a /etc/rc.local
sudo chmod +x /etc/rc.local   

---
### Step 3: Identify Your Network Subnets (host)
ip addr show vmnet8
ip addr show vmnet1

---
### Step 4: Create an Isolated Host-Only Network (host, GUI)
1. Edit → Virtual Network Editor → Change Settings
2. Add Network → select VMnet2 (or next available)
3. Set type to Host-only
4. Uncheck "Connect a host virtual adapter to this network"
5. Enable DHCP within the assigned subnet
6. Apply
   
This network will later host a Windows victim VM for dynamic analysis.

---
### Step 5: Download and Import the REMnux OVA (host, GUI)
1. Download the General OVA from docs.remnux.org
2. Verify the SHA-256 hash
3. In VMware: File → Open → select the .ova → Import
4. Choose "I copied it" if prompted
   
This network will later host a Windows victim VM for dynamic analysis.

---
### Step 6: Configure REMnux VM Networking (host, GUI)
With the VM powered off, go to VM → Settings → Network:

|Adapter	| Type	| Purpose
|Adapter 1	| Custom → VMnet2 (isolated host-only)  | Lab traffic (no internet)
|Adapter 2	| NAT (vmnet8)	| Internet for initial setup only

Also set:

RAM: 2048 MB minimum
CPUs: 2
Disable Shared Folders and Drag and Drop under Options

| Inside the VM: Adapter 1 = ens33, Adapter 2 = ens34.

---
### Step 7: Boot and Fix NAT Networking (REMnux)
Power on the VM. It auto-logs in as remnux (no password prompt).

Replace 192.168.45.100 and 192.168.45.1 with your actual vmnet8 values, then copy-paste the entire block:
sudo ip addr flush dev ens34
sudo ip link set ens34 up
sudo ip addr add 192.168.45.100/24 dev ens34
sudo ip route add default via 192.168.45.1 dev ens34
echo -e "[Resolve]\nDNS=8.8.8.8" | sudo tee /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
ping -c 3 192.168.45.1
ping -c 3 8.8.8.8
getent hosts api.github.com   
If ping 8.8.8.8 fails (gateway responds but internet doesn't), on the host:
ip route | grep default
sudo iptables -t nat -A POSTROUTING -s 192.168.45.0/24 -o wlp2s0 -j MASQUERADE
sudo apt install -y iptables-persistent
sudo netfilter-persistent save   

ip route | grep default
sudo iptables -t nat -A POSTROUTING -s 192.168.45.0/24 -o wlp2s0 -j MASQUERADE
sudo apt install -y iptables-persistent
sudo netfilter-persistent save   
Replace wlp2s0 with your actual uplink and 192.168.45.0/24 with your vmnet8 subnet.
















