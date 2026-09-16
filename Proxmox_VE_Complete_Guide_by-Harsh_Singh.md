

**PROXMOX VE**

**Complete Installation & Cluster Configuration Guide**

*3-Node High Availability Cluster  |  Debian Bookworm  |  Version 3.0*

| Node | IP Address | Hostname | Role | Web UI Port |
| ----- | ----- | ----- | ----- | ----- |
| VPS 1 | 192.168.10.11 | node1 | Master / Primary | 8006 |
| VPS 2 | 192.168.10.12 | node2 | Worker Node | 8006 |
| VPS 3 | 192.168.10.13 | node3 | Worker Node | 8006 |

*Method A: Install on Debian  |  Method B: Direct ISO Install  |  GUI Cluster Join*

# **Table of Contents**

| Section | Title | Page |
| ----- | ----- | ----- |
| 1 | Pre-Installation: Debian OS Setup | 3 |
| 2 | Method A: Install Proxmox VE on Debian | 4 |
| 3 | Method B: Install Proxmox VE Directly (ISO — No OS) | 6 |
| 4 | Errors Encountered & Solutions (incl. live cluster join SSL cert issue) | 8 |
| 5 | Post-Installation: Opening Ports & Firewall | 9 |
| 6 | Accessing the Proxmox Web UI | 10 |
| 7 | Creating a 3-Node Cluster via Web UI (GUI) | 11 |
| 8 | Joining Nodes to Cluster via Web UI (GUI) | 13 |
| 9 | Post-Cluster Configuration | 15 |
| 10 | Creating VMs & Containers in Proxmox | 16 |
| 11 | Progress Checklist | 18 |
| 12 | Quick Reference & Command Glossary | 19 |
| 13 | Restricting Web UI Access to Trusted Networks Only (Proxmox Firewall) | 21 |
| 14 | Automating VM Provisioning with HostBill (Cloud-Init, API, Products) | 23 |
| 15 | Automating VM Provisioning with FOSSBilling (Free Open Source) | 26 |

# **Section 1 — Pre-Installation: Debian OS Setup**

This section covers the initial Debian Bookworm installation and base configuration steps required before installing Proxmox VE using Method A (on top of Debian).

*If you are using Method B (direct ISO install), skip to Section 3\.*

## **1.1  Debian OS Installation Notes**

During the Debian installation wizard, two important configuration prompts appear:

**Postfix — System Mail Name**

* The installer shows a 'System mail name' field

* Action: Keep the default hostname as-is (e.g. 7758-24442)

* Press Tab → select \<Ok\> → Enter to continue

| ⚠️  WARNING:   For VPS environments not sending email, this value does not affect cluster functionality. Keep default. |
| :---- |

**GRUB Bootloader Prompt**

* During apt upgrade, GRUB may show a configuration conflict prompt

* Always select: 'keep the local version currently installed'

| ⚠️  WARNING:   Never choose 'install the package maintainer's version' on VPS — can cause boot failure. |
| :---- |

## **1.2  Set Unique Hostnames on Each VPS**

Proxmox requires each node to have a unique hostname. Run on each respective VPS:

\# On VPS 1 (192.168.10.11)  
hostnamectl set-hostname node1  
   
\# On VPS 2 (192.168.10.12)  
hostnamectl set-hostname node2  
   
\# On VPS 3 (192.168.10.13)  
hostnamectl set-hostname node3  
   
\# Verify hostname on each node  
hostname

## **1.3  Configure /etc/hosts on ALL 3 Nodes**

Edit /etc/hosts to allow nodes to communicate using hostnames without DNS:

nano /etc/hosts  
   
\# Add these lines at the bottom:  
192.168.10.11  node1  
192.168.10.12   node2  
192.168.10.13  node3

| 💡  TIP:   Save with Ctrl+X → Y → Enter. This must be done identically on ALL 3 nodes. |
| :---- |

## **1.4  System Update**

apt update && apt upgrade \-y

# **Section 2 — Method A: Install Proxmox VE on Debian**

This is the method used in our live setup. Proxmox VE is installed on top of an existing Debian Bookworm installation. This is the officially supported method by Proxmox.

| Requirement | Details |
| ----- | ----- |
| Operating System | Debian 12 (Bookworm) — 64-bit |
| RAM | Minimum 2 GB (4 GB+ recommended) |
| Storage | Minimum 32 GB |
| Network | Static IP or consistent DHCP |
| Access | Root SSH access |

## **2.1  Fix Proxmox Repository**

By default, Debian may have the Proxmox Enterprise repository configured, which requires a paid subscription. Remove it and add the free repository:

\# Remove enterprise repo  
rm /etc/apt/sources.list.d/pve-enterprise.list  
   
\# Add free no-subscription repo  
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \\  
  \> /etc/apt/sources.list.d/pve-no-subscription.list  
   
\# Add Proxmox GPG key  
wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg \\  
  \-O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg  
   
\# Update package lists  
apt update && apt upgrade \-y

## **2.2  Install Proxmox VE**

apt install proxmox-ve postfix open-iscsi \-y

| ℹ️  INFO:   Installation takes 5-10 minutes. If Postfix prompts appear: select 'Local only' and keep the default mail name. |
| :---- |

## **2.3  Kernel Decision**

Standard Proxmox docs recommend removing the Debian kernel. This step is optional and was skipped in our setup for VPS stability.

| Step | Command | Recommendation for VPS |
| ----- | ----- | ----- |
| Remove Debian kernel | apt remove linux-image-amd64 'linux-image-6.1\*' \-y | Optional — skip for stability |
| Update GRUB | update-grub | Only if kernel was removed |

| 💡  TIP:   On bare-metal servers, removing the Debian kernel is recommended. On VPS environments, keeping it is safer. |
| :---- |

## **2.4  Reboot**

reboot

| ℹ️  INFO:   Wait 60-90 seconds after reboot before accessing the Web UI. |
| :---- |

## **2.5  Verify Proxmox Service**

systemctl status pveproxy  
   
\# Should show: active (running)

# **Section 3 — Method B: Direct ISO Install (No OS on VPS)**

This method installs Proxmox VE directly as the operating system — no Debian pre-installation needed. This is the cleanest and most recommended method for bare-metal servers and VPS providers that allow custom ISO mounting.

| ⚠️  WARNING:   Not all VPS providers support ISO mounting. Check with your provider before attempting this method. |
| :---- |

## **3.1  Requirements**

| Requirement | Details |
| ----- | ----- |
| VPS Provider | Must support custom ISO upload or mounting |
| ISO File | Proxmox VE latest ISO from proxmox.com |
| RAM | Minimum 2 GB (4 GB+ recommended) |
| Storage | Minimum 32 GB SSD |
| Console Access | VNC or noVNC console from provider panel |

## **3.2  Download Proxmox VE ISO**

Download the latest Proxmox VE ISO from the official website:

\# Official download URL:  
https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso  
   
\# Example filename:  
proxmox-ve\_8.x.x-1.iso

## **3.3  Mount ISO on VPS (Provider Panel Steps)**

The exact steps vary by provider but follow this general flow:

| 1 | Login to VPS Provider Control Panel Navigate to your VPS management dashboard |
| :---: | :---- |

| 2 | Find ISO / Media Section Look for 'ISO Library', 'Media', or 'OS Templates' in the panel |
| :---: | :---- |

| 3 | Upload or Link Proxmox ISO Either upload the ISO file or paste the download URL if provider supports remote ISO |
| :---: | :---- |

| 4 | Mount ISO to VPS In VPS settings → select boot device → choose the uploaded Proxmox ISO |
| :---: | :---- |

| 5 | Set Boot Order Set CD/DVD (ISO) as first boot device |
| :---: | :---- |

| 6 | Open VNC Console Open the noVNC or VNC console from the provider panel before rebooting |
| :---: | :---- |

| 7 | Reboot VPS Reboot the VPS — the Proxmox installer will appear in the VNC console |
| :---: | :---- |

## **3.4  Proxmox VE Installer Walkthrough**

Once the VPS boots from the ISO, the graphical Proxmox installer appears. Follow these steps:

**Step 1 — Welcome Screen**

* Click: Install Proxmox VE (Graphical)

**Step 2 — License Agreement**

* Click: I agree

**Step 3 — Target Disk Selection**

* Select the disk to install Proxmox VE on

* Click Options to configure filesystem (ext4 or ZFS recommended)

* Click Next

| ⚠️  WARNING:   All data on the selected disk will be erased. Make sure you select the correct disk. |
| :---- |

**Step 4 — Location and Timezone**

* Country: Select your country (e.g. India)

* Timezone: Asia/Kolkata (or your timezone)

* Keyboard: en-us

* Click Next

**Step 5 — Admin Password & Email**

* Password: Set a strong root password (minimum 8 characters)

* Confirm Password: Re-enter the same password

* Email: Enter an admin email (used for alerts)

* Click Next

| 💡  TIP:   Save this root password securely — you will need it to login to the Web UI. |
| :---- |

**Step 6 — Network Configuration**

* Management Interface: Select the main network interface (usually eth0 or ens3)

* Hostname (FQDN): Enter the node hostname — e.g. pve-master.homelab.local

* IP Address: Enter your VPS IP — e.g. 192.168.10.11

* Netmask: Enter your subnet mask — e.g. 255.255.255.0

* Gateway: Enter your gateway IP (from provider panel)

* DNS Server: 8.8.8.8 (or provider DNS)

* Click Next

| ⚠️  WARNING:   The IP, Netmask and Gateway values must exactly match your VPS network settings from the provider panel. |
| :---- |

**Step 7 — Summary & Install**

* Review all settings on the summary screen

* Click Install to begin the installation

* Installation takes 5-15 minutes

* System will reboot automatically when done

**Step 8 — After Reboot**

* Remove the ISO from boot device in provider panel after reboot

* Set boot order back to: Hard Disk first

* The VPS will now boot directly into Proxmox VE

| ℹ️  INFO:   After successful install, Proxmox VE will show a console message with the Web UI URL: https://\<your-ip\>:8006 |
| :---- |

# **Section 4 — Errors Encountered & Solutions**

This section documents every error encountered during the live setup with root cause analysis and exact fixes.

## **Error 1 — Proxmox Enterprise Repository 401 Unauthorized**

| 🔴  ERROR Err:7 https://enterprise.proxmox.com/debian/pve bookworm InRelease  401  Unauthorized \[IP: 117.120.5.231 443\]E: Failed to fetch .../InRelease 401 UnauthorizedE: The repository is not signed. Root Cause: The enterprise.proxmox.com repository requires a paid subscription licence. Without a valid key, every apt update returns HTTP 401 Unauthorized. |
| :---- |
| **✅  SOLUTION** Remove the enterprise repo. Add the official free no-subscription repository instead. Apply on ALL 3 nodes. rm /etc/apt/sources.list.d/pve-enterprise.list echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \\   \> /etc/apt/sources.list.d/pve-no-subscription.list wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg \\   \-O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg apt update && apt upgrade \-y |

## **Error 2 — Web UI Port 8006 ERR\_TIMED\_OUT**

| 🔴  ERROR ERR\_TIMED\_OUT — https://192.168.10.11:8006192.168.10.11 took too long to respond. Root Cause: Two-layer firewall block: (1) OS-level iptables was not allowing inbound port 8006\. (2) VPS provider network-level firewall was blocking port 8006 externally before packets even reached the server. |
| :---- |
| **✅  SOLUTION** Two-step fix: First open port 8006 in iptables on the server. Second, contact VPS provider to open port 8006 at network/datacenter level. \# Fix 1 — OS level (run on ALL 3 nodes) iptables \-A INPUT \-p tcp \--dport 8006 \-j ACCEPT iptables \-A INPUT \-p tcp \--dport 443  \-j ACCEPT iptables \-A INPUT \-p tcp \--dport 80   \-j ACCEPT apt install iptables-persistent \-y netfilter-persistent save   \# Verify port is listening ss \-tlnp | grep 8006   \# Fix 2 — Contact VPS provider to open port 8006 at network level |

## **Error 3 — Cluster Join Warning: No Cluster Network Links Passed**

| 🔴  ERROR No cluster network links passed explicitly, falling back to default.Request addition of this node... (hangs or fails) Root Cause: When joining a node to the cluster via pvecm, no dedicated cluster network interface was explicitly specified. Proxmox falls back to the default interface (vmk0/eth0) which may not be routable between nodes. |
| :---- |
| **✅  SOLUTION** Wait 1-2 minutes first — if the join completes with TASK OK, no action needed. If it hangs or fails, explicitly specify the network interface during the join command. \# Check which VMkernel/interface is active on the joining node: ip a   \# Rejoin with explicit network interface specified: pvecm add 192.168.10.11 \--link0 \<joining-node-ip\>   \# Example — joining node2 with explicit IP: pvecm add 192.168.10.11 \--link0 192.168.10.12   \# Verify cluster status after join: pvecm status pvecm nodes |

| ℹ️  INFO:   This warning message is informational in most cases. As long as the join process completes with TASK OK and pvecm nodes shows all 3 nodes, the cluster is healthy. |
| :---- |

## **Error 4 — SSL Certificate Missing After Node Join: pve-ssl.pem Does Not Exist**

