PROXMOX VE

Complete Installation & Cluster Configuration Guide

3-Node High Availability Cluster | Debian Bookworm | Version 3.0

Node	IP Address	Hostname	Role	Web UI Port
VPS 1	192.168.10.11	node1	Master / Primary	8006
VPS 2	192.168.10.12	node2	Worker Node	8006
VPS 3	192.168.10.13	node3	Worker Node	8006
Method A: Install on Debian | Method B: Direct ISO Install | GUI Cluster Join

Table of Contents
Section	Title	Page
1	Pre-Installation: Debian OS Setup	3
2	Method A: Install Proxmox VE on Debian	4
3	Method B: Install Proxmox VE Directly (ISO — No OS)	6
4	Errors Encountered & Solutions (incl. live cluster join SSL cert issue)	8
5	Post-Installation: Opening Ports & Firewall	9
6	Accessing the Proxmox Web UI	10
7	Creating a 3-Node Cluster via Web UI (GUI)	11
8	Joining Nodes to Cluster via Web UI (GUI)	13
9	Post-Cluster Configuration	15
10	Creating VMs & Containers in Proxmox	16
11	Progress Checklist	18
12	Quick Reference & Command Glossary	19
13	Restricting Web UI Access to Trusted Networks Only (Proxmox Firewall)	21
14	Automating VM Provisioning with HostBill (Cloud-Init, API, Products)	23
15	Automating VM Provisioning with FOSSBilling (Free Open Source)	26



Section 1 — Pre-Installation: Debian OS Setup

This section covers the initial Debian Bookworm installation and base configuration steps required before installing Proxmox VE using Method A (on top of Debian).

If you are using Method B (direct ISO install), skip to Section 3.

1.1 Debian OS Installation Notes
During the Debian installation wizard, two important configuration prompts appear:

Postfix — System Mail Name

The installer shows a 'System mail name' field

Action: Keep the default hostname as-is (e.g. 7758-24442)

Press Tab → select <Ok> → Enter to continue

⚠️ WARNING: For VPS environments not sending email, this value does not affect cluster functionality. Keep default.
GRUB Bootloader Prompt

During apt upgrade, GRUB may show a configuration conflict prompt

Always select: 'keep the local version currently installed'

⚠️ WARNING: Never choose 'install the package maintainer's version' on VPS — can cause boot failure.
1.2 Set Unique Hostnames on Each VPS
Proxmox requires each node to have a unique hostname. Run on each respective VPS:

# On VPS 1 (192.168.10.11)
hostnamectl set-hostname node1

# On VPS 2 (192.168.10.12)
hostnamectl set-hostname node2

# On VPS 3 (192.168.10.13)
hostnamectl set-hostname node3

# Verify hostname on each node
hostname

1.3 Configure /etc/hosts on ALL 3 Nodes
Edit /etc/hosts to allow nodes to communicate using hostnames without DNS:

nano /etc/hosts

# Add these lines at the bottom:
192.168.10.11 node1
192.168.10.12 node2
192.168.10.13 node3

💡 TIP: Save with Ctrl+X → Y → Enter. This must be done identically on ALL 3 nodes.
1.4 System Update
apt update && apt upgrade -y

Section 2 — Method A: Install Proxmox VE on Debian
This is the method used in our live setup. Proxmox VE is installed on top of an existing Debian Bookworm installation. This is the officially supported method by Proxmox.

Requirement	Details
Operating System	Debian 12 (Bookworm) — 64-bit
RAM	Minimum 2 GB (4 GB+ recommended)
Storage	Minimum 32 GB
Network	Static IP or consistent DHCP
Access	Root SSH access
2.1 Fix Proxmox Repository
By default, Debian may have the Proxmox Enterprise repository configured, which requires a paid subscription. Remove it and add the free repository:

# Remove enterprise repo
rm /etc/apt/sources.list.d/pve-enterprise.list

# Add free no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \
> /etc/apt/sources.list.d/pve-no-subscription.list

# Add Proxmox GPG key
wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg \
-O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg

# Update package lists
apt update && apt upgrade -y

2.2 Install Proxmox VE
apt install proxmox-ve postfix open-iscsi -y

ℹ️ INFO: Installation takes 5-10 minutes. If Postfix prompts appear: select 'Local only' and keep the default mail name.
2.3 Kernel Decision
Standard Proxmox docs recommend removing the Debian kernel. This step is optional and was skipped in our setup for VPS stability.

Step	Command	Recommendation for VPS
Remove Debian kernel	apt remove linux-image-amd64 'linux-image-6.1*' -y	Optional — skip for stability
Update GRUB	update-grub	Only if kernel was removed
💡 TIP: On bare-metal servers, removing the Debian kernel is recommended. On VPS environments, keeping it is safer.
2.4 Reboot
reboot

ℹ️ INFO: Wait 60-90 seconds after reboot before accessing the Web UI.
2.5 Verify Proxmox Service
systemctl status pveproxy

# Should show: active (running)

Section 3 — Method B: Direct ISO Install (No OS on VPS)
This method installs Proxmox VE directly as the operating system — no Debian pre-installation needed. This is the cleanest and most recommended method for bare-metal servers and VPS providers that allow custom ISO mounting.

⚠️ WARNING: Not all VPS providers support ISO mounting. Check with your provider before attempting this method.
3.1 Requirements
Requirement	Details
VPS Provider	Must support custom ISO upload or mounting
ISO File	Proxmox VE latest ISO from proxmox.com
RAM	Minimum 2 GB (4 GB+ recommended)
Storage	Minimum 32 GB SSD
Console Access	VNC or noVNC console from provider panel
3.2 Download Proxmox VE ISO
Download the latest Proxmox VE ISO from the official website:

# Official download URL:
https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso

# Example filename:
proxmox-ve_8.x.x-1.iso

3.3 Mount ISO on VPS (Provider Panel Steps)
The exact steps vary by provider but follow this general flow:

1	Login to VPS Provider Control Panel Navigate to your VPS management dashboard
2	Find ISO / Media Section Look for 'ISO Library', 'Media', or 'OS Templates' in the panel
3	Upload or Link Proxmox ISO Either upload the ISO file or paste the download URL if provider supports remote ISO
4	Mount ISO to VPS In VPS settings → select boot device → choose the uploaded Proxmox ISO
5	Set Boot Order Set CD/DVD (ISO) as first boot device
6	Open VNC Console Open the noVNC or VNC console from the provider panel before rebooting
7	Reboot VPS Reboot the VPS — the Proxmox installer will appear in the VNC console
3.4 Proxmox VE Installer Walkthrough
Once the VPS boots from the ISO, the graphical Proxmox installer appears. Follow these steps:

Step 1 — Welcome Screen

Click: Install Proxmox VE (Graphical)
Step 2 — License Agreement

Click: I agree
Step 3 — Target Disk Selection

Select the disk to install Proxmox VE on

Click Options to configure filesystem (ext4 or ZFS recommended)

Click Next

⚠️ WARNING: All data on the selected disk will be erased. Make sure you select the correct disk.
Step 4 — Location and Timezone

Country: Select your country (e.g. India)

Timezone: Asia/Kolkata (or your timezone)

Keyboard: en-us

Click Next

Step 5 — Admin Password & Email

Password: Set a strong root password (minimum 8 characters)

Confirm Password: Re-enter the same password

Email: Enter an admin email (used for alerts)

Click Next

💡 TIP: Save this root password securely — you will need it to login to the Web UI.
Step 6 — Network Configuration

Management Interface: Select the main network interface (usually eth0 or ens3)

Hostname (FQDN): Enter the node hostname — e.g. pve-master.homelab.local

IP Address: Enter your VPS IP — e.g. 192.168.10.11

Netmask: Enter your subnet mask — e.g. 255.255.255.0

Gateway: Enter your gateway IP (from provider panel)

DNS Server: 8.8.8.8 (or provider DNS)

Click Next

⚠️ WARNING: The IP, Netmask and Gateway values must exactly match your VPS network settings from the provider panel.
Step 7 — Summary & Install

Review all settings on the summary screen

Click Install to begin the installation

Installation takes 5-15 minutes

System will reboot automatically when done

Step 8 — After Reboot

Remove the ISO from boot device in provider panel after reboot

Set boot order back to: Hard Disk first

The VPS will now boot directly into Proxmox VE

ℹ️ INFO: After successful install, Proxmox VE will show a console message with the Web UI URL: https://<your-ip>:8006
Section 4 — Errors Encountered & Solutions
This section documents every error encountered during the live setup with root cause analysis and exact fixes.

Error 1 — Proxmox Enterprise Repository 401 Unauthorized
🔴 ERROR Err:7 https://enterprise.proxmox.com/debian/pve bookworm InRelease 401 Unauthorized [IP: 117.120.5.231 443]E: Failed to fetch .../InRelease 401 UnauthorizedE: The repository is not signed. Root Cause: The enterprise.proxmox.com repository requires a paid subscription licence. Without a valid key, every apt update returns HTTP 401 Unauthorized.
✅ SOLUTION Remove the enterprise repo. Add the official free no-subscription repository instead. Apply on ALL 3 nodes. rm /etc/apt/sources.list.d/pve-enterprise.list echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" \ > /etc/apt/sources.list.d/pve-no-subscription.list wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg \ -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg apt update && apt upgrade -y
Error 2 — Web UI Port 8006 ERR_TIMED_OUT
🔴 ERROR ERR_TIMED_OUT — https://192.168.10.11:8006192.168.10.11 took too long to respond. Root Cause: Two-layer firewall block: (1) OS-level iptables was not allowing inbound port 8006. (2) VPS provider network-level firewall was blocking port 8006 externally before packets even reached the server.
✅ SOLUTION Two-step fix: First open port 8006 in iptables on the server. Second, contact VPS provider to open port 8006 at network/datacenter level. # Fix 1 — OS level (run on ALL 3 nodes) iptables -A INPUT -p tcp --dport 8006 -j ACCEPT iptables -A INPUT -p tcp --dport 443 -j ACCEPT iptables -A INPUT -p tcp --dport 80 -j ACCEPT apt install iptables-persistent -y netfilter-persistent save # Verify port is listening ss -tlnp
Error 3 — Cluster Join Warning: No Cluster Network Links Passed
🔴 ERROR No cluster network links passed explicitly, falling back to default.Request addition of this node... (hangs or fails) Root Cause: When joining a node to the cluster via pvecm, no dedicated cluster network interface was explicitly specified. Proxmox falls back to the default interface (vmk0/eth0) which may not be routable between nodes.
✅ SOLUTION Wait 1-2 minutes first — if the join completes with TASK OK, no action needed. If it hangs or fails, explicitly specify the network interface during the join command. # Check which VMkernel/interface is active on the joining node: ip a # Rejoin with explicit network interface specified: pvecm add 192.168.10.11 --link0 <joining-node-ip> # Example — joining node2 with explicit IP: pvecm add 192.168.10.11 --link0 192.168.10.12 # Verify cluster status after join: pvecm status pvecm nodes
ℹ️ INFO: This warning message is informational in most cases. As long as the join process completes with TASK OK and pvecm nodes shows all 3 nodes, the cluster is healthy.
Error 4 — SSL Certificate Missing After Node Join: pve-ssl.pem Does Not Exist
🔴 ERROR '/etc/pve/nodes/node2/pve-ssl.pem' does not exist! (500)Node2 shows error in Proxmox Web UI after cluster join. Root Cause: After node2 joined the cluster, its SSL certificate was not correctly generated or synced to node1's cluster filesystem (/etc/pve/nodes/node2/). This is a certificate sync issue — NOT a network or cluster quorum issue. The cluster itself is healthy.
✅ SOLUTION Regenerate the SSL certificate on node2 using pvecm updatecerts --force, restart cluster services on both nodes, then verify the cert file exists in /etc/pve/nodes/node2/ on node1. # ── STEP 1: Verify cluster IS actually healthy (run on node1) ── pvecm status pvecm nodes # Expected: all 3 nodes listed, Quorum: Yes # ── STEP 2: SSH from node1 into node2 ── ssh root@192.168.10.12 # Type: yes (accept SSH fingerprint) # Enter node2 root password # ── STEP 3: Regenerate SSL cert on node2 ── pvecm updatecerts --force # Expected output: # generate new node certificate # merge authorized SSH keys # ── STEP 4: Restart all cluster services on node2 ── systemctl restart pvedaemon pveproxy pve-cluster # ── STEP 5: Verify cert now exists on node2 ── ls /etc/pve/nodes/node2/ # You should see: pve-ssl.pem pve-ssl.key pve.crt # ── STEP 6: Back on node1 — restart services ── systemctl restart pve-cluster pvedaemon pveproxy # ── STEP 7: Verify cert synced to node1 ── ls /etc/pve/nodes/node2/ # pve-ssl.pem must appear here on node1 too # ── STEP 8: Repeat for node3 if needed ── ssh root@192.168.10.13 pvecm updatecerts --force systemctl restart pvedaemon pveproxy pve-cluster
ℹ️ INFO: This error was encountered live during the 3-node cluster setup in this guide. The cluster quorum was healthy (all 3 nodes joined successfully) — only the SSL certificate sync had failed. After pvecm updatecerts --force the Web UI error was fully resolved.
Error 4b — SSH Fingerprint Prompt When Connecting Between Nodes
🔴 ERROR The authenticity of host '192.168.10.12' can't be established.ECDSA key fingerprint is SHA256:xxxx...Are you sure you want to continue connecting (yes/no/[fingerprint])? Root Cause: First-time SSH connection between cluster nodes — SSH has not yet stored the remote host's fingerprint in known_hosts. This is expected and normal on first inter-node connection.
✅ SOLUTION Type 'yes' and press Enter to accept and store the fingerprint. This is safe — you are connecting to your own cluster node. # When prompted — always type: yes # Then enter the node's root password when asked # To pre-accept all node fingerprints (run on node1): ssh-keyscan 192.168.10.12 >> ~/.ssh/known_hosts ssh-keyscan 192.168.10.13 >> ~/.ssh/known_hosts # Set up passwordless SSH between nodes (optional but recommended): ssh-keygen -t rsa -b 4096 # Generate key on node1 ssh-copy-id root@192.168.10.12 ssh-copy-id root@192.168.10.13
Error 5 — Cluster Join Fails: Corosync Port Not Reachable
🔴 ERROR pvecm add failed: corosync not reachable on node1cluster join aborted — cannot connect to quorum service Root Cause: Corosync UDP ports 5404 and 5405 are not open between nodes. The joining node cannot reach the cluster quorum service on the master node. This is a firewall / provider-level port blocking issue.
✅ SOLUTION Open UDP ports 5404 and 5405 in iptables on ALL nodes AND request the VPS provider to open these ports at network level. # Open corosync ports on ALL 3 nodes: iptables -A INPUT -p udp --dport 5404 -j ACCEPT iptables -A INPUT -p udp --dport 5405 -j ACCEPT iptables -A OUTPUT -p udp --dport 5404 -j ACCEPT iptables -A OUTPUT -p udp --dport 5405 -j ACCEPT netfilter-persistent save # Test corosync port reachability from node2 to node1: nc -vzu 192.168.10.11 5404 nc -vzu 192.168.10.11 5405 # Check corosync service status on node1: systemctl status corosync corosync-cfgtool -s