| 🔴  ERROR '/etc/pve/nodes/node2/pve-ssl.pem' does not exist\! (500)Node2 shows error in Proxmox Web UI after cluster join. Root Cause: After node2 joined the cluster, its SSL certificate was not correctly generated or synced to node1's cluster filesystem (/etc/pve/nodes/node2/). This is a certificate sync issue — NOT a network or cluster quorum issue. The cluster itself is healthy. |
| :---- |
| **✅  SOLUTION** Regenerate the SSL certificate on node2 using pvecm updatecerts \--force, restart cluster services on both nodes, then verify the cert file exists in /etc/pve/nodes/node2/ on node1. \# ── STEP 1: Verify cluster IS actually healthy (run on node1) ── pvecm status pvecm nodes \# Expected: all 3 nodes listed, Quorum: Yes   \# ── STEP 2: SSH from node1 into node2 ── ssh root@192.168.10.12 \# Type: yes  (accept SSH fingerprint) \# Enter node2 root password   \# ── STEP 3: Regenerate SSL cert on node2 ── pvecm updatecerts \--force \# Expected output: \# generate new node certificate \# merge authorized SSH keys   \# ── STEP 4: Restart all cluster services on node2 ── systemctl restart pvedaemon pveproxy pve-cluster   \# ── STEP 5: Verify cert now exists on node2 ── ls /etc/pve/nodes/node2/ \# You should see: pve-ssl.pem  pve-ssl.key  pve.crt   \# ── STEP 6: Back on node1 — restart services ── systemctl restart pve-cluster pvedaemon pveproxy   \# ── STEP 7: Verify cert synced to node1 ── ls /etc/pve/nodes/node2/ \# pve-ssl.pem must appear here on node1 too   \# ── STEP 8: Repeat for node3 if needed ── ssh root@192.168.10.13 pvecm updatecerts \--force systemctl restart pvedaemon pveproxy pve-cluster |

| ℹ️  INFO:   This error was encountered live during the 3-node cluster setup in this guide. The cluster quorum was healthy (all 3 nodes joined successfully) — only the SSL certificate sync had failed. After pvecm updatecerts \--force the Web UI error was fully resolved. |
| :---- |

## **Error 4b — SSH Fingerprint Prompt When Connecting Between Nodes**

| 🔴  ERROR The authenticity of host '192.168.10.12' can't be established.ECDSA key fingerprint is SHA256:xxxx...Are you sure you want to continue connecting (yes/no/\[fingerprint\])? Root Cause: First-time SSH connection between cluster nodes — SSH has not yet stored the remote host's fingerprint in known\_hosts. This is expected and normal on first inter-node connection. |
| :---- |
| **✅  SOLUTION** Type 'yes' and press Enter to accept and store the fingerprint. This is safe — you are connecting to your own cluster node. \# When prompted — always type: yes   \# Then enter the node's root password when asked   \# To pre-accept all node fingerprints (run on node1): ssh-keyscan 192.168.10.12 \>\> \~/.ssh/known\_hosts ssh-keyscan 192.168.10.13 \>\> \~/.ssh/known\_hosts   \# Set up passwordless SSH between nodes (optional but recommended): ssh-keygen \-t rsa \-b 4096   \# Generate key on node1 ssh-copy-id root@192.168.10.12 ssh-copy-id root@192.168.10.13 |

## **Error 5 — Cluster Join Fails: Corosync Port Not Reachable**

| 🔴  ERROR pvecm add failed: corosync not reachable on node1cluster join aborted — cannot connect to quorum service Root Cause: Corosync UDP ports 5404 and 5405 are not open between nodes. The joining node cannot reach the cluster quorum service on the master node. This is a firewall / provider-level port blocking issue. |
| :---- |
| **✅  SOLUTION** Open UDP ports 5404 and 5405 in iptables on ALL nodes AND request the VPS provider to open these ports at network level. \# Open corosync ports on ALL 3 nodes: iptables \-A INPUT \-p udp \--dport 5404 \-j ACCEPT iptables \-A INPUT \-p udp \--dport 5405 \-j ACCEPT iptables \-A OUTPUT \-p udp \--dport 5404 \-j ACCEPT iptables \-A OUTPUT \-p udp \--dport 5405 \-j ACCEPT netfilter-persistent save   \# Test corosync port reachability from node2 to node1: nc \-vzu 192.168.10.11 5404 nc \-vzu 192.168.10.11 5405   \# Check corosync service status on node1: systemctl status corosync corosync-cfgtool \-s |

# **Section 5 — Post-Installation: Ports & Firewall**

All required ports must be open at both the OS level (iptables) and the provider/network level. Request all ports from your provider at once.

## **5.1  Required Ports Reference**

| Port | Protocol | Purpose | Priority |
| ----- | ----- | ----- | ----- |
| 22 | TCP | SSH access | Critical |
| 80 | TCP | HTTP redirect | Required |
| 443 | TCP | HTTPS | Required |
| 8006 | TCP | Proxmox Web UI | Critical |
| 111 | TCP+UDP | rpcbind / NFS portmapper | Cluster |
| 3128 | TCP | SPICE proxy (VM console) | Optional |
| 5404 | UDP | Corosync cluster communication | Critical for cluster |
| 5405 | UDP | Corosync cluster communication | Critical for cluster |
| 5900-5999 | TCP | VNC console access to VMs | Optional |
| 8007 | TCP | Proxmox backup server | Optional |

## **5.2  Open Ports in iptables (OS Level)**

\# Run on ALL 3 nodes  
iptables \-A INPUT \-p tcp \--dport 22   \-j ACCEPT  
iptables \-A INPUT \-p tcp \--dport 80   \-j ACCEPT  
iptables \-A INPUT \-p tcp \--dport 443  \-j ACCEPT  
iptables \-A INPUT \-p tcp \--dport 8006 \-j ACCEPT  
iptables \-A INPUT \-p tcp \--dport 3128 \-j ACCEPT  
iptables \-A INPUT \-p udp \--dport 5404 \-j ACCEPT  
iptables \-A INPUT \-p udp \--dport 5405 \-j ACCEPT  
iptables \-A INPUT \-p tcp \--dport 111  \-j ACCEPT  
iptables \-A INPUT \-p udp \--dport 111  \-j ACCEPT  
   
\# Save rules persistently  
apt install iptables-persistent \-y  
netfilter-persistent save  
   
\# Verify  
iptables \-L \-n | grep ACCEPT

## **5.3  Provider-Level Port Request Template**

Send this message to your VPS provider support to open ports at the network level:

Subject: Open Ports on 3 VPS Servers for Proxmox VE Cluster  
   
Please open the following ports on all 3 of my VPS servers:  
   
Ports (TCP): 22, 80, 443, 8006, 3128, 111, 5900-5999  
Ports (UDP): 5404, 5405, 111  
   
VPS Server IPs:  
  192.168.10.11  (node1 — Master)  
  192.168.10.12   (node2 — Worker)  
  192.168.10.13  (node3 — Worker)

## **5A.5 WireGuard Encrypted Inter-Node Overlay — Final Implementation**

### **5A.5.1 Why WireGuard Was Used**


((((# Also we have tried for the GRE  but due to provider  network limitation cant able to do for reference  attache beloe
)))(((THis includes the network bridge creation and all)))) [[[[[[## **Production Network Architecture, VLANs & Dedicated Bridges**

This section documents the production network layer used by the 3-node Proxmox environment. It is intentionally placed immediately after the base firewall/port configuration and before Web UI and cluster-management procedures, because network topology and bridge design affect VM/CT connectivity, migration, and future service isolation.

## **5A.1 Production Network Design Goals**

The production network is designed to keep the following traffic logically separated wherever the provider/network topology supports it:

| Traffic Type       | Purpose                                    | Recommended Isolation                |
| ------------------ | ------------------------------------------ | ------------------------------------ |
| Management         | Proxmox Web UI, SSH, node administration   | Management network / trusted access  |
| Cluster / Corosync | Node heartbeat and cluster control traffic | Dedicated private network            |
| VM / Container     | Customer and workload traffic              | VLAN-aware bridge                    |
| Overlay / VPN      | Encrypted private inter-node connectivity  | WireGuard tunnel                     |
| Backup             | VM/CT backup traffic                       | Dedicated network or restricted path |

**Important:** VLAN IDs, IP ranges, gateways, and physical/virtual interface names must match the actual network supplied by the VPS/provider. Do not copy example values from this guide directly into production.

## **5A.2 VLAN-Aware Linux Bridge**

For production VM and container networking, the Proxmox bridge can be configured as VLAN-aware. This allows multiple VLANs to be carried through the same bridge while assigning the required VLAN tag to individual VM/CT network interfaces.

### Web UI

1. Open **Node → System → Network**.
2. Select the production VM bridge (for example `vmbr0` or the dedicated VM bridge).
3. Click **Edit**.
4. Enable **VLAN aware**.
5. Verify the bridge port/interface is the intended uplink.
6. Click **OK** → **Apply Configuration**.
7. Repeat the required configuration consistently on all cluster nodes.

### Verify from terminal

```bash
ip -br link
ip -br addr
bridge link
cat /etc/network/interfaces
```

The bridge configuration should show VLAN awareness and the correct uplink/port.

## **5A.3 Dedicated VM/Container Bridge**

Where the production design requires a separate bridge for workload traffic, create a dedicated Linux bridge rather than mixing workload traffic with the management bridge.

Example structure:

```text
Provider / Uplink
       |
       +---------------- Management
       |
       +---------------- VM/CT VLAN traffic
                          |
                        vmbrX
                       /    \
                     VM      CT
```

The exact bridge name, physical/uplink interface, IP addressing, and VLAN configuration must be taken from the actual production network design.

**Production rule:** Do not add a default gateway to a dedicated private/overlay bridge unless that bridge is explicitly intended to provide the node's default routing.

## **5A.4 GRE Network / Overlay — Initial Attempt**

GRE was initially considered for creating a private overlay network between the Proxmox nodes.

The initial design involved establishing GRE tunnels between the nodes and using the resulting private network for inter-node communication.

However, the required GRE functionality was **not enabled/supported through the provider network**, so the GRE-based approach could not be used reliably in the production environment.

Therefore, the GRE tunnels were not retained as part of the final implementation.

### Initial GRE Verification

During the initial attempt, the following commands were used to verify the tunnel:

```bash
ip tunnel show
ip -br addr
ip route
ping -c 4 <remote-gre-ip>
```

If a GRE interface was attached to a Linux bridge, the relationship could be checked using:

```bash
bridge link
ip link show
```

### GRE Limitation

**⚠️ WARNING:** GRE itself does not provide encryption. Even if GRE functionality is available, sensitive production traffic should use an encrypted transport such as WireGuard or IPsec.

Because GRE was not usable through the provider network, the final implementation was changed to **WireGuard**.

## **5A.5 WireGuard Encrypted Inter-Node Overlay — Final Implementation**

### **5A.5.1 Why WireGuard Was Used**

After the GRE implementation was found to be unsuitable because the required GRE functionality was not enabled/supported by the provider network, **WireGuard was selected as the final overlay solution**.

WireGuard provides:

* Encrypted node-to-node communication
* Secure peer authentication using public/private keys
* Lightweight VPN tunneling
* Persistent tunnel support through systemd
* Direct connectivity between all three Proxmox nodes
* No dependency on provider-side GRE support

The final production topology uses WireGuard for encrypted private connectivity between:

```text
Node1 <--------> Node2
  |                |
  |                |
  +--------------> Node3
         ^
         |
       Node2
```

All three nodes were configured as WireGuard peers.

### **5A.5.2 Final WireGuard Connectivity**

The final WireGuard implementation was verified successfully:

| Connection                    | Status       |
| ----------------------------- | ------------ |
| Node1 ↔ Node2                 | ✅ Working    |
| Node1 ↔ Node3                 | ✅ Working    |
| Node2 ↔ Node3                 | ✅ Working    |
| WireGuard tunnel after reboot | ✅ Persistent |
| Old GRE configuration         | ✅ Removed    |

### **5A.5.3 WireGuard Installation**

Install WireGuard on **all three Proxmox nodes**:

```bash
apt update
apt install wireguard -y
```

Verify the installation:

```bash
wg --version
```

### **5A.5.4 WireGuard Configuration Directory**

WireGuard configuration files are stored under:

```bash
/etc/wireguard/
```

Create and secure the directory:

```bash
mkdir -p /etc/wireguard
chmod 700 /etc/wireguard
```

The primary tunnel used in this implementation is:

```text
wg0
```

with its configuration stored at:

```text
/etc/wireguard/wg0.conf
```

### **5A.5.5 WireGuard Key Generation**

Each Proxmox node requires its own private/public key pair.

Run on each node:

```bash
cd /etc/wireguard
umask 077

wg genkey | tee privatekey | wg pubkey > publickey
```

Check the generated files:

```bash
ls -l /etc/wireguard/
```

The following files are created:

```text
/etc/wireguard/privatekey
/etc/wireguard/publickey
```

**Security:** Never share or document the private key. Only the public key should be exchanged between peers.

### **5A.5.6 Create the `wg0.conf` Configuration**

Create the WireGuard configuration:

```bash
nano /etc/wireguard/wg0.conf
```

Generic node configuration structure:

```ini
[Interface]
Address = <node-wireguard-ip>/<subnet>
PrivateKey = <node-private-key>
ListenPort = <wireguard-udp-port>

[Peer]
PublicKey = <remote-node-public-key>
AllowedIPs = <remote-node-wireguard-ip>/32
Endpoint = <remote-node-public-ip>:<wireguard-udp-port>
PersistentKeepalive = 25
```

Each node contains its own `[Interface]` section and the appropriate `[Peer]` entries for the other Proxmox nodes.

**Important:** Actual WireGuard IP addresses, public keys, private keys, UDP port, and public endpoints must be taken from the production configuration. Do not replace them with example values from this document.

### **5A.5.7 Three-Node WireGuard Peer Design**

The final design is a full three-node peer topology:

```text
                WireGuard VPN
             Encrypted Overlay
                    |
        +-----------+-----------+
        |                       |
     Node1                   Node2
        | \                   / |
        |  \                 /  |
        |   \               /   |
        |    +---- Node3 ---+   |
        |                       |
        +-----------------------+
```

Logical peer relationships:

```text
Node1
 ├── Peer: Node2
 └── Peer: Node3

Node2
 ├── Peer: Node1
 └── Peer: Node3

Node3
 ├── Peer: Node1
 └── Peer: Node2
```

This ensures that every Proxmox node can communicate with the other two through the encrypted WireGuard overlay.

### **5A.5.8 Protect the WireGuard Configuration**

The WireGuard configuration contains the node's private key and must therefore be protected:

```bash
chmod 600 /etc/wireguard/wg0.conf
chmod 600 /etc/wireguard/privatekey
chmod 644 /etc/wireguard/publickey
```

Verify permissions:

```bash
ls -l /etc/wireguard/
```

### **5A.5.9 Start the WireGuard Tunnel**

Bring up the WireGuard interface:

```bash
wg-quick up wg0
```

Verify the interface:

```bash
ip addr show wg0
```

Check WireGuard peer status:

```bash
wg show
```

Verify routing:

```bash
ip route
```

### **5A.5.10 Enable WireGuard at Boot**

WireGuard was configured to start automatically after a node reboot.

Enable the systemd service:

```bash
systemctl enable wg-quick@wg0
```

Start the service:

```bash
systemctl start wg-quick@wg0
```

Verify:

```bash
systemctl status wg-quick@wg0 --no-pager
```

The service should show:

```text
active (exited)
```

or an equivalent successful WireGuard startup state.

### **5A.5.11 Firewall / UDP Requirement**

WireGuard requires its configured UDP listening port to be reachable between the participating nodes.

Verify the listening socket:

```bash
ss -lunp
```

The configured WireGuard UDP port must be permitted by the applicable:

* Proxmox firewall rules
* VPS/provider firewall
* Host firewall
* Security group, if applicable

The exact UDP port must match the production WireGuard configuration.

### **5A.5.12 Verify Node-to-Node Connectivity**

After bringing up WireGuard on all nodes, verify:

```bash
wg show
```

The output should show the configured peers and recent handshakes.

Verify the WireGuard interface:

```bash
ip -br addr show wg0
```

Test the private WireGuard IP of each remote node:

```bash
ping -c 4 <node2-wireguard-ip>
ping -c 4 <node3-wireguard-ip>
```

Perform equivalent tests from Node2 and Node3.

Expected result:

```text
Node1 → Node2  ✅
Node1 → Node3  ✅
Node2 → Node1  ✅
Node2 → Node3  ✅
Node3 → Node1  ✅
Node3 → Node2  ✅
```

### **5A.5.13 Reboot Persistence Verification**

Because the WireGuard tunnel is required for persistent inter-node connectivity, reboot testing was performed after enabling the systemd service.

After reboot:

```bash
systemctl status wg-quick@wg0 --no-pager
```

Then verify:

```bash
wg show
ip addr show wg0
```

Finally test connectivity to the other nodes:

```bash
ping -c 4 <remote-node-wireguard-ip>
```

**Result:** WireGuard successfully comes back after reboot and the three-node encrypted connectivity remains operational.

### **5A.5.14 GRE Cleanup**

Once WireGuard was confirmed working, the old GRE configuration was removed to prevent stale interfaces, routes, or conflicting overlay configuration.

Remove old GRE entries from:

```bash
nano /etc/network/interfaces
```

Delete the old GRE-related configuration.

Remove the previously created GRE tunnels:

```bash
ip tunnel del gre-node2 2>/dev/null
ip tunnel del gre-node3 2>/dev/null
ip tunnel del gre-node1 2>/dev/null
```

Verify that no unwanted GRE tunnel remains:

```bash
ip tunnel show
```

Verify the final routing and interfaces:

```bash
ip -br addr
ip route
```

The final production environment should contain the **WireGuard overlay**, not the abandoned GRE tunnels.

### **5A.5.15 Final Implementation Status**

The production overlay networking was finalized using WireGuard because GRE could not be used through the provider network.

| Component                    | Final Status                                     |
| ---------------------------- | ------------------------------------------------ |
| GRE attempt                  | ⚠️ Not usable due to provider network limitation |
| WireGuard installation       | ✅ Completed                                      |
| WireGuard key generation     | ✅ Completed                                      |
| `wg0` configuration          | ✅ Completed                                      |
| Node1 ↔ Node2                | ✅ Working                                        |
| Node1 ↔ Node3                | ✅ Working                                        |
| Node2 ↔ Node3                | ✅ Working                                        |
| WireGuard persistent startup | ✅ Enabled                                        |
| Reboot verification          | ✅ Passed                                         |
| Old GRE cleanup              | ✅ Completed                                      |
| Encrypted inter-node overlay | ✅ Operational                                    |

**Final architecture:**

```text
                 Provider Network
                        |
          +-------------+-------------+
          |             |             |
        Node1         Node2         Node3
          | \           / \           / |
          |  \         /   \         /  |
          |   +-------+-----+-------+   |
          |        WireGuard VPN        |
          +-----------------------------+
                 Encrypted
              Inter-Node Overlay
```

The WireGuard overlay is now the **final encrypted inter-node network path** used by the three-node Proxmox environment. GRE was only the initial approach and has been removed after the provider-side limitation was identified.

## **5A.6 `/etc/network/interfaces` Change Control**

Network configuration changes on a production Proxmox node must be performed carefully because an incorrect bridge, interface, route, or gateway can disconnect the node.

Before making changes:

```bash
cp /etc/network/interfaces /etc/network/interfaces.bak

ip -br addr
ip route
```

After changes:

```bash
ifreload -a

ip -br addr
ip route
bridge link
```

If the node becomes unreachable, use the VPS provider's console/out-of-band access to restore the previous working configuration.

## **5A.7 VLAN Assignment to VMs and Containers**

Once the VLAN-aware bridge is operational, a VLAN can be assigned per VM/CT network interface.

### VM

**VM → Hardware → Network Device → Edit**

* Bridge: select the production VLAN-aware bridge
* VLAN Tag: enter the required VLAN ID
* Model: VirtIO

### LXC Container

**CT → Network → Edit**

* Bridge: select the production VLAN-aware bridge
* VLAN Tag: enter the required VLAN ID where supported by the chosen network design
* Configure the container IP/gateway according to the assigned VLAN

Example logical design:

| Workload    | Bridge          | VLAN Tag    | Purpose            |
| ----------- | --------------- | ----------- | ------------------ |
| Management  | `vmbr0`         | — / native  | Proxmox management |
| Customer VM | `<vmbr-vm>`     | `<VLAN-ID>` | Customer workload  |
| Customer CT | `<vmbr-vm>`     | `<VLAN-ID>` | Customer workload  |
| Backup      | `<vmbr-backup>` | `<VLAN-ID>` | Backup traffic     |

Replace placeholders only after the provider/network VLAN plan is finalized.

## **5A.8 Network Validation Checklist**

Before proceeding to production VM/CT provisioning, validate:

* [ ] VLAN-aware bridge enabled on required nodes
* [ ] Correct uplink/interface attached to the bridge
* [ ] Dedicated VM/CT bridge verified where required
* [x] GRE attempted and provider-side limitation identified
* [x] Old GRE tunnels/configuration removed
* [x] WireGuard installed on all three nodes
* [x] WireGuard peer configuration completed
* [x] Node1 ↔ Node2 connectivity verified
* [x] Node1 ↔ Node3 connectivity verified
* [x] Node2 ↔ Node3 connectivity verified
* [x] WireGuard persistence after reboot verified
* [ ] `/etc/network/interfaces` consistent with the approved final design
* [ ] Node management connectivity still working
* [ ] VLAN-tagged test VM receives expected connectivity
* [ ] VLAN-tagged test CT receives expected connectivity
* [ ] Default gateway exists only where required
* [ ] Firewall rules permit required VLAN/overlay traffic
* [ ] Live migration networking remains functional
]]]]]]]]]

During the initial production network implementation, a **GRE-based private overlay** was attempted between the Proxmox nodes.

However, the required GRE functionality was not available/usable through the VPS provider's network. Due to this provider-side network limitation, the GRE tunnel could not be used as the final inter-node communication method.

Therefore, **WireGuard was selected and implemented as the final encrypted overlay network** between the three Proxmox nodes.

The final implementation provides secure private connectivity between:

| Connection    | Status    |
| ------------- | --------- |
| node1 ↔ node2 | ✅ Working |
| node1 ↔ node3 | ✅ Working |
| node2 ↔ node3 | ✅ Working |

WireGuard was also configured to start automatically after system boot, making the tunnel persistent across reboots.

---

### **5A.5.2 Final Network Approach**

The network implementation followed this sequence:

```text
                VPS Provider Network
                        |
                        |
                 GRE initially tried
                        |
                        v
              GRE not usable/supported
                 through provider
                        |
                        v
                WireGuard selected
                        |
          +-------------+-------------+
          |             |             |
        node1         node2         node3
          |             |             |
         wg0           wg0           wg0
          |             |             |
          +-------------+-------------+
                Encrypted Overlay
```

**Final decision:** GRE was treated as the initial/failed implementation attempt, while **WireGuard became the final production overlay**.

---

### **5A.5.3 WireGuard Installation**

WireGuard was installed on all three Proxmox nodes.

```bash
apt update
apt install wireguard -y
```

Verify the installation:

```bash
wg --version
```

The installation was performed on:

```text
node1
node2
node3
```

---

### **5A.5.4 WireGuard Configuration Directory**

WireGuard configuration files are maintained under:

```bash
/etc/wireguard/
```

The primary tunnel interface used for the implementation is:

```text
wg0
```

The main configuration file is:

```bash
/etc/wireguard/wg0.conf
```

Create the directory if required:

```bash
mkdir -p /etc/wireguard
chmod 700 /etc/wireguard
```

---

### **5A.5.5 WireGuard Key Generation**

Each Proxmox node requires its own WireGuard key pair.

The general key-generation process is:

```bash
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```

This generates:

```text
privatekey
publickey
```

Each node keeps its own private key.

**⚠️ SECURITY WARNING:** Never place actual WireGuard private keys inside this documentation, Git repositories, screenshots, or public support messages.

Only the **public key** should be exchanged with the other WireGuard peers.

---

### **5A.5.6 Create the `wg0.conf` Configuration**

The WireGuard tunnel configuration is stored at:

```bash
/etc/wireguard/wg0.conf
```

Create/edit the file:

```bash
nano /etc/wireguard/wg0.conf
```

The configuration follows the standard WireGuard structure:

```ini
[Interface]
Address = <node-wireguard-ip>/<subnet>
PrivateKey = <node-private-key>
ListenPort = <wireguard-udp-port>

[Peer]
PublicKey = <remote-node-public-key>
AllowedIPs = <remote-node-wireguard-ip>/32
Endpoint = <remote-node-public-ip>:<wireguard-udp-port>
PersistentKeepalive = 25
```

Additional `[Peer]` sections are configured for the remaining Proxmox nodes.

**Note:** The actual WireGuard IP addresses, public keys, endpoint addresses and UDP port used on the live cluster should be inserted here if this document is intended to contain the complete production configuration.

---

### **5A.5.7 Three-Node WireGuard Peer Design**

The final WireGuard design provides encrypted communication between all three Proxmox nodes.

```text
node1
 └── wg0
      ├── Peer → node2
      └── Peer → node3

node2
 └── wg0
      ├── Peer → node1
      └── Peer → node3

node3
 └── wg0
      ├── Peer → node1
      └── Peer → node2
```

This creates direct encrypted paths between all nodes.

The `AllowedIPs` configuration determines which destination addresses are routed through each WireGuard peer.

---

### **5A.5.8 Protect the WireGuard Configuration**

Because `wg0.conf` contains the node's private key, its permissions must be restricted.

```bash
chmod 600 /etc/wireguard/wg0.conf
```

Verify:

```bash
ls -l /etc/wireguard/wg0.conf
```

The configuration should only be accessible by the required privileged account.

---

### **5A.5.9 Start the WireGuard Tunnel**

For the initial test, bring the tunnel up manually:

```bash
wg-quick up wg0
```

Verify the WireGuard interface:

```bash
ip addr show wg0
```

Check WireGuard status:

```bash
wg show
```

Check routing:

```bash
ip route
```

The `wg show` output should display the configured peers and, after successful communication, a recent handshake.

---

### **5A.5.10 Enable WireGuard at Boot**

After confirming that the tunnel works correctly, WireGuard was configured to start automatically with the operating system.

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

Verify:

```bash
systemctl status wg-quick@wg0 --no-pager
```

This ensures that the WireGuard interface is automatically restored after a node reboot.

---

### **5A.5.11 Firewall / UDP Requirement**

WireGuard communicates through UDP using the configured listening port.