Section 5 — Post-Installation: Ports & Firewall
All required ports must be open at both the OS level (iptables) and the provider/network level. Request all ports from your provider at once.

5.1 Required Ports Reference
Port	Protocol	Purpose	Priority
22	TCP	SSH access	Critical
80	TCP	HTTP redirect	Required
443	TCP	HTTPS	Required
8006	TCP	Proxmox Web UI	Critical
111	TCP+UDP	rpcbind / NFS portmapper	Cluster
3128	TCP	SPICE proxy (VM console)	Optional
5404	UDP	Corosync cluster communication	Critical for cluster
5405	UDP	Corosync cluster communication	Critical for cluster
5900-5999	TCP	VNC console access to VMs	Optional
8007	TCP	Proxmox backup server	Optional
5.2 Open Ports in iptables (OS Level)
# Run on ALL 3 nodes
iptables -A INPUT -p tcp --dport 22 -j ACCEPT
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
iptables -A INPUT -p tcp --dport 443 -j ACCEPT
iptables -A INPUT -p tcp --dport 8006 -j ACCEPT
iptables -A INPUT -p tcp --dport 3128 -j ACCEPT
iptables -A INPUT -p udp --dport 5404 -j ACCEPT
iptables -A INPUT -p udp --dport 5405 -j ACCEPT
iptables -A INPUT -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -p udp --dport 111 -j ACCEPT

# Save rules persistently
apt install iptables-persistent -y
netfilter-persistent save

# Verify
iptables -L -n | grep ACCEPT

5.3 Provider-Level Port Request Template
Send this message to your VPS provider support to open ports at the network level:

Subject: Open Ports on 3 VPS Servers for Proxmox VE Cluster

Please open the following ports on all 3 of my VPS servers:

Ports (TCP): 22, 80, 443, 8006, 3128, 111, 5900-5999
Ports (UDP): 5404, 5405, 111

VPS Server IPs:
192.168.10.11 (node1 — Master)
192.168.10.12 (node2 — Worker)
192.168.10.13 (node3 — Worker)