Verify the listening socket:

```bash
ss -lunp
```

The configured WireGuard UDP port must be permitted by:

* VPS provider/network firewall
* OS firewall, if enabled
* Any additional upstream firewall/security group

This was one of the important reasons for moving away from GRE: the provider network did not provide the required GRE functionality, while WireGuard could operate using a normal provider-supported UDP transport.

---

### **5A.5.12 Verify Node-to-Node Connectivity**

After WireGuard was configured on all three nodes, connectivity was tested between the WireGuard/private addresses.

On each node:

```bash
wg show
ip -br addr show wg0
ip route
```

Test the remote WireGuard addresses:

```bash
ping -c 4 <node2-wireguard-ip>
ping -c 4 <node3-wireguard-ip>
```

The same tests should be performed from node2 and node3.

Final connectivity result:

| Test                      | Result |
| ------------------------- | ------ |
| node1 → node2             | ✅      |
| node1 → node3             | ✅      |
| node2 → node1             | ✅      |
| node2 → node3             | ✅      |
| node3 → node1             | ✅      |
| node3 → node2             | ✅      |
| WireGuard handshake       | ✅      |
| Encrypted private overlay | ✅      |

---

### **5A.5.13 Reboot Persistence Verification**

WireGuard persistence was also verified after reboot.

After restarting a node:

```bash
reboot
```

Check the WireGuard service:

```bash
systemctl status wg-quick@wg0 --no-pager
```

Verify the interface:

```bash
ip addr show wg0
```

Check peer status:

```bash
wg show
```

Finally, test connectivity again:

```bash
ping -c 4 <remote-wireguard-ip>
```

Successful connectivity after reboot confirms that the WireGuard overlay is persistent and automatically restored.

---

### **5A.5.14 GRE Cleanup**

Once WireGuard was confirmed to be working, the old GRE configuration was removed to avoid unnecessary tunnel interfaces or routing conflicts.

Edit the network configuration:

```bash
nano /etc/network/interfaces
```

Remove the old GRE-related entries that were created during the initial GRE implementation attempt.

The previously created GRE interfaces were then removed where applicable:

```bash
ip tunnel del gre-node2 2>/dev/null
ip tunnel del gre-node3 2>/dev/null
ip tunnel del gre-node1 2>/dev/null
```

Verify that the old GRE tunnels are no longer active:

```bash
ip tunnel show
ip -br link
ip route
```

**⚠️ NOTE:** Only remove GRE interfaces that were actually created on the respective node. Interface names can differ depending on the original configuration.

---

### **5A.5.15 Final Implementation Status**

The final inter-node network implementation is:

```text
                         Internet
                            |
                    VPS Provider Network
                            |
          +-----------------+-----------------+
          |                 |                 |
        node1             node2             node3
          |                 |                 |
         wg0               wg0               wg0
          |                 |                 |
          +-----------------+-----------------+
                 WireGuard Overlay
              Encrypted Node Network
```

**Final status:**

* ✅ GRE initially attempted
* ✅ GRE provider/network limitation identified
* ✅ WireGuard selected as replacement
* ✅ WireGuard installed on all three nodes
* ✅ `wg0` configured
* ✅ Peer configuration completed
* ✅ Node1 ↔ Node2 connectivity verified
* ✅ Node1 ↔ Node3 connectivity verified
* ✅ Node2 ↔ Node3 connectivity verified
* ✅ WireGuard handshake verified
* ✅ WireGuard configured for automatic startup
* ✅ Reboot persistence verified
* ✅ Old GRE configuration/tunnels cleaned up

**Production network decision:**

> Due to GRE not being usable through the VPS provider network, the Proxmox cluster uses **WireGuard as the final encrypted private overlay** between node1, node2 and node3.



# Also we have tried for the GRE  but due to provider  network limitation cant able to do for reference  attache beloe




# **Section 6 — Accessing the Proxmox Web UI**

## **6.1  Web UI URLs**

| Node | URL | Login User | Password |
| ----- | ----- | ----- | ----- |
| node1 (Master) | https://192.168.10.11:8006 | root | (set during install) |
| node2 | https://192.168.10.12:8006 | root | (set during install) |
| node3 | https://192.168.10.13:8006 | root | (set during install) |

## **6.2  Login Steps**

| 1 | Open Browser Open Chrome, Firefox or Edge |
| :---: | :---- |

| 2 | Navigate to URL Type: https://192.168.10.11:8006  (always use https://) |
| :---: | :---- |

| 3 | Accept SSL Warning Click Advanced → Proceed to site (self-signed certificate is normal) |
| :---: | :---- |

| 4 | Login Screen Username: root  |  Password: your root password  |  Realm: Linux PAM  |  Click Login |
| :---: | :---- |

| 5 | Subscription Notice A 'No valid subscription' popup may appear — click OK to dismiss (this is normal for free version) |
| :---: | :---- |

| 💡  TIP:   The SSL certificate warning is normal for self-signed certificates. It is safe to proceed. |
| :---- |

## **6.3  Proxmox Dashboard Overview**

| Panel | Location | Purpose |
| ----- | ----- | ----- |
| Datacenter | Left sidebar — top | Cluster-wide settings and overview |
| Node (node1/2/3) | Left sidebar — under Datacenter | Individual node management |
| Summary | Right panel — Summary tab | CPU, RAM, storage usage |
| Shell | Right panel — Shell tab | Direct terminal access to node |
| Network | Node → Network tab | Network interface configuration |
| Storage | Datacenter → Storage | Storage management (local, NFS, Ceph) |

# **Section 7 — Creating a 3-Node Cluster via Web UI (GUI)**

These steps are performed on node1 (the master node) only. The cluster is created first on node1, then node2 and node3 are joined.

| ⚠️  WARNING:   Before creating the cluster, ensure all 3 nodes are accessible via Web UI and can ping each other. |
| :---- |

## **7.1  Pre-Cluster Checks**

Run on ALL 3 nodes before creating cluster:

\# Check nodes can reach each other  
ping \-c 3 node2  
ping \-c 3 node3  
   
\# Check time is synced (critical)  
timedatectl  
   
\# Install chrony for time sync if not installed  
apt install chrony \-y  
systemctl enable chrony && systemctl start chrony

| ⚠️  WARNING:   Time drift between nodes causes cluster split-brain and quorum failures. Always sync time before clustering. |
| :---- |

## **7.2  Create Cluster on node1 — Web UI Steps**

| 1 | Open node1 Web UI Navigate to https://192.168.10.11:8006 and login as root |
| :---: | :---- |

| 2 | Go to Datacenter In the left sidebar, click 'Datacenter' (top item) |
| :---: | :---- |

| 3 | Open Cluster Panel In the right panel, click the 'Cluster' tab |
| :---: | :---- |

| 4 | Click Create Cluster Click the blue 'Create Cluster' button |
| :---: | :---- |

| 5 | Enter Cluster Name In the dialog box, type a cluster name — e.g. proxmox-cluster — then click Create |
| :---: | :---- |

| 6 | Wait for Completion A task log will show. Wait for 'TASK OK' to appear at the bottom |
| :---: | :---- |

| 7 | Verify Cluster Created Go back to Datacenter → Cluster tab — you should now see the cluster name and node1 listed |
| :---: | :---- |

| ℹ️  INFO:   After cluster creation, the Cluster Information panel will show a 'Join Information' section — you will need this in Section 8\. |
| :---- |

## **7.3  Get Cluster Join Information**

After the cluster is created, you need to copy the join information to add other nodes:

| 1 | Go to Datacenter → Cluster On node1 Web UI |
| :---: | :---- |

| 2 | Click 'Join Information' A dialog appears showing the cluster join token and fingerprint |
| :---: | :---- |

| 3 | Click 'Copy Information' Copy the entire join information to clipboard — you will paste this in Section 8 |
| :---: | :---- |

| 💡  TIP:   Alternatively you can get the join information via terminal on node1 with: pvecm status |
| :---- |

# **Section 8 — Joining Nodes to Cluster via Web UI (GUI)**

These steps are performed on node2 and node3 separately. Each node is joined to the cluster created on node1.

| ⚠️  WARNING:   Joining a node to a cluster will merge its configuration with the cluster. Ensure no critical VMs are running on the node before joining. |
| :---- |

## **8.1  Join node2 to Cluster**

| 1 | Open node2 Web UI Navigate to https://192.168.10.12:8006 and login as root |
| :---: | :---- |

| 2 | Go to Datacenter → Cluster Click Datacenter in left sidebar → Click Cluster tab |
| :---: | :---- |

| 3 | Click 'Join Cluster' Click the blue 'Join Cluster' button |
| :---: | :---- |

| 4 | Paste Join Information Paste the join information copied from node1 into the 'Information' field |
| :---: | :---- |

| 5 | Enter node1 root Password In the 'Password' field, enter the root password of node1 (not node2) |
| :---: | :---- |

| 6 | Verify Details Auto-Fill Peer Address (node1 IP), Fingerprint, and Cluster Name should auto-populate |
| :---: | :---- |

| 7 | Click Join Click the 'Join proxmox-cluster' button |
| :---: | :---- |

| 8 | Wait for Completion The node will sync with the cluster. Wait for 'TASK OK'. The Web UI may disconnect briefly — this is normal |
| :---: | :---- |

| 9 | Verify in node1 Web UI Go back to node1 → Datacenter → Cluster — node2 should now appear in the node list |
| :---: | :---- |

| ℹ️  INFO:   After joining, node2's Web UI will redirect to the cluster view. You can manage all nodes from any single node's Web UI. |
| :---- |

## **8.2  Join node3 to Cluster**

Repeat the exact same steps as 8.1, but using node3's Web UI (https://192.168.10.13:8006).

| 1 | Open node3 Web UI Navigate to https://192.168.10.13:8006 and login as root |
| :---: | :---- |

| 2 | Datacenter → Cluster → Join Cluster Follow same steps as Section 8.1 |
| :---: | :---- |

| 3 | Paste Join Info & Enter node1 Password Same join information from node1 |
| :---: | :---- |

| 4 | Click Join & Wait Wait for TASK OK |
| :---: | :---- |

| 5 | Verify All 3 Nodes On node1 Web UI → Datacenter → Cluster — all 3 nodes should be listed |
| :---: | :---- |

## **8.3  Verify Cluster via Terminal**

\# Run on any node  
pvecm status  
   
\# Expected output:  
\# Cluster information  
\# \------------------  
\# Name:             proxmox-cluster  
\# Config Version:   3  
\# Transport:        knet  
\# Secure auth:      on  
   
\# Membership information  
\# \----------------------  
\# Nodeid  Votes Name  
\#      1      1 node1 (local)  
\#      2      1 node2  
\#      3      1 node3

\# List all nodes  
pvecm nodes

# **Section 9 — Post-Cluster Configuration**

## **9.1  Configure Shared Storage (Optional but Recommended)**

For VM high availability (HA), shared storage is needed so VMs can migrate between nodes.

| Storage Type | Best For | Setup Complexity |
| ----- | ----- | ----- |
| Local-LVM | Single node VMs, no migration | Simple — already configured |
| NFS | Shared storage over network | Medium |
| Ceph | High availability distributed storage | Complex — requires 3+ nodes |
| GlusterFS | Distributed storage over existing nodes | Medium |

| 💡  TIP:   For a 3-node cluster, Ceph is the gold standard for HA shared storage. It can be set up directly from Proxmox Web UI under Datacenter → Ceph. |
| :---- |

## **9.2  Configure High Availability (HA)**

HA allows VMs to automatically migrate to a healthy node if one node fails:

| 1 | Go to Datacenter → HA In left sidebar under Datacenter |
| :---: | :---- |

| 2 | Click Add under HA Groups Create an HA group with all 3 nodes |
| :---: | :---- |

| 3 | Add VMs to HA In the HA Resources tab, add the VMs you want protected |
| :---: | :---- |

| 4 | Set Priority Set node priority for each VM (which node it prefers to run on) |
| :---: | :---- |

| ⚠️  WARNING:   HA requires quorum — at least 2 out of 3 nodes must be running for the cluster to remain operational. |
| :---- |

## **9.3  Configure Backup Jobs**

| 1 | Go to Datacenter → Backup In left sidebar under Datacenter |
| :---: | :---- |

| 2 | Click Add Create a new backup schedule |
| :---: | :---- |

| 3 | Set Schedule Configure: Storage, Schedule (daily/weekly), Compress, Mode (snapshot/suspend/stop) |
| :---: | :---- |

| 4 | Select VMs Choose which VMs to backup |
| :---: | :---- |

## **9.4  Update All Nodes**

\# Run on ALL 3 nodes after cluster setup  
apt update && apt upgrade \-y  
   
\# Or via Web UI: Node → Updates → Refresh → Upgrade

# **Section 9A — Post-Cluster: Corosync Network, Repository & Time Sync Setup**

This section covers three critical post-cluster tasks that must be performed on every node after the 3-node cluster is formed: (1) configuring a dedicated Corosync network interface for cluster communication, (2) switching from the enterprise repository to the free no-subscription repository, and (3) setting up Chrony for accurate time synchronisation across all nodes.

## **9A.1 Configure a Dedicated Corosync Network Interface**