5A.5 WireGuard Encrypted Inter-Node Overlay — Final Implementation
5A.5.1 Why WireGuard Was Used
((((# Also we have tried for the GRE but due to provider network limitation cant able to do for reference attache beloe )))(((THis includes the network bridge creation and all)))) [[[[[[## Production Network Architecture, VLANs & Dedicated Bridges

This section documents the production network layer used by the 3-node Proxmox environment. It is intentionally placed immediately after the base firewall/port configuration and before Web UI and cluster-management procedures, because network topology and bridge design affect VM/CT connectivity, migration, and future service isolation.

5A.1 Production Network Design Goals
The production network is designed to keep the following traffic logically separated wherever the provider/network topology supports it:

Traffic Type	Purpose	Recommended Isolation
Management	Proxmox Web UI, SSH, node administration	Management network / trusted access
Cluster / Corosync	Node heartbeat and cluster control traffic	Dedicated private network
VM / Container	Customer and workload traffic	VLAN-aware bridge
Overlay / VPN	Encrypted private inter-node connectivity	WireGuard tunnel
Backup	VM/CT backup traffic	Dedicated network or restricted path
Important: VLAN IDs, IP ranges, gateways, and physical/virtual interface names must match the actual network supplied by the VPS/provider. Do not copy example values from this guide directly into production.

5A.2 VLAN-Aware Linux Bridge
For production VM and container networking, the Proxmox bridge can be configured as VLAN-aware. This allows multiple VLANs to be carried through the same bridge while assigning the required VLAN tag to individual VM/CT network interfaces.

Web UI
Open Node → System → Network.
Select the production VM bridge (for example vmbr0 or the dedicated VM bridge).
Click Edit.
Enable VLAN aware.
Verify the bridge port/interface is the intended uplink.
Click OK → Apply Configuration.
Repeat the required configuration consistently on all cluster nodes.
Verify from terminal
ip -br link
ip -br addr
bridge link
cat /etc/network/interfaces
The bridge configuration should show VLAN awareness and the correct uplink/port.

5A.3 Dedicated VM/Container Bridge
Where the production design requires a separate bridge for workload traffic, create a dedicated Linux bridge rather than mixing workload traffic with the management bridge.

Example structure:

Provider / Uplink
       |
       +---------------- Management
       |
       +---------------- VM/CT VLAN traffic
                          |
                        vmbrX
                       /    \
                     VM      CT
The exact bridge name, physical/uplink interface, IP addressing, and VLAN configuration must be taken from the actual production network design.

Production rule: Do not add a default gateway to a dedicated private/overlay bridge unless that bridge is explicitly intended to provide the node's default routing.

5A.4 GRE Network / Overlay — Initial Attempt
GRE was initially considered for creating a private overlay network between the Proxmox nodes.

The initial design involved establishing GRE tunnels between the nodes and using the resulting private network for inter-node communication.

However, the required GRE functionality was not enabled/supported through the provider network, so the GRE-based approach could not be used reliably in the production environment.

Therefore, the GRE tunnels were not retained as part of the final implementation.

Initial GRE Verification
During the initial attempt, the following commands were used to verify the tunnel:

ip tunnel show
ip -br addr
ip route
ping -c 4 <remote-gre-ip>
If a GRE interface was attached to a Linux bridge, the relationship could be checked using:

bridge link
ip link show
GRE Limitation
⚠️ WARNING: GRE itself does not provide encryption. Even if GRE functionality is available, sensitive production traffic should use an encrypted transport such as WireGuard or IPsec.

Because GRE was not usable through the provider network, the final implementation was changed to WireGuard.

5A.5 WireGuard Encrypted Inter-Node Overlay — Final Implementation
5A.5.1 Why WireGuard Was Used
After the GRE implementation was found to be unsuitable because the required GRE functionality was not enabled/supported by the provider network, WireGuard was selected as the final overlay solution.

WireGuard provides:

Encrypted node-to-node communication
Secure peer authentication using public/private keys
Lightweight VPN tunneling
Persistent tunnel support through systemd
Direct connectivity between all three Proxmox nodes
No dependency on provider-side GRE support
The final production topology uses WireGuard for encrypted private connectivity between:

Node1 <--------> Node2
  |                |
  |                |
  +--------------> Node3
         ^
         |
       Node2
All three nodes were configured as WireGuard peers.

5A.5.2 Final WireGuard Connectivity
The final WireGuard implementation was verified successfully:

Connection	Status
Node1 ↔ Node2	✅ Working
Node1 ↔ Node3	✅ Working
Node2 ↔ Node3	✅ Working
WireGuard tunnel after reboot	✅ Persistent
Old GRE configuration	✅ Removed
5A.5.3 WireGuard Installation
Install WireGuard on all three Proxmox nodes:

apt update
apt install wireguard -y
Verify the installation:

wg --version
5A.5.4 WireGuard Configuration Directory
WireGuard configuration files are stored under:

/etc/wireguard/
Create and secure the directory:

mkdir -p /etc/wireguard
chmod 700 /etc/wireguard
The primary tunnel used in this implementation is:

wg0
with its configuration stored at:

/etc/wireguard/wg0.conf
5A.5.5 WireGuard Key Generation
Each Proxmox node requires its own private/public key pair.

Run on each node:

cd /etc/wireguard
umask 077

wg genkey | tee privatekey | wg pubkey > publickey
Check the generated files:

ls -l /etc/wireguard/
The following files are created:

/etc/wireguard/privatekey
/etc/wireguard/publickey
Security: Never share or document the private key. Only the public key should be exchanged between peers.

5A.5.6 Create the wg0.conf Configuration
Create the WireGuard configuration:

nano /etc/wireguard/wg0.conf
Generic node configuration structure:

[Interface]
Address = <node-wireguard-ip>/<subnet>
PrivateKey = <node-private-key>
ListenPort = <wireguard-udp-port>

[Peer]
PublicKey = <remote-node-public-key>
AllowedIPs = <remote-node-wireguard-ip>/32
Endpoint = <remote-node-public-ip>:<wireguard-udp-port>
PersistentKeepalive = 25
Each node contains its own [Interface] section and the appropriate [Peer] entries for the other Proxmox nodes.

Important: Actual WireGuard IP addresses, public keys, private keys, UDP port, and public endpoints must be taken from the production configuration. Do not replace them with example values from this document.

5A.5.7 Three-Node WireGuard Peer Design
The final design is a full three-node peer topology:

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
Logical peer relationships:

Node1
 ├── Peer: Node2
 └── Peer: Node3

Node2
 ├── Peer: Node1
 └── Peer: Node3

Node3
 ├── Peer: Node1
 └── Peer: Node2
This ensures that every Proxmox node can communicate with the other two through the encrypted WireGuard overlay.

5A.5.8 Protect the WireGuard Configuration
The WireGuard configuration contains the node's private key and must therefore be protected:

chmod 600 /etc/wireguard/wg0.conf
chmod 600 /etc/wireguard/privatekey
chmod 644 /etc/wireguard/publickey
Verify permissions:

ls -l /etc/wireguard/
5A.5.9 Start the WireGuard Tunnel
Bring up the WireGuard interface:

wg-quick up wg0
Verify the interface:

ip addr show wg0
Check WireGuard peer status:

wg show
Verify routing:

ip route
5A.5.10 Enable WireGuard at Boot
WireGuard was configured to start automatically after a node reboot.

Enable the systemd service:

systemctl enable wg-quick@wg0
Start the service:

systemctl start wg-quick@wg0
Verify:

systemctl status wg-quick@wg0 --no-pager
The service should show:

active (exited)
or an equivalent successful WireGuard startup state.

5A.5.11 Firewall / UDP Requirement
WireGuard requires its configured UDP listening port to be reachable between the participating nodes.

Verify the listening socket:

ss -lunp
The configured WireGuard UDP port must be permitted by the applicable:

Proxmox firewall rules
VPS/provider firewall
Host firewall
Security group, if applicable
The exact UDP port must match the production WireGuard configuration.

5A.5.12 Verify Node-to-Node Connectivity
After bringing up WireGuard on all nodes, verify:

wg show
The output should show the configured peers and recent handshakes.

Verify the WireGuard interface:

ip -br addr show wg0
Test the private WireGuard IP of each remote node:

ping -c 4 <node2-wireguard-ip>
ping -c 4 <node3-wireguard-ip>
Perform equivalent tests from Node2 and Node3.

Expected result:

Node1 → Node2  ✅
Node1 → Node3  ✅
Node2 → Node1  ✅
Node2 → Node3  ✅
Node3 → Node1  ✅
Node3 → Node2  ✅
5A.5.13 Reboot Persistence Verification
Because the WireGuard tunnel is required for persistent inter-node connectivity, reboot testing was performed after enabling the systemd service.

After reboot:

systemctl status wg-quick@wg0 --no-pager
Then verify:

wg show
ip addr show wg0
Finally test connectivity to the other nodes:

ping -c 4 <remote-node-wireguard-ip>
Result: WireGuard successfully comes back after reboot and the three-node encrypted connectivity remains operational.

5A.5.14 GRE Cleanup
Once WireGuard was confirmed working, the old GRE configuration was removed to prevent stale interfaces, routes, or conflicting overlay configuration.

Remove old GRE entries from:

nano /etc/network/interfaces
Delete the old GRE-related configuration.

Remove the previously created GRE tunnels:

ip tunnel del gre-node2 2>/dev/null
ip tunnel del gre-node3 2>/dev/null
ip tunnel del gre-node1 2>/dev/null
Verify that no unwanted GRE tunnel remains:

ip tunnel show
Verify the final routing and interfaces:

ip -br addr
ip route
The final production environment should contain the WireGuard overlay, not the abandoned GRE tunnels.

5A.5.15 Final Implementation Status
The production overlay networking was finalized using WireGuard because GRE could not be used through the provider network.

Component	Final Status
GRE attempt	⚠️ Not usable due to provider network limitation
WireGuard installation	✅ Completed
WireGuard key generation	✅ Completed
wg0 configuration	✅ Completed
Node1 ↔ Node2	✅ Working
Node1 ↔ Node3	✅ Working
Node2 ↔ Node3	✅ Working
WireGuard persistent startup	✅ Enabled
Reboot verification	✅ Passed
Old GRE cleanup	✅ Completed
Encrypted inter-node overlay	✅ Operational
Final architecture:

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
The WireGuard overlay is now the final encrypted inter-node network path used by the three-node Proxmox environment. GRE was only the initial approach and has been removed after the provider-side limitation was identified.

5A.6 /etc/network/interfaces Change Control
Network configuration changes on a production Proxmox node must be performed carefully because an incorrect bridge, interface, route, or gateway can disconnect the node.

Before making changes:

cp /etc/network/interfaces /etc/network/interfaces.bak

ip -br addr
ip route
After changes:

ifreload -a

ip -br addr
ip route
bridge link
If the node becomes unreachable, use the VPS provider's console/out-of-band access to restore the previous working configuration.

5A.7 VLAN Assignment to VMs and Containers
Once the VLAN-aware bridge is operational, a VLAN can be assigned per VM/CT network interface.

VM
VM → Hardware → Network Device → Edit

Bridge: select the production VLAN-aware bridge
VLAN Tag: enter the required VLAN ID
Model: VirtIO
LXC Container
CT → Network → Edit

Bridge: select the production VLAN-aware bridge
VLAN Tag: enter the required VLAN ID where supported by the chosen network design
Configure the container IP/gateway according to the assigned VLAN
Example logical design:

Workload	Bridge	VLAN Tag	Purpose
Management	vmbr0	— / native	Proxmox management
Customer VM	<vmbr-vm>	<VLAN-ID>	Customer workload
Customer CT	<vmbr-vm>	<VLAN-ID>	Customer workload
Backup	<vmbr-backup>	<VLAN-ID>	Backup traffic
Replace placeholders only after the provider/network VLAN plan is finalized.

5A.8 Network Validation Checklist
Before proceeding to production VM/CT provisioning, validate:

 VLAN-aware bridge enabled on required nodes
 Correct uplink/interface attached to the bridge
 Dedicated VM/CT bridge verified where required
 GRE attempted and provider-side limitation identified
 Old GRE tunnels/configuration removed
 WireGuard installed on all three nodes
 WireGuard peer configuration completed
 Node1 ↔ Node2 connectivity verified
 Node1 ↔ Node3 connectivity verified
 Node2 ↔ Node3 connectivity verified
 WireGuard persistence after reboot verified
 /etc/network/interfaces consistent with the approved final design
 Node management connectivity still working
 VLAN-tagged test VM receives expected connectivity
 VLAN-tagged test CT receives expected connectivity
 Default gateway exists only where required
 Firewall rules permit required VLAN/overlay traffic
 Live migration networking remains functional ]]]]]]]]]