By default Proxmox uses the main management interface (vmbr0) for both VM traffic and Corosync cluster heartbeat traffic. This is not ideal for production. A dedicated network interface should be used exclusively for Corosync so cluster heartbeats are isolated from VM and web UI traffic. The steps below guide you through adding a new network bridge, removing its IP from vmbr0, and pointing Corosync at the new dedicated interface.

**⚠️ WARNING:** Do NOT add a gateway to the Corosync interface. The Corosync network is a private inter-node link only. Adding a gateway will break routing on the node.

### **Step 1 — Add a New Network Bridge on Each Node (Web UI)**

Perform these steps on node1 first, then repeat identically on node2 and node3.

| 1 | Open the Node’s Network Settings In the Proxmox Web UI left sidebar, click on pve-master (node1) → System → Network. |
| :---- | :---- |
| **2** | **Create a New Linux Bridge** Click Create → Linux Bridge. Name it vmbr1. In the IP address field enter the Corosync IP for this node (see table below). Leave the Gateway field completely empty. Set the bridge port to the second physical NIC (e.g. eth1 or ens4) if available, or leave blank for a VLAN-based setup. Click OK. |
| **3** | **Remove IP from vmbr0 (Optional but Recommended)** Click on vmbr0 → Edit. Remove any secondary IP that was previously used for cluster traffic. vmbr0 should only carry the node’s public management IP. Click OK. |
| **4** | **Apply Network Configuration** Click Apply Configuration at the top of the Network page. The changes will be written to /etc/network/interfaces and applied without a full reboot. |
| **5** | **Repeat on node2 and node3** Repeat steps 1–4 on pve-worker1 (node2) and pve-worker2 (node3), using the respective Corosync IPs from the table below. |

**Corosync Interface IP Reference**

| Node | Hostname | vmbr1 Corosync IP | Gateway |
| :---- | :---- | :---- | :---- |
| node1 | pve-master.homelab.local | 172.17.17.1/24 | **None — do not set** |
| node2 | pve-worker1.homelab.local | 172.17.17.2/24 | **None — do not set** |
| node3 | pve-worker2.homelab.local | 172.17.17.3/24 | **None — do not set** |

### **Step 2 — Point Corosync to the New Interface (Terminal)**

After the vmbr1 bridge is up on all three nodes, update Corosync to use the new 172.17.17.x network for its heartbeat link. Run the following on node1:

\# Edit corosync config on node1

nano /etc/corosync/corosync.conf  
\# Find the nodelist section and update the ring0\_addr of each node  
nodelist {  
  node { ring0\_addr: 172.17.17.1; nodeid: 1; }  
  node { ring0\_addr: 172.17.17.2; nodeid: 2; }  
  node { ring0\_addr: 172.17.17.3; nodeid: 3; }

}  
\# Restart corosync on all nodes  
systemctl restart corosync pve-cluster

### **Step 3 — Verify Corosync Network Connectivity (Ping Test)**

From each node, ping the Corosync IP of the other nodes to confirm the 172.17.17.x network is reachable. A successful ping confirms the Corosync heartbeat path is working.

\# From node1 — ping node2 and node3 Corosync IPs  
ping \-c 4 172.17.17.2  
ping \-c 4 172.17.17.3  
\# From node2 — ping node1 and node3  
ping \-c 4 172.17.17.1  
ping \-c 4 172.17.17.3

**💡 TIP:** If ping fails between nodes, check that vmbr1 is up (ip link show vmbr1) and that no firewall rule is blocking ICMP on the 172.17.17.0/24 subnet. Run pvecm status to confirm Corosync sees all 3 nodes.

## **9A.2 Disable Enterprise Repository and Enable Free No-Subscription Repo**

By default Proxmox VE ships with the Enterprise repository enabled. This repo requires a paid subscription licence. Without a valid licence key, every apt update returns a 401 Unauthorised error. The fix is to disable all enterprise repositories and add the official free no-subscription repository. Perform these steps on every node.

**⚠️ WARNING:** Do not leave the enterprise repository enabled without a valid subscription key. It will cause apt update to fail with a 401 error and block all future package updates on the node.

### **Method A — Via Web UI (Recommended)**

| 1 | Open Updates & Repositories In the left sidebar click on the node (e.g. pve-master) → Updates → Repositories. |
| :---- | :---- |
| **2** | **Disable the Enterprise Repository** In the repository list find the entry for https://enterprise.proxmox.com/debian/pve (the enterprise repo). Click on it to select it and then click the Disable button. The row will grey out indicating it is disabled. Repeat for the Ceph enterprise repo if it appears in the list. |
| **3** | **Add the No-Subscription Repository** Click the Add button. A dropdown appears with repository options. Select Proxmox VE No-Subscription from the list and click Add. The free community repository will appear as an active entry in the list. |
| **4** | **Run apt update from Shell** Click on the node → Shell to open the terminal. Run apt update to confirm there are no 401 errors. Then run apt upgrade \-y to apply all pending updates. Repeat on each node. |
| **5** | **Reboot the Node** After upgrades complete, reboot the node from the Web UI: select the node → click Reboot in the top-right menu. Wait for the node to come back online and verify it rejoins the cluster (Datacenter → Cluster). |

### **Method B — Via Terminal (All Nodes)**

If you prefer the command line or are connecting over SSH, run the following on each node individually:

\# Disable enterprise repo  
echo "\# disabled" \> /etc/apt/sources.list.d/pve-enterprise.list  
\# Add free no-subscription repo  
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \\  
  \> /etc/apt/sources.list.d/pve-no-subscription.list  
\# Update and upgrade  
apt update && apt upgrade \-y  
reboot

## **9A.3 Configure Chrony for Cluster Time Synchronisation**

Accurate time synchronisation is critical for a Proxmox cluster. Clock drift between nodes can cause Corosync quorum failures, split-brain events, and incorrect timestamps in logs and backups. Chrony is the recommended NTP implementation for Proxmox VE on Debian Bookworm.

**⚠️ WARNING:** Time drift greater than 2 seconds between cluster nodes will cause Corosync to mark nodes as unreachable and trigger a split-brain event. Always verify time sync is working before adding nodes to a cluster.

### **Install and Configure Chrony (Run on All 3 Nodes)**

\# Install chrony  
apt install chrony \-y  
\# Enable and start chrony service  
systemctl enable chrony && systemctl start chrony  
\# Verify chrony is syncing  
chronyc tracking  
chronyc sources \-v  
\# Check system clock status  
timedatectl

**ℹ️ INFO:** chronyc tracking should show System time offset close to 0.000000 seconds and NTP synchronized: yes in timedatectl output. A \* symbol next to a source in chronyc sources \-v means that source is currently selected as the primary time reference.

**📝 PENDING — To Be Expanded:** Advanced Chrony configuration (designating node1 as an internal NTP server for node2 and node3, stratum settings, and offline fallback behaviour) will be documented here once testing is complete.

# **Section 10 — Creating VMs & Containers in Proxmox**

## **10.1  Upload an ISO for VM Installation**

| 1 | Go to Node → local storage In left sidebar: node1 → local (node1) |
| :---: | :---- |

| 2 | Click ISO Images tab Then click Upload or Download from URL |
| :---: | :---- |

| 3 | Upload or paste URL Upload ISO file from PC, or paste a direct download URL |
| :---: | :---- |

| 4 | Wait for upload Progress bar shows upload status |
| :---: | :---- |

## **10.2  Create a Virtual Machine (VM)**

| 1 | Click 'Create VM' Blue button at top right of Web UI |
| :---: | :---- |

| 2 | General Tab Set Node (node1/2/3), VM ID (auto), Name — Click Next |
| :---: | :---- |

| 3 | OS Tab Select ISO image, Guest OS type (Linux/Windows), Version — Click Next |
| :---: | :---- |

| 4 | System Tab Keep defaults (SeaBIOS, IDE, VirtIO) — Click Next |
| :---: | :---- |

| 5 | Disks Tab Set disk size (e.g. 32GB), Storage (local-lvm), Format (raw) — Click Next |
| :---: | :---- |

| 6 | CPU Tab Set CPU cores (e.g. 2), Type: host — Click Next |
| :---: | :---- |

| 7 | Memory Tab Set RAM in MB (e.g. 2048 \= 2GB) — Click Next |
| :---: | :---- |

| 8 | Network Tab Bridge: vmbr0, Model: VirtIO — Click Next |
| :---: | :---- |

| 9 | Confirm Tab Review all settings — Click Finish |
| :---: | :---- |

| 10 | Start VM Select VM in left sidebar → Click Start → Click Console to see the VM screen |
| :---: | :---- |

## **10.3  Create a Linux Container (LXC)**

LXC containers are lighter than VMs and share the host kernel — ideal for Linux workloads.

| 1 | Download CT Template Node → local → CT Templates → Templates → Select and Download (e.g. debian-12) |
| :---: | :---- |

| 2 | Click 'Create CT' Blue button at top right |
| :---: | :---- |

| 3 | General Set Node, CT ID, Hostname, Password — Click Next |
| :---: | :---- |

| 4 | Template Select downloaded template — Click Next |
| :---: | :---- |

| 5 | Disks Set disk size — Click Next |
| :---: | :---- |

| 6 | CPU Set CPU cores — Click Next |
| :---: | :---- |

| 7 | Memory Set RAM and Swap — Click Next |
| :---: | :---- |

| 8 | Network Set IP (DHCP or static), Bridge — Click Next |
| :---: | :---- |

| 9 | DNS Keep default or set custom DNS — Click Next |
| :---: | :---- |

| 10 | Finish Review → Click Finish → Start container |
| :---: | :---- |

# **Section 11 — Live Progress Checklist**

## **Phase 1 — OS & Base Setup**

✅  Debian Bookworm installed on all 3 VPS

✅  Hostnames set: node1, node2, node3

✅  /etc/hosts configured on all 3 nodes

✅  Enterprise Proxmox repo removed

✅  Free no-subscription repo added

✅  Proxmox GPG key added

✅  apt update && upgrade completed on all nodes

## **Phase 2 — Proxmox VE Installation**

✅  proxmox-ve installed on node1

✅  proxmox-ve installed on node2

✅  proxmox-ve installed on node3

✅  All nodes rebooted

✅  pveproxy service verified active on node1

## **Phase 3 — Firewall & Port Access**

✅  iptables port 8006 opened on all nodes

✅  iptables rules saved (iptables-persistent)

🔄  *VPS provider contacted to open port 8006 at network level*

⬜  Web UI accessible on node1

⬜  Web UI accessible on node2

⬜  Web UI accessible on node3

## **Phase 4 — Cluster Setup**

⬜  Time sync (chrony) installed on all nodes

⬜  All nodes can ping each other by hostname

⬜  Cluster created on node1 via Web UI

⬜  node2 joined cluster via Web UI

⬜  node3 joined cluster via Web UI

⬜  All 3 nodes visible in Datacenter view

⬜  Cluster status verified: pvecm status

## **Phase 5 — Post Configuration**

⬜  Shared storage configured

⬜  HA configured for critical VMs

⬜  Backup schedule configured

⬜  First VM or container created

# **Section 12 — Quick Reference & Command Glossary**

## **12.1  Essential Proxmox Commands**

| Command | Purpose |
| ----- | ----- |
| pvecm status | Show cluster status and quorum |
| pvecm nodes | List all nodes in cluster |
| pvecm create \<name\> | Create a new cluster (node1 only) |
| pvecm add \<IP\> | Join an existing cluster |
| pvecm expected 1 | Force quorum with 1 node (emergency only) |
| pveversion | Show Proxmox VE version |
| pveversion \-v | Show detailed version info |
| systemctl status pveproxy | Check Web UI service status |
| systemctl restart pveproxy | Restart Web UI service |
| systemctl status pvedaemon | Check main Proxmox daemon |
| qm list | List all VMs |
| qm start \<vmid\> | Start a VM |
| qm stop \<vmid\> | Stop a VM |
| qm shutdown \<vmid\> | Gracefully shutdown a VM |
| pct list | List all containers |
| pct start \<ctid\> | Start a container |
| pct stop \<ctid\> | Stop a container |
| pvesm status | Show storage status |
| pvesh get /nodes | API: list nodes |

## **12.2  Firewall & Network Commands**

| Command | Purpose |
| ----- | ----- |
| ss \-tlnp | grep 8006 | Check if port 8006 is listening |
| iptables \-L \-n | List all firewall rules |
| iptables \-A INPUT \-p tcp \--dport 8006 \-j ACCEPT | Open port 8006 |
| netfilter-persistent save | Save iptables rules permanently |
| ping node2 | Test connectivity to node2 |
| ip a | Show all network interfaces and IPs |
| ip route | Show routing table |

## **12.3  Repository Reference**

| Repo Type | URL | Requires |
| ----- | ----- | ----- |
| Enterprise (paid) | https://enterprise.proxmox.com/debian/pve | Paid subscription key |
| No-Subscription (free) | http://download.proxmox.com/debian/pve | Nothing — free |
| Debian Bookworm | https://deb.debian.org/debian bookworm | Nothing — default |

## **12.4  Web UI Quick Access**

| Node | URL | Role |
| ----- | ----- | ----- |
| node1 | https://192.168.10.11:8006 | Master — Create cluster here |
| node2 | https://192.168.10.12:8006 | Worker — Join cluster here |
| node3 | https://192.168.10.13:8006 | Worker — Join cluster here |

# **Section 13 — Restricting Web UI Access to Trusted Networks Only**

By default, the Proxmox Web UI (port 8006\) is publicly accessible to anyone on the internet. This section covers how to restrict access to only your trusted networks — office, home, and VPN — using the Proxmox built-in firewall.

| ⚠️  WARNING:   Never skip this step on a production cluster. A publicly exposed Proxmox Web UI is a critical security risk. Always restrict access to known IPs only. |
| :---- |

## **13.1  Overview — How Proxmox Firewall Works**

The Proxmox firewall operates at two levels — Datacenter level (applies to all nodes) and Node level (per-node). Both must be enabled for the rules to take effect.

| Level | Location | Applies To |
| ----- | ----- | ----- |
| Datacenter Firewall | Datacenter → Firewall | All nodes in cluster — global rules |
| Node Firewall | node1/2/3 → Firewall → Options | Individual node — per-node rules |
| VM/CT Firewall | VM → Firewall | Individual VM or container |

| ℹ️  INFO:   For Web UI access restriction, configuring rules at the Datacenter level and enabling firewall on each node is sufficient. |
| :---- |

## **13.2  Step 1 — Create an IPSet for Trusted Networks**

An IPSet is a named group of IP addresses/ranges. By using an IPSet, you can manage all your trusted IPs in one place and reference them in multiple rules.

| 1 | Navigate to IPSet Datacenter → Firewall → IPSet (left sidebar) |
| :---: | :---- |

| 2 | Create New IPSet Click Create button → Enter name: allowed-networks → Comment: My Trusted Networks → Click OK |
| :---: | :---- |

| 3 | Select the IPSet Click allowed-networks in the left panel to select it — right panel shows IP/CIDR list |
| :---: | :---- |

| 4 | Add Office IP Click Add (right side) → IP/CIDR: \<office-ip\>/32 → Comment: Office → OK |
| :---: | :---- |

| 5 | Add Home IP Click Add → IP/CIDR: \<home-ip\>/32 → Comment: Home → OK |
| :---: | :---- |

| 6 | Add VPN Network Click Add → IP/CIDR: \<vpn-subnet\>/24 → Comment: VPN → OK |
| :---: | :---- |

| 💡  TIP:   Use /32 for a single static IP (e.g. 130.225.0.14/32). Use /24 for a full subnet range (e.g. 10.8.0.0/24). Check your current IP at whatismyip.com before proceeding. |
| :---- |

| ⚠️  WARNING:   CRITICAL: Before enabling the firewall, verify your CURRENT IP is in the IPSet. Run whatismyip.com and confirm the IP appears in allowed-networks. If not — add it first or you will be locked out immediately. |
| :---- |

## **13.3  Step 2 — Add Firewall Rules**

Navigate to Datacenter → Firewall → Click Add for each rule below. Rules are processed top to bottom — order matters.

| Rule \# | Direction | Action | Protocol | Source | Dest. Port | Comment |
| ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| 0 | in | ACCEPT | tcp | \+allowed-networks | 22 | Allow SSH from trusted networks |
| 1 | in | ACCEPT | tcp | \+allowed-networks | 8006 | Allow Web UI from trusted networks |
| 2 | in | DROP | tcp | (empty — any) | 8006 | Block all public Web UI access |

| ⚠️  WARNING:   Rule order is critical. ACCEPT rules (0 and 1\) must appear BEFORE the DROP rule (2). Proxmox processes rules top-to-bottom — if DROP comes first, even trusted IPs will be blocked. |
| :---- |

**How to Add Each Rule in Web UI:**

Datacenter → Firewall → Click Add button  
   
Rule 0 — SSH Allow:  
  Direction:   in  
  Action:      ACCEPT  
  Protocol:    tcp  
  Source:      \+allowed-networks  
  Dest. Port:  22  
  Comment:     Allow SSH trusted only  
  → Click Add  
   
Rule 1 — Web UI Allow:  
  Direction:   in  
  Action:      ACCEPT  
  Protocol:    tcp  
  Source:      \+allowed-networks  
  Dest. Port:  8006  
  Comment:     Allow WebUI trusted only  
  → Click Add  
   
Rule 2 — Block Public:  
  Direction:   in  
  Action:      DROP  
  Protocol:    tcp  
  Source:      (leave empty)  
  Dest. Port:  8006  
  Comment:     Block public WebUI access  
  → Click Add

## **13.4  Step 3 — Enable All Rules**

After adding the 3 rules, they will appear in the list but with unchecked checkboxes in the 'On' column — meaning they are disabled by default.

| 1 | Check all rule checkboxes In Datacenter → Firewall, click the checkbox (On column) next to Rule 0, Rule 1, and Rule 2 to enable each one |
| :---: | :---- |

| 2 | Verify checkboxes are filled All 3 rules should now show ☑ in the On column |
| :---: | :---- |

| 💡  TIP:   Rules must be individually enabled via the checkbox AND the firewall must be turned ON in Options. Both steps are required. |
| :---- |

## **13.5  Step 4 — Enable Firewall in Options**

Enabling rules alone is not enough — the Proxmox firewall service itself must also be turned ON.

| 1 | Datacenter Firewall Datacenter → Firewall → Options → Firewall: YES → Click OK |
| :---: | :---- |

| 2 | node1 Firewall node1 → Firewall → Options → Firewall: YES → Click OK |
| :---: | :---- |

| 3 | node2 Firewall node2 → Firewall → Options → Firewall: YES → Click OK |
| :---: | :---- |

| 4 | node3 Firewall node3 → Firewall → Options → Firewall: YES → Click OK |
| :---: | :---- |

| ⚠️  WARNING:   Enable node-level firewall ONLY after Datacenter firewall is enabled and rules are verified. Enable one node at a time and test between each. |
| :---- |

## **13.6  Step 5 — Test the Firewall**

Always test immediately after enabling — keep your current browser session open in case you need to roll back.

| Test | Method | Expected Result |
| ----- | ----- | ----- |
| Trusted network access | Open https://192.168.10.11:8006 from your current browser | ✅ Loads normally — you are in allowed-networks |
| Public access blocked | Open https://192.168.10.11:8006 from mobile hotspot (different network) | ⛔ Times out — public access blocked |
| SSH still works | SSH from trusted IP: ssh root@192.168.10.11 | ✅ SSH connects — Rule 0 allows it |
| SSH from public blocked | Try SSH from mobile hotspot | ⛔ Connection refused — blocked correctly |

| 💡  TIP:   Always test from a mobile hotspot or ask someone on a different network to test. This confirms public access is actually blocked and not just appearing to work. |
| :---- |

## **13.7  Managing Trusted Networks — Add/Remove IPs**

To add or remove IPs later without touching the firewall rules, simply update the IPSet:

\# Via Web UI:  
Datacenter → Firewall → IPSet → allowed-networks  
→ Add new IP  (click Add on right panel)  
→ Remove IP   (select IP → click Remove)  
   
\# Via terminal on node1:  
nano /etc/pve/firewall/cluster.fw  
   
\# IPSet entries look like this in the file:  
\[IPSET allowed-networks\]  
130.225.0.14/32  \# Home  
203.x.x.x/32    \# Office  
10.8.0.0/24     \# VPN

## **13.8  Emergency Recovery — Locked Out of Web UI**

If you accidentally lock yourself out, you can disable the firewall via SSH or console:

| ⚠️  WARNING:   If locked out of Web UI — do NOT panic. SSH on port 22 may still work if you added the SSH ACCEPT rule. If SSH is also blocked, use your VPS provider's console/VNC. |
| :---- |

\# Option 1 — Disable via SSH (if SSH still works):  
ssh root@192.168.10.11  
   
\# Disable Proxmox firewall immediately:  
pvesh set /nodes/node1/firewall/options \--enable 0  
   
\# Or edit the cluster firewall config directly:  
nano /etc/pve/firewall/cluster.fw  
\# Change:  enable: 1  
\# To:      enable: 0  
\# Save: Ctrl+X → Y → Enter  
   
\# Option 2 — Via VPS provider console (if SSH also blocked):  
\# Login to provider panel → Open VNC/noVNC console  
\# Login as root → run commands above  
   
\# Option 3 — Flush all iptables rules (nuclear option):  
iptables \-F  
iptables \-X  
iptables \-P INPUT ACCEPT

## **13.9  Firewall Configuration Summary**

| Component | Setting | Value |
| ----- | ----- | ----- |
| Datacenter Firewall | Options → Firewall | Enabled: YES |
| Node Firewall (all 3\) | node → Firewall → Options | Enabled: YES |
| Input Policy | Datacenter → Firewall → Options | ACCEPT (default — only specific ports dropped) |
| Trusted Networks IPSet | allowed-networks | Office /32 \+ Home /32 \+ VPN /24 |
| Rule 0 | ACCEPT SSH port 22 | Source: \+allowed-networks only |
| Rule 1 | ACCEPT port 8006 | Source: \+allowed-networks only |
| Rule 2 | DROP port 8006 | Source: any (blocks all public access) |

# **Section 14 — Automating VM Provisioning with HostBill**

HostBill is a billing and automation platform that integrates directly with Proxmox VE via API. When a customer places an order, HostBill automatically creates a VM on your cluster, assigns an IP, injects credentials, and emails the customer with zero manual intervention.

## **14.1  How the Automation Flow Works**

Customer places VPS order on HostBill  
         |  
HostBill calls Proxmox API on port 8006  
         |  
VM automatically created from template on cluster  
         |  
IP assigned from IP pool  
         |  
Cloud-Init injects: password, SSH key, hostname  
         |  
Customer receives login credentials via email  
         |  
Customer manages VM from HostBill client area

## **14.2  What Gets Automated**

| Action | Automated | Trigger |
| ----- | ----- | ----- |
| VM creation | YES | New order placed |
| OS installation via Cloud-Init template | YES | Order activation |
| IP address assignment from pool | YES | On activation |
| SSH key and password injection | YES | Via Cloud-Init on first boot |
| Welcome email with credentials | YES | After VM creation complete |
| Start / Stop / Reboot from client area | YES | Customer action |
| Suspend VM on non-payment | YES | Automatic billing trigger |
| Terminate VM on cancellation | YES | Automatic billing trigger |
| CPU and RAM resize on plan upgrade | YES | Customer upgrades product |
| Backup and snapshot management | YES | Customer configures in client area |
| Firewall rule management | YES | Via HostBill client area |
| Bandwidth and disk usage monitoring | YES | Real-time in client area |

## **14.3  Pre-Setup Requirements**

| Requirement | Details |
| ----- | ----- |
| HostBill installation | Active HostBill with Proxmox module purchased or included in edition |
| Port 8006 access | Must be open from HostBill server IP to Proxmox nodes |
| Cloud-Init template | Required on Proxmox for OS injection and credential automation |
| IP Pool | Available IP range configured in HostBill IP Manager |
| HostBill server IP in IPSet | Add to allowed-networks IPSet in Proxmox firewall (Section 13\) |

| ⚠️  WARNING:   The HostBill server IP must be added to your Proxmox firewall allowed-networks IPSet. Without this, all HostBill API calls to port 8006 will be blocked and automation will fail. |
| :---- |

## **14.4  Step 1 — Install HostBill Proxmox Module**

| 1 | Login to HostBill Admin Navigate to https://\<your-hostbill-domain\>/admin |
| :---: | :---- |

| 2 | Download Module HostBill Client Portal → Downloads → Proxmox Module → Download ZIP |
| :---: | :---- |

| 3 | Upload Module Extract ZIP → Upload all files to HostBill root directory on server |
| :---: | :---- |

| 4 | Activate Module HostBill Admin → Settings → Modules → Hosting Modules → Find Proxmox → Click Activate |
| :---: | :---- |

## **14.5  Step 2 — Create Proxmox API User for HostBill**

Run on node1 to create a dedicated API user with the required permissions:

\# Create dedicated API user  
pveum user add hostbill@pve \--comment 'HostBill Automation User'  
   
\# Set user password  
pveum passwd hostbill@pve  
   
\# Grant Administrator role  
pveum aclmod / \-user hostbill@pve \-role Administrator  
   
\# RECOMMENDED: Use API Token instead of password  
pveum user token add hostbill@pve hostbill-token \--privsep 0  
pveum aclmod / \-token 'hostbill@pve\!hostbill-token' \-role Administrator  
   
\# Verify user created successfully  
pveum user list

| 💡  TIP:   Always use API tokens over passwords in production. Tokens can be revoked individually without changing the user password, and each integration can have its own token. |
| :---- |

## **14.6  Step 3 — Connect HostBill to Proxmox App**

| 1 | Open Apps HostBill Admin → Settings → Apps → Add New App |
| :---: | :---- |

| 2 | Select Type App Type: Proxmox VE → Click Next |
| :---: | :---- |

| 3 | Enter Details Fill in connection settings: |
| :---: | :---- |

App Name:   My Proxmox Cluster  
Hostname:   192.168.10.11  
Port:       8006  
Username:   hostbill@pve  
Password:   your-hostbill-user-password  
            OR use API Token:  
Token ID:   hostbill@pve\!hostbill-token  
Token Val:  \<generated-token-value\>

| 4 | Test Connection Click Test Connection — expect green success response |
| :---: | :---- |

| 5 | Save App Click Save |
| :---: | :---- |

| ⚠️  WARNING:   If Test Connection fails: (1) Verify HostBill server IP is in Proxmox allowed-networks IPSet. (2) Confirm port 8006 is not blocked by provider firewall for HostBill server IP. |
| :---- |