During the initial production network implementation, a GRE-based private overlay was attempted between the Proxmox nodes.

However, the required GRE functionality was not available/usable through the VPS provider's network. Due to this provider-side network limitation, the GRE tunnel could not be used as the final inter-node communication method.

Therefore, WireGuard was selected and implemented as the final encrypted overlay network between the three Proxmox nodes.

The final implementation provides secure private connectivity between:

Connection	Status
node1 ↔ node2	✅ Working
node1 ↔ node3	✅ Working
node2 ↔ node3	✅ Working
WireGuard was also configured to start automatically after system boot, making the tunnel persistent across reboots.

5A.5.2 Final Network Approach
The network implementation followed this sequence:

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
Final decision: GRE was treated as the initial/failed implementation attempt, while WireGuard became the final production overlay.

5A.5.3 WireGuard Installation
WireGuard was installed on all three Proxmox nodes.

apt update
apt install wireguard -y
Verify the installation:

wg --version
The installation was performed on:

node1
node2
node3
5A.5.4 WireGuard Configuration Directory
WireGuard configuration files are maintained under:

/etc/wireguard/
The primary tunnel interface used for the implementation is:

wg0
The main configuration file is:

/etc/wireguard/wg0.conf
Create the directory if required:

mkdir -p /etc/wireguard
chmod 700 /etc/wireguard
5A.5.5 WireGuard Key Generation
Each Proxmox node requires its own WireGuard key pair.

The general key-generation process is:

umask 077
wg genkey | tee privatekey | wg pubkey > publickey
This generates:

privatekey
publickey
Each node keeps its own private key.

⚠️ SECURITY WARNING: Never place actual WireGuard private keys inside this documentation, Git repositories, screenshots, or public support messages.

Only the public key should be exchanged with the other WireGuard peers.

5A.5.6 Create the wg0.conf Configuration
The WireGuard tunnel configuration is stored at:

/etc/wireguard/wg0.conf
Create/edit the file:

nano /etc/wireguard/wg0.conf
The configuration follows the standard WireGuard structure:

[Interface]
Address = <node-wireguard-ip>/<subnet>
PrivateKey = <node-private-key>
ListenPort = <wireguard-udp-port>

[Peer]
PublicKey = <remote-node-public-key>
AllowedIPs = <remote-node-wireguard-ip>/32
Endpoint = <remote-node-public-ip>:<wireguard-udp-port>
PersistentKeepalive = 25
Additional [Peer] sections are configured for the remaining Proxmox nodes.

Note: The actual WireGuard IP addresses, public keys, endpoint addresses and UDP port used on the live cluster should be inserted here if this document is intended to contain the complete production configuration.

5A.5.7 Three-Node WireGuard Peer Design
The final WireGuard design provides encrypted communication between all three Proxmox nodes.

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
This creates direct encrypted paths between all nodes.

The AllowedIPs configuration determines which destination addresses are routed through each WireGuard peer.

5A.5.8 Protect the WireGuard Configuration
Because wg0.conf contains the node's private key, its permissions must be restricted.

chmod 600 /etc/wireguard/wg0.conf
Verify:

ls -l /etc/wireguard/wg0.conf
The configuration should only be accessible by the required privileged account.

5A.5.9 Start the WireGuard Tunnel
For the initial test, bring the tunnel up manually:

wg-quick up wg0
Verify the WireGuard interface:

ip addr show wg0
Check WireGuard status:

wg show
Check routing:

ip route
The wg show output should display the configured peers and, after successful communication, a recent handshake.

5A.5.10 Enable WireGuard at Boot
After confirming that the tunnel works correctly, WireGuard was configured to start automatically with the operating system.

systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
Verify:

systemctl status wg-quick@wg0 --no-pager
This ensures that the WireGuard interface is automatically restored after a node reboot.

5A.5.11 Firewall / UDP Requirement
WireGuard communicates through UDP using the configured listening port.

Verify the listening socket:

ss -lunp
The configured WireGuard UDP port must be permitted by:

VPS provider/network firewall
OS firewall, if enabled
Any additional upstream firewall/security group
This was one of the important reasons for moving away from GRE: the provider network did not provide the required GRE functionality, while WireGuard could operate using a normal provider-supported UDP transport.

5A.5.12 Verify Node-to-Node Connectivity
After WireGuard was configured on all three nodes, connectivity was tested between the WireGuard/private addresses.

On each node:

wg show
ip -br addr show wg0
ip route
Test the remote WireGuard addresses:

ping -c 4 <node2-wireguard-ip>
ping -c 4 <node3-wireguard-ip>
The same tests should be performed from node2 and node3.

Final connectivity result:

Test	Result
node1 → node2	✅
node1 → node3	✅
node2 → node1	✅
node2 → node3	✅
node3 → node1	✅
node3 → node2	✅
WireGuard handshake	✅
Encrypted private overlay	✅
5A.5.13 Reboot Persistence Verification
WireGuard persistence was also verified after reboot.

After restarting a node:

reboot
Check the WireGuard service:

systemctl status wg-quick@wg0 --no-pager
Verify the interface:

ip addr show wg0
Check peer status:

wg show
Finally, test connectivity again:

ping -c 4 <remote-wireguard-ip>
Successful connectivity after reboot confirms that the WireGuard overlay is persistent and automatically restored.

5A.5.14 GRE Cleanup
Once WireGuard was confirmed to be working, the old GRE configuration was removed to avoid unnecessary tunnel interfaces or routing conflicts.

Edit the network configuration:

nano /etc/network/interfaces
Remove the old GRE-related entries that were created during the initial GRE implementation attempt.

The previously created GRE interfaces were then removed where applicable:

ip tunnel del gre-node2 2>/dev/null
ip tunnel del gre-node3 2>/dev/null
ip tunnel del gre-node1 2>/dev/null
Verify that the old GRE tunnels are no longer active:

ip tunnel show
ip -br link
ip route
⚠️ NOTE: Only remove GRE interfaces that were actually created on the respective node. Interface names can differ depending on the original configuration.

5A.5.15 Final Implementation Status
The final inter-node network implementation is:

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
Final status:

✅ GRE initially attempted
✅ GRE provider/network limitation identified
✅ WireGuard selected as replacement
✅ WireGuard installed on all three nodes
✅ wg0 configured
✅ Peer configuration completed
✅ Node1 ↔ Node2 connectivity verified
✅ Node1 ↔ Node3 connectivity verified
✅ Node2 ↔ Node3 connectivity verified
✅ WireGuard handshake verified
✅ WireGuard configured for automatic startup
✅ Reboot persistence verified
✅ Old GRE configuration/tunnels cleaned up
Production network decision:

Due to GRE not being usable through the VPS provider network, the Proxmox cluster uses WireGuard as the final encrypted private overlay between node1, node2 and node3.

Also we have tried for the GRE but due to provider network limitation cant able to do for reference attache beloe


Section 6 — VM / Cloud-Init / Paymenter Deployment
6.1 — VM Network Planning and Internal Workload Network
The VM networking stage established the separation between the public-facing bridge and the internal workload network used for test/customer VM deployment. The project uses vmbr0 for the public/uplink side and vmbr2 for the internal 10.10.10.x workload network.
Project note: The exact provider-assigned VLAN IDs, public IP ranges, gateways, and physical interfaces are deployment-specific. The internal 10.10.10.x values documented here are the values used for the project test environment.
The intended traffic model is:
                   Proxmox Cluster
                          |
             +------------+------------+
             |                         |
          vmbr0                      vmbr2
       Public/Uplink             Internal VM Network
             |                         |
      Provider network          10.10.10.0/24
                                       |
                              VM / Container workloads

6.2 — Test VM Creation and Initial Network Configuration
A test VM was used to validate the internal workload network before continuing with the Paymenter deployment. Test VM 101 was configured with an internal address in the 10.10.10.x range.
The initial configuration exposed a bridge/IP mismatch: the VM used vmbr0 while its address and gateway belonged to the vmbr2 internal network. This was later corrected in Section 6.8.
The test values were:
Parameter
Project Test Value
VM ID
101
VM IP
10.10.10.50
Gateway
10.10.10.1
Intended Bridge
vmbr2
Network Type
Internal VM network


6.3 — Internal VM Routing and Internet Access Design
The internal VM network is not intended to become the Proxmox node's default route. Instead, internal VMs can use the Proxmox node as their routing/NAT point when internet access is required.
The design is:
Internal VM
10.10.10.x
     |
     v
   vmbr2
     |
     v
 Proxmox Node
     |
     | NAT / forwarding
     v
 Public/Uplink interface
     |
     v
 Internet