## **14.7  Step 4 — Create Cloud-Init VM Templates**

Cloud-Init templates are required for automation. They allow Proxmox to inject login credentials, SSH keys, and network config into each new customer VM at first boot.

**Debian 12 Template — VM ID 9000**

\# Download Debian 12 cloud image on node1  
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2  
   
\# Create VM 9000  
qm create 9000 \--name debian12-template \--memory 2048 \--cores 2 \--net0 virtio,bridge=vmbr0  
   
\# Import cloud disk into local-lvm storage  
qm importdisk 9000 debian-12-generic-amd64.qcow2 local-lvm  
   
\# Configure VM hardware  
qm set 9000 \--scsihw virtio-scsi-pci \--scsi0 local-lvm:vm-9000-disk-0  
qm set 9000 \--ide2 local-lvm:cloudinit  
qm set 9000 \--boot c \--bootdisk scsi0  
qm set 9000 \--serial0 socket \--vga serial0  
qm set 9000 \--agent enabled=1  
   
\# Convert to template — cannot be reversed  
qm template 9000  
qm list

**Ubuntu 22.04 Template — VM ID 9001**

wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img  
qm create 9001 \--name ubuntu2204-template \--memory 2048 \--cores 2 \--net0 virtio,bridge=vmbr0  
qm importdisk 9001 jammy-server-cloudimg-amd64.img local-lvm  
qm set 9001 \--scsihw virtio-scsi-pci \--scsi0 local-lvm:vm-9001-disk-0  
qm set 9001 \--ide2 local-lvm:cloudinit  
qm set 9001 \--boot c \--bootdisk scsi0  
qm set 9001 \--serial0 socket \--vga serial0  
qm set 9001 \--agent enabled=1  
qm template 9001

| 💡  TIP:   Create one template per OS offering. Each template needs a unique VM ID. HostBill references these IDs when provisioning new customer VMs automatically. |
| :---- |

## **14.8  Step 5 — Configure IP Pool in HostBill**

| 1 | Open IP Manager HostBill Admin → Settings → IP Manager → Add IP Range |
| :---: | :---- |

| 2 | Fill IP Range Enter the IP range available for customer VMs: |
| :---: | :---- |

IP Range Start:   192.168.10.50  
IP Range End:     192.168.10.100  
Subnet Mask:      255.255.255.0  
Gateway:          192.168.10.1  
DNS Primary:      8.8.8.8  
DNS Secondary:    8.8.4.4  
Assigned App:     My Proxmox Cluster

| 3 | Save Click Save — IPs now available for auto-assignment |
| :---: | :---- |

| ⚠️  WARNING:   Only add IPs that are genuinely allocated to your servers by your provider. Using unallocated IPs will cause customer VMs to have broken network connectivity. |
| :---- |

## **14.9  Step 6 — Create VPS Products in HostBill**

| 1 | Add Product HostBill Admin → Products and Services → Add Product → VPS / Cloud Hosting |
| :---: | :---- |

| 2 | Module Setup Module: Proxmox | Server: My Proxmox Cluster |
| :---: | :---- |

| 3 | VM Configuration Set VM parameters for this plan: |
| :---: | :---- |

| Field | Example Value | Notes |
| ----- | ----- | ----- |
| Node | node1 | Which cluster node to deploy VMs on |
| Template VM ID | 9000 | Cloud-Init template created in Step 14.7 |
| CPU Cores | 2 | vCPUs assigned to customer VM |
| RAM (MB) | 2048 | Memory in megabytes |
| Disk Size (GB) | 20 | Root disk size for customer VM |
| Storage Pool | local-lvm | Proxmox storage target for VM disks |
| Network Bridge | vmbr0 | Virtual network bridge for VM |
| IP Assignment | From Pool | Auto-assigned from HostBill IP pool |
| OS Type | Linux | Linux or Windows |

| 4 | Set Pricing Pricing tab → Set monthly and yearly prices |
| :---: | :---- |

| 5 | Publish Settings → Visible: Yes → Save |
| :---: | :---- |

## **14.10  Multi-Node VM Distribution**

To spread customer VMs across all 3 nodes, create products per node or configure auto-assignment:

| Product Name | Node | Template | Use Case |
| ----- | ----- | ----- | ----- |
| VPS Starter \- Node1 | node1 | 9000 Debian 12 | Entry Linux VPS |
| VPS Starter \- Node2 | node2 | 9000 Debian 12 | Entry Linux VPS |
| VPS Starter \- Node3 | node3 | 9000 Debian 12 | Entry Linux VPS |
| VPS Ubuntu \- Node1 | node1 | 9001 Ubuntu 22.04 | Ubuntu plans |
| VPS Performance | node1 | 9001 Ubuntu | High CPU/RAM plans |

| ℹ️  INFO:   HostBill can auto-select the least loaded node when multiple servers are in the same app group, distributing new VMs evenly across all 3 nodes automatically. |
| :---- |

## **14.11  Step 7 — Test Full Automation End-to-End**

| 1 | Place Test Order HostBill Admin → Orders → Add Order → Select VPS product → Use test account |
| :---: | :---- |

| 2 | Activate Order Orders → Find the test order → Click Activate |
| :---: | :---- |

| 3 | Watch Proxmox Proxmox Web UI → Datacenter → watch new VM appear under the node |
| :---: | :---- |

| 4 | Verify IP Assigned New VM should show running status with IP from pool |
| :---: | :---- |

| 5 | Check Welcome Email Test account inbox should have IP, username, and password |
| :---: | :---- |

| 6 | Test SSH Login SSH to new VM with emailed credentials — confirm access works |
| :---: | :---- |

| 7 | Test Client Controls HostBill client area → Start/Stop/Reboot → verify Proxmox responds |
| :---: | :---- |

## **14.12  Common Errors and Solutions**

| 🔴  ERROR Test Connection Failed — Connection refused on 192.168.10.11:8006 Root Cause: HostBill server IP not in Proxmox firewall allowed-networks IPSet, or provider firewall blocking port 8006 from HostBill server IP. |
| :---- |
| **✅  SOLUTION** Add HostBill server IP to Proxmox allowed-networks IPSet via Web UI or terminal. \# Via Proxmox Web UI: Datacenter → Firewall → IPSet → allowed-networks → Add → IP/CIDR: \<hostbill-ip\>/32  Comment: HostBill Server   \# Via terminal on node1: nano /etc/pve/firewall/cluster.fw \# Under \[IPSET allowed-networks\] add: \# \<hostbill-server-ip\>/32  \# HostBill Server |

| 🔴  ERROR VM created but no IP assigned to customer VM Root Cause: IP pool in HostBill is empty, exhausted, or not linked to the correct Proxmox app. |
| :---- |
| **✅  SOLUTION** Verify IP pool status and linkage in HostBill IP Manager. Add more IPs if pool is exhausted. HostBill Admin → Settings → IP Manager → Verify pool is linked to My Proxmox Cluster app → Check Available IP count → Add more IPs if count is 0 |

| 🔴  ERROR VM created but customer cannot SSH — Cloud-Init not running Root Cause: Cloud-Init package not installed in the template OS before converting to template, or Cloud-Init drive (ide2) not attached. |
| :---- |
| **✅  SOLUTION** Rebuild the template with Cloud-Init installed. Verify ide2 cloudinit drive attached before converting VM to template. \# Verify Cloud-Init drive on template: qm config 9000 | grep ide2 \# Expected: ide2: local-lvm:vm-9000-cloudinit   \# Re-add if missing: qm set 9000 \--ide2 local-lvm:cloudinit   \# Verify cloud-init package inside template (boot VM first): apt install cloud-init \-y     \# Debian/Ubuntu yum install cloud-init \-y     \# CentOS/RHEL   \# Re-convert to template: qm template 9000 |

## **14.13  Security Best Practices for HostBill Integration**

| Practice | How To Implement |
| ----- | ----- |
| Dedicated API user only | Use hostbill@pve — never use root account for API calls |
| API token over password | pveum user token add — tokens are individually revocable |
| Restrict HostBill IP only | Add only the exact HostBill server IP — not a broad /24 subnet |
| Rotate API tokens | Regenerate tokens every 90 days via pveum user token add |
| Monitor access logs | tail \-f /var/log/pveproxy/access.log — watch for unexpected calls |
| Separate management VLAN | Run HostBill on dedicated network separate from customer VM traffic |

# **Section 15 — Automating VM Provisioning with FOSSBilling (Free)**

FOSSBilling is a 100% free and open source billing and automation platform that integrates with Proxmox VE. It is the recommended free alternative to HostBill for small to medium hosting setups. When a customer places an order, FOSSBilling automatically creates a VM on your Proxmox cluster, assigns an IP, and sends credentials.

| Software | Cost | Proxmox Support | Best For |
| ----- | ----- | ----- | ----- |
| FOSSBilling | 100% Free & Open Source | Yes — via module | Small to medium hosting setups |
| WHMCS | $15.95/month | Yes | Large commercial hosting |
| HostBill | Paid license | Yes | Enterprise setups |
| WISECP | Free tier (limited) | Yes | Very small setups only |

| 💡  TIP:   FOSSBilling is the recommended free billing solution for your 3-node Proxmox cluster. It is actively maintained, fully open source, and has a native Proxmox module for VM automation. |
| :---- |

## **15.1  Recommended Architecture**

FOSSBilling Server (deployed as VM on node1)  
         |  
   Proxmox API on port 8006  
         |  
  \+-------+-------+  
node1   node2   node3  
(your existing 3-node cluster)

| ℹ️  INFO:   Deploying FOSSBilling as a VM on your own Proxmox cluster is the cleanest approach. It uses your existing infrastructure and keeps everything in one place. |
| :---- |

## **15.2  What FOSSBilling Automates on Proxmox**

| Task | Automated | Trigger |
| ----- | ----- | ----- |
| VM creation on new order | YES | Customer places order |
| OS deployment via template | YES | Order activation |
| IP assignment | YES | From IP pool on activation |
| Welcome email with credentials | YES | After VM creation |
| VM suspension on non-payment | YES | Automatic billing trigger |
| VM termination on cancellation | YES | Automatic billing trigger |
| Client VM start/stop/reboot | YES | Customer clicks in client portal |
| Resource limit enforcement | YES | CPU, RAM, disk per plan |
| Invoice generation | YES | Monthly/annual billing cycle |
| Payment processing | YES | PayPal, Stripe, crypto gateways |
| Support ticket system | YES | Client submits ticket |
| Email notifications | YES | Order, invoice, suspension alerts |
| Multi-node support | YES | Distribute VMs across nodes |

## **15.3  Pre-Installation Requirements**

| Requirement | Details |
| ----- | ----- |
| VM for FOSSBilling | Deploy a Debian 12 or Ubuntu 22.04 VM on node1 (2 vCPU, 2GB RAM, 20GB disk) |
| PHP | PHP 8.1 or higher with required extensions |
| Web Server | Apache2 or Nginx |
| Database | MySQL 8.0 or MariaDB 10.6+ |
| Proxmox API access | Port 8006 accessible from FOSSBilling VM IP |
| Domain or IP | FOSSBilling needs a public IP or domain for client portal |

| ⚠️  WARNING:   Before installing FOSSBilling, add the FOSSBilling VM IP to your Proxmox firewall allowed-networks IPSet (Section 13). Otherwise FOSSBilling API calls to port 8006 will be blocked. |
| :---- |

## **15.4  Step 1 — Create FOSSBilling VM on Proxmox**

Create a dedicated VM on node1 to host FOSSBilling:

| 1 | Login to Proxmox https://192.168.10.11:8006 → root |
| :---: | :---- |

| 2 | Create VM Click Create VM → Name: fossbilling → OS: Debian 12 ISO → 2 vCPU, 2GB RAM, 20GB disk |
| :---: | :---- |

| 3 | Install Debian Boot VM → Install Debian 12 minimal → Set static IP → Note the VM IP |
| :---: | :---- |

| 4 | Add VM IP to Firewall Datacenter → Firewall → IPSet → allowed-networks → Add → FOSSBilling VM IP/32 |
| :---: | :---- |

## **15.5  Step 2 — Install LAMP Stack on FOSSBilling VM**

SSH into the FOSSBilling VM and install the required software stack:

\# SSH into FOSSBilling VM  
ssh root@\<fossbilling-vm-ip\>  
   
\# Update system  
apt update && apt upgrade \-y  
   
\# Install Apache, MySQL, PHP 8.1 and required extensions  
apt install apache2 mysql-server php8.1 php8.1-mysql php8.1-curl php8.1-xml php8.1-mbstring php8.1-intl php8.1-zip php8.1-gd php8.1-bcmath unzip wget \-y  
   
\# Enable Apache modules  
a2enmod rewrite  
systemctl restart apache2  
   
\# Verify PHP version  
php \-v  
\# Expected: PHP 8.1.x

## **15.6  Step 3 — Configure MySQL Database**

\# Secure MySQL installation  
mysql\_secure\_installation  
\# Follow prompts: set root password, remove test db, disable remote root login  
   
\# Login to MySQL  
mysql \-u root \-p  
   
\# Create FOSSBilling database and user  
CREATE DATABASE fossbilling CHARACTER SET utf8mb4 COLLATE utf8mb4\_unicode\_ci;  
CREATE USER 'fbuser'@'localhost' IDENTIFIED BY 'StrongPassword123\!';  
GRANT ALL PRIVILEGES ON fossbilling.\* TO 'fbuser'@'localhost';  
FLUSH PRIVILEGES;  
EXIT;