This design was validated through the test VM. The actual forwarding and NAT commands are documented in Section 6.8 so the implementation history remains in chronological order.

6.4 — Cloud-Init Template Preparation
Cloud-Init templates are required for the planned automated VPS provisioning workflow. The templates provide a reusable base image from which customer VMs can later be cloned and configured automatically.
The project prepared separate templates for the required Linux operating systems. Each template uses a unique VM ID and includes a Cloud-Init drive (ide2) before being converted into a Proxmox template.
The general template workflow is:
Cloud image
    ↓
Create temporary VM
    ↓
Import disk into local-lvm
    ↓
Attach Cloud-Init drive
    ↓
Configure boot / serial / guest agent
    ↓
Validate template
    ↓
Convert VM to template

6.5 — Debian 12 Cloud-Init Template
The Debian 12 Cloud-Init template uses VM ID 9000. The template is intended to be the base image for Debian VPS offerings in the future automated provisioning workflow.
Example project commands:
wget https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-generic-amd64.qcow2

qm create 9000 --name debian12-template --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 9000 debian-12-generic-amd64.qcow2 local-lvm
qm set 9000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0
qm set 9000 --ide2 local-lvm:cloudinit
qm set 9000 --boot c --bootdisk scsi0
qm set 9000 --serial0 socket --vga serial0
qm set 9000 --agent enabled=1
qm template 9000
Template rule: The VM must be fully validated before converting it to a template because qm template changes the VM into a reusable template object.

6.6 — Ubuntu 22.04 Cloud-Init Template
An Ubuntu 22.04 Cloud-Init template uses VM ID 9001. It provides a second OS offering for the planned VPS service.
Example project commands:
wget https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

qm create 9001 --name ubuntu2204-template --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0
qm importdisk 9001 jammy-server-cloudimg-amd64.img local-lvm
qm set 9001 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9001-disk-0
qm set 9001 --ide2 local-lvm:cloudinit
qm set 9001 --boot c --bootdisk scsi0
qm set 9001 --serial0 socket --vga serial0
qm set 9001 --agent enabled=1
qm template 9001
The two template IDs are reserved for the future provisioning layer:
Template
VM ID
Intended Use
Debian 12
9000
Debian-based VPS offering
Ubuntu 22.04
9001
Ubuntu-based VPS offering

Automation note: The final provisioning system will select the appropriate template according to the customer's selected OS/product. IP allocation, resource configuration, Cloud-Init network injection, VM creation, and customer delivery remain later automation stages.

6.7 — Step 5: IP Pool / Customer IP Allocation — Planned
The customer VM IP pool is part of the planned cloud-provider automation architecture.
Only IP addresses genuinely allocated/routed to the infrastructure by the VPS/network provider should be placed into the production customer IP pool.
The final implementation will maintain IP availability and assignment so that an automated provisioning workflow can:
Select a free IP.
Associate the IP with the customer/order.
Provision the VM from the selected Cloud-Init template.
Inject the required network configuration.
Mark the IP as assigned.
Return the VM/IP details to the customer through the billing platform.

6.8 — Internal Test VM Network Correction and NAT Configuration
During the test VM setup, a networking mismatch was identified.
The VM had its net0 interface connected to vmbr0 (public network), while the VM was assigned an IP from the internal vmbr2 network:
VM IP: 10.10.10.50
Gateway: 10.10.10.1
Bridge: vmbr0
This caused the VM to have an internal-network IP while being attached to the public bridge. As a result, the configured gateway was not reachable and the VM could not access the internet. The DNS resolution failure was a consequence of the missing external route rather than a standalone DNS configuration problem.
6.8.1 Move Test VM to the Internal Bridge
From the Node1 shell:
qm stop 101
qm set 101 --net0 virtio,bridge=vmbr2
qm start 101
The VM is now attached to vmbr2, matching the 10.10.10.x internal network.
6.8.2 Check the Host Default Route
First identify the interface used by Node1 for the public/default route:
ip route show default
The interface returned by this command is used for the NAT configuration below.
6.8.3 Enable IPv4 Forwarding and Configure NAT
Enable IPv4 forwarding:
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
Configure masquerading for the internal VM network:
iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o <interface_name> -j MASQUERADE
Replace <interface_name> with the actual interface returned by:
ip route show default
6.8.4 Make the NAT Rule Persistent
apt install -y iptables-persistent
netfilter-persistent save
6.8.5 Test Internet Access from the VM
From the test VM:
sudo apt update
This verifies whether the internal VM can reach the internet through Node1's public connectivity.
Architecture note: This NAT configuration is intended to provide internet access to internal VMs using the Node's public connectivity. The same design can later be used for customer VMs deployed on the internal network, subject to the final production network/security design.
6.9 — Paymenter Database Credential Verification
The Paymenter database credentials were reviewed during the installation process.
MariaDB does not provide the plaintext password for an existing user. The practical verification method is to attempt a login with the known password.
Run:
mysql -u paymenter -p paymenter
If the login succeeds and the MariaDB prompt appears:
MariaDB [paymenter]>
the credentials are valid.
If MariaDB returns an Access denied error, the password is incorrect or the expected user/database does not exist.
Note: Database password recovery/reset can be handled later if required. This was left for a later stage during the current deployment.
6.10 — Composer Installation for Paymenter
Composer was installed for the Paymenter application.
Install Composer:
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
Verify:
composer --version
6.11 — Download and Extract Paymenter
Create the Paymenter directory and download the release archive:
sudo mkdir -p /var/www/paymenter
cd /var/www/paymenter
sudo curl -Lo paymenter.tar.gz https://github.com/paymenter/paymenter/releases/latest/download/paymenter.tar.gz
sudo tar -xzvf paymenter.tar.gz
sudo chmod -R 755 storage/* bootstrap/cache/
The extracted application was checked to confirm that the Paymenter files were present.
Expected application structure included items such as:
.env.example
composer.json
app
public
vendor
The presence of the vendor directory indicated that dependencies were already bundled in the downloaded archive.
6.12 — Paymenter Environment Configuration
Create the environment file:
sudo cp .env.example .env
Verify/install application dependencies:
sudo composer install --no-dev --optimize-autoloader
If Composer displays the root-user/plugin safety warning, use:
sudo COMPOSER_ALLOW_SUPERUSER=1 composer install --no-dev --optimize-autoloader
Generate the application key:
sudo php artisan key:generate --force
Set the web-server permissions:
sudo chown -R www-data:www-data /var/www/paymenter
sudo chmod -R 755 /var/www/paymenter/storage /var/www/paymenter/bootstrap/cache
6.12.1 .env Line-Concatenation Error
During .env editing, a line concatenation problem occurred because a new variable was appended without a newline.
The affected content became:
REDIS_PORT=6379DB_CHARSET=utf8mb4
This caused REDIS_PORT to contain an invalid value instead of only the Redis port.
The correct format is:
REDIS_PORT=6379
DB_CHARSET=utf8mb4
The issue was addressed by editing the .env file directly.
6.12.2 Verify .env Variables
The relevant variables were checked using:
grep -E "REDIS_PORT|DB_CHARSET|DB_COLLATION" .env
The expected result is that the three values appear as separate lines.
6.12.3 Clear Configuration Cache
After correcting .env:
sudo php artisan config:clear
Then retry database migration:
sudo php artisan migrate --force
6.13 — Paymenter Scheduler and Queue Worker
The Paymenter scheduler was configured through cron.
The existing cron list was checked:
sudo crontab -l
An accidental duplicate entry was found:
* * * * * php /var/www/paymenter/artisan schedule:run >> /dev/null 2>&1
* * * * * php /var/www/paymenter/artisan schedule:run >> /dev/null 2>&1
The duplicate entry was cleaned using:
sudo crontab -l | sort -u | sudo crontab -
Verify:
sudo crontab -l
The Paymenter queue worker service was checked:
sudo systemctl status paymenter.service
The queue worker was confirmed active (running) during the setup.
6.14 — Nginx Configuration for Paymenter
Create the Nginx site configuration:
sudo tee /etc/nginx/sites-available/paymenter.conf > /dev/null <<'EOF'
server {
    listen 80;
    server_name 10.10.10.100;
    root /var/www/paymenter/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }
}
EOF
Enable the Paymenter site:
sudo ln -s /etc/nginx/sites-available/paymenter.conf /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
Test the Nginx configuration:
sudo nginx -t
Restart Nginx:
sudo systemctl restart nginx
The Nginx configuration test completed successfully during the setup.
6.15 — Paymenter Web Access Test
After Nginx was configured, the Paymenter service was tested from the network.
From a machine that can reach the internal 10.10.10.x network, test:
curl -I http://10.10.10.100
A successful HTTP response indicates that Nginx and the Paymenter web application are responding.
The Paymenter web interface can also be opened using:
http://10.10.10.100
The Paymenter login screen should appear when the application is correctly accessible.
6.16 — Current Deployment Status
Component
Status
Test VM network mismatch identified
✅
Test VM moved from vmbr0 to vmbr2
✅
Node NAT configuration prepared
✅
Internal VM internet-access testing
✅
Composer installed
✅
Paymenter files downloaded/extracted
✅
Paymenter .env created
✅
Composer dependency verification
✅
Application key generation
✅
File permissions configured
✅
.env concatenation issue identified/corrected
✅
Paymenter cron duplicate identified/cleaned
✅
Paymenter queue worker
✅ Active
Nginx configuration
✅
nginx -t validation
✅ Successful
Paymenter web test
✅ Pending
Paymenter DB credential verification
⏳ Deferred
Final production IP pool
⏳ Pending
Automated billing → Proxmox provisioning
⏳ Pending

Project note: The current work establishes the internal VM networking and Paymenter application layer required for the planned cloud-service-provider architecture. The next stages can continue sequentially from here without changing the documented history of the implementation.
6.17 — Errors Encountered During This Implementation
Error 1 — Internal IP Assigned to Public Bridge
Problem: VM 101 was attached to vmbr0 but configured with an IP/gateway from the vmbr2 internal network.
Effect: The VM could not reach its configured gateway or the internet.
Resolution:
qm stop 101
qm set 101 --net0 virtio,bridge=vmbr2
qm start 101
NAT was then configured on Node1.
Error 2 — Temporary failure resolving 'deb.debian.org'
The VM produced repository resolution errors while running APT:
Err:3 https://deb.debian.org/debian bookworm InRelease
  Temporary failure resolving 'deb.debian.org'
Err:4 https://deb.debian.org/debian bookworm-updates InRelease
  Temporary failure resolving 'deb.debian.org'
Err:5 https://deb.debian.org/debian bookworm-backports InRelease
  Temporary failure resolving 'deb.debian.org'
Err:6 https://deb.debian.org/debian-security bookworm-security InRelease
  Temporary failure resolving 'deb.debian.org'
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
All packages are up to date.
W: Failed to fetch mirror+file:/etc/apt/mirrors/debian.list/dists/bookworm/InRelease  Temporary failure resolving 'deb.debian.org'
W: Failed to fetch mirror+file:/etc/apt/mirrors/debian.list/dists/bookworm-updates/InRelease  Temporary failure resolving 'deb.debian.org'
W: Failed to fetch mirror+file:/etc/apt/mirrors/debian-security.list/dists/bookworm-security/InRelease  Temporary failure resolving 'deb.debian.org'
W: Some index files failed to download. They have been ignored, or old ones used instead.
harsh@test-vm1:~$
The issue was traced back to the VM networking mismatch and lack of a working external route. The networking/NAT correction above was therefore performed before continuing.
Error 3 — .env Variable Concatenation
The following malformed line was created:
REDIS_PORT=6379DB_CHARSET=utf8mb4
This was corrected so each environment variable appears on its own line:
REDIS_PORT=6379
DB_CHARSET=utf8mb4
Error 4 — Duplicate Paymenter Cron Entry
The same scheduler entry appeared twice:
* * * * * php /var/www/paymenter/artisan schedule:run >> /dev/null 2>&1
* * * * * php /var/www/paymenter/artisan schedule:run >> /dev/null 2>&1
It was cleaned with:
sudo crontab -l | sort -u | sudo crontab -
6.18 — Next Implementation Stage
The next work will continue from the completed Paymenter and internal-network setup.
Planned sequence:
IP Pool / Customer IP Allocation
        ↓
Cloud-Init Template Finalization
        ↓
Proxmox API / Automation Layer
        ↓
Node Selection
        ↓
Automatic VM Creation
        ↓
Automatic IP Assignment
        ↓
Paymenter / Billing Order Integration
        ↓
End-to-End VPS Provisioning Test
The implementation will continue from this section in sequence, with each completed practical step and any errors encountered documented before moving to the next stage.