| 💡  TIP:   Use a strong database password. Replace StrongPassword123\! with your own secure password and save it somewhere safe. |
| :---- |

## **15.7  Step 4 — Download and Install FOSSBilling**

\# Navigate to web root  
cd /var/www/html  
   
\# Remove default Apache page  
rm \-f index.html  
   
\# Download latest FOSSBilling stable release  
wget https://fossbilling.org/downloads/stable \-O fossbilling.zip  
   
\# Extract files  
unzip fossbilling.zip  
   
\# Set correct file permissions  
chown \-R www-data:www-data /var/www/html  
chmod \-R 755 /var/www/html  
   
\# Create writable directories  
mkdir \-p /var/www/html/data/cache  
mkdir \-p /var/www/html/data/log  
chmod \-R 777 /var/www/html/data

## **15.8  Step 5 — Configure Apache Virtual Host**

\# Create virtual host config  
nano /etc/apache2/sites-available/fossbilling.conf  
   
\# Paste the following:  
\<VirtualHost \*:80\>  
    ServerName \<fossbilling-vm-ip\>  
    DocumentRoot /var/www/html  
    \<Directory /var/www/html\>  
        AllowOverride All  
        Require all granted  
    \</Directory\>  
    ErrorLog /var/log/apache2/fossbilling-error.log  
    CustomLog /var/log/apache2/fossbilling-access.log combined  
\</VirtualHost\>  
   
\# Enable site and restart Apache  
a2ensite fossbilling.conf  
a2dissite 000-default.conf  
systemctl restart apache2

## **15.9  Step 6 — Run FOSSBilling Web Installer**

| 1 | Open Browser Navigate to http://\<fossbilling-vm-ip\>/install |
| :---: | :---- |

| 2 | System Check Installer checks PHP extensions — all should show green. Fix any red items. |
| :---: | :---- |

| 3 | Database Config Enter: Host: localhost | DB: fossbilling | User: fbuser | Password: your-db-password |
| :---: | :---- |

| 4 | Admin Account Set admin email and password for FOSSBilling admin panel |
| :---: | :---- |

| 5 | Site Settings Enter site name, URL, timezone, currency |
| :---: | :---- |

| 6 | Complete Install Click Install — FOSSBilling sets up all tables and config |
| :---: | :---- |

| 7 | Delete Install Folder IMPORTANT: Delete /var/www/html/install folder after setup |
| :---: | :---- |

\# Delete install folder after setup (security requirement)  
rm \-rf /var/www/html/install

| ⚠️  WARNING:   Never leave the /install folder on your server after setup. It is a critical security risk as anyone could re-run the installer and overwrite your database. |
| :---- |

## **15.10  Step 7 — Install Proxmox Module for FOSSBilling**

The Proxmox module allows FOSSBilling to communicate with your cluster API to create and manage VMs automatically:

| 1 | Download Module Visit fossbilling.org/extensions or GitHub → search Proxmox module → download ZIP |
| :---: | :---- |

| 2 | Upload Module Extract ZIP → Copy module folder to /var/www/html/modules/Serviceproxmox/ |
| :---: | :---- |

\# Create modules directory if not exists  
mkdir \-p /var/www/html/modules/Serviceproxmox  
   
\# Copy downloaded module files  
cp \-r proxmox-module/\* /var/www/html/modules/Serviceproxmox/  
   
\# Set permissions  
chown \-R www-data:www-data /var/www/html/modules/  
chmod \-R 755 /var/www/html/modules/

| 3 | Activate Module FOSSBilling Admin → Extensions → Modules → Find Proxmox → Click Activate |
| :---: | :---- |

## **15.11  Step 8 — Create Proxmox API User for FOSSBilling**

Run on node1 — create a dedicated API user for FOSSBilling:

\# SSH into node1  
ssh root@192.168.10.11  
   
\# Create dedicated user for FOSSBilling  
pveum user add fossbilling@pve \--comment 'FOSSBilling Automation User'  
   
\# Set password  
pveum passwd fossbilling@pve  
   
\# Grant Administrator role  
pveum aclmod / \-user fossbilling@pve \-role Administrator  
   
\# RECOMMENDED: Use API token instead  
pveum user token add fossbilling@pve fb-token \--privsep 0  
pveum aclmod / \-token 'fossbilling@pve\!fb-token' \-role Administrator  
   
\# Verify  
pveum user list

## **15.12  Step 9 — Connect FOSSBilling to Proxmox**

| 1 | Open Server Manager FOSSBilling Admin → Products → Hosting Servers → Add New Server |
| :---: | :---- |

| 2 | Select Type Server Type: Proxmox |
| :---: | :---- |

| 3 | Enter Connection Details Fill in: |
| :---: | :---- |

| Field | Value | Notes |
| ----- | ----- | ----- |
| Server Name | My Proxmox Cluster | Display name in FOSSBilling |
| Hostname | 192.168.10.11 | node1 IP address |
| Port | 8006 | Proxmox Web UI port |
| Username | fossbilling@pve | API user created in Step 15.11 |
| Password / Token | your-password or token | API credentials |
| Node | node1 | Default node for VM deployment |
| Secure | Yes (HTTPS) | Always use HTTPS |

| 4 | Test Connection Click Test Connection → expect green success |
| :---: | :---- |

| 5 | Save Click Save Server |
| :---: | :---- |

## **15.13  Step 10 — Create VM Templates for FOSSBilling**

FOSSBilling uses the same Cloud-Init templates as described in Section 14.7. If you have already created them, skip this step. If not, create templates on node1:

\# If templates not yet created — run on node1:  
   
\# Debian 12 template (VM ID 9000\)  
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2  
qm create 9000 \--name debian12-template \--memory 2048 \--cores 2 \--net0 virtio,bridge=vmbr0  
qm importdisk 9000 debian-12-generic-amd64.qcow2 local-lvm  
qm set 9000 \--scsihw virtio-scsi-pci \--scsi0 local-lvm:vm-9000-disk-0  
qm set 9000 \--ide2 local-lvm:cloudinit  
qm set 9000 \--boot c \--bootdisk scsi0  
qm set 9000 \--serial0 socket \--vga serial0  
qm set 9000 \--agent enabled=1  
qm template 9000  
   
\# Ubuntu 22.04 template (VM ID 9001\)  
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img  
qm create 9001 \--name ubuntu2204-template \--memory 2048 \--cores 2 \--net0 virtio,bridge=vmbr0  
qm importdisk 9001 jammy-server-cloudimg-amd64.img local-lvm  
qm set 9001 \--scsihw virtio-scsi-pci \--scsi0 local-lvm:vm-9001-disk-0  
qm set 9001 \--ide2 local-lvm:cloudinit  
qm set 9001 \--boot c \--bootdisk scsi0  
qm set 9001 \--serial0 socket \--vga serial0  
qm set 9001 \--agent enabled=1  
qm template 9001

## **15.14  Step 11 — Create VPS Products in FOSSBilling**

| 1 | Add Product FOSSBilling Admin → Products → New Product → Hosting |
| :---: | :---- |

| 2 | Link Server Select: My Proxmox Cluster server |
| :---: | :---- |

| 3 | Configure Plan Set VM parameters: |
| :---: | :---- |

| Field | Example Value | Notes |
| ----- | ----- | ----- |
| Product Name | VPS Starter | Name shown to customers |
| Template VM ID | 9000 | Cloud-Init template ID |
| CPU Cores | 2 | vCPUs for this plan |
| RAM (MB) | 2048 | Memory in MB |
| Disk (GB) | 20 | Root disk size |
| Storage | local-lvm | Proxmox storage pool |
| Network Bridge | vmbr0 | VM network bridge |
| Monthly Price | 499 | Price in your currency |

| 4 | Set Pricing Pricing tab → Set monthly/annual/free trial prices |
| :---: | :---- |

| 5 | Publish Active: Yes → Save |
| :---: | :---- |

## **15.15  Step 12 — Configure Payment Gateways**

FOSSBilling supports multiple payment gateways out of the box:

| Gateway | Type | Setup Location |
| ----- | ----- | ----- |
| PayPal | Online payments | Admin → Payment Gateways → PayPal → Enter API credentials |
| Stripe | Credit/debit cards | Admin → Payment Gateways → Stripe → Enter API keys |
| Razorpay | India payments | Admin → Payment Gateways → Razorpay → Enter key ID and secret |
| Bank Transfer | Manual | Admin → Payment Gateways → Custom → Add bank details |
| Crypto (CoinGate) | Cryptocurrency | Admin → Payment Gateways → CoinGate → API key |

| 💡  TIP:   For Indian customers, Razorpay is the recommended gateway — supports UPI, net banking, cards, and wallets with easy INR settlement. |
| :---- |

## **15.16  Step 13 — Test Full Automation**

| 1 | Place Test Order FOSSBilling Admin → Orders → Create Order → Select VPS Starter → Test account |
| :---: | :---- |

| 2 | Activate Order Orders → Find order → Click Activate |
| :---: | :---- |

| 3 | Watch Proxmox Proxmox Web UI → Datacenter → new VM should appear on node1 |
| :---: | :---- |

| 4 | Check Email Test account inbox should receive VM IP and login credentials |
| :---: | :---- |

| 5 | Test SSH SSH into new VM with credentials — confirm access works |
| :---: | :---- |

| 6 | Test Client Portal Login to FOSSBilling client area → try Start/Stop/Reboot controls |
| :---: | :---- |

| 7 | Test Suspension Simulate non-payment → verify VM gets suspended automatically |
| :---: | :---- |

## **15.17  FOSSBilling vs HostBill — Which to Use**

| Feature | FOSSBilling | HostBill |
| ----- | ----- | ----- |
| Cost | Free / Open Source | Paid license |
| Proxmox Module | Yes (community module) | Yes (official module) |
| VM Auto-provisioning | Yes | Yes |
| Client Portal | Yes | Yes |
| Payment Gateways | Yes (PayPal, Stripe, Razorpay etc.) | Yes (more options) |
| Support | Community forums / GitHub | Official paid support |
| Customization | Full source code access | Limited |
| Updates | Community driven | Vendor driven |
| Best For | Budget / small-medium setups | Enterprise / large setups |
| Recommendation | Start here | Upgrade if business grows |

## **15.18  Common FOSSBilling Errors and Solutions**

| 🔴  ERROR FOSSBilling Test Connection to Proxmox Failed Root Cause: FOSSBilling VM IP not in Proxmox firewall allowed-networks IPSet, or port 8006 blocked. |
| :---- |
| **✅  SOLUTION** Add FOSSBilling VM IP to Proxmox allowed-networks IPSet via Web UI or terminal. \# In Proxmox Web UI: Datacenter → Firewall → IPSet → allowed-networks → Add IP/CIDR: \<fossbilling-vm-ip\>/32 Comment: FOSSBilling Server   \# Verify connectivity from FOSSBilling VM: curl \-k https://192.168.10.11:8006/api2/json/version \# Should return Proxmox version JSON |

| 🔴  ERROR FOSSBilling white screen or 500 error after install Root Cause: Missing PHP extensions, wrong file permissions, or .htaccess not working (mod\_rewrite not enabled). |
| :---- |
| **✅  SOLUTION** Check Apache error log, verify all PHP extensions installed, ensure mod\_rewrite is enabled. \# Check Apache error log: tail \-50 /var/log/apache2/fossbilling-error.log   \# Enable mod\_rewrite if missing: a2enmod rewrite systemctl restart apache2   \# Fix file permissions: chown \-R www-data:www-data /var/www/html chmod \-R 755 /var/www/html chmod \-R 777 /var/www/html/data   \# Check all required PHP extensions: php \-m | grep \-E 'curl|xml|mbstring|intl|zip|gd|bcmath|pdo' |

| 🔴  ERROR VM created by FOSSBilling but Cloud-Init not running — no credentials sent Root Cause: Cloud-Init not installed in template VM or Cloud-Init drive not attached to template. |
| :---- |
| **✅  SOLUTION** Rebuild template with Cloud-Init properly installed. Verify ide2 cloudinit drive exists on template before converting. \# Verify Cloud-Init drive on template: qm config 9000 | grep ide2 \# Expected: ide2: local-lvm:vm-9000-cloudinit   \# Re-add if missing: qm set 9000 \--ide2 local-lvm:cloudinit   \# Inside template VM — install cloud-init: apt install cloud-init \-y   \# Re-convert: qm template 9000 |

## **15.19  FOSSBilling Security Best Practices**

| Practice | Implementation |
| ----- | ----- |
| Dedicated Proxmox API user | Use fossbilling@pve — never root |
| API token over password | pveum user token add — individually revocable |
| HTTPS for FOSSBilling | Install SSL certificate via Let's Encrypt (certbot) |
| Delete install folder | rm \-rf /var/www/html/install after setup |
| Regular backups | Backup FOSSBilling VM and MySQL database daily |
| Keep FOSSBilling updated | Check fossbilling.org for updates regularly |
| Firewall FOSSBilling VM | Restrict SSH to trusted IPs on FOSSBilling VM too |

\# Install Let's Encrypt SSL on FOSSBilling server (recommended):  
apt install certbot python3-certbot-apache \-y  
certbot \--apache \-d \<your-domain.com\>  
\# Follow prompts — auto-renews every 90 days

