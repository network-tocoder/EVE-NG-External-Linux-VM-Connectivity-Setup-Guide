# EVE-NG External Linux VM Connectivity Setup Guide

Complete step-by-step guide to configure EVE-NG with external Linux VM for internet connectivity and lab networking.

## Architecture Overview

```
[EVE-NG Devices] <--> [Pnet1: 20.0.0.1] <--> [VMnet10] <--> [Linux VM ens34: 20.0.0.10]
                                                         <--> [Linux VM ens33: 192.168.1.17] <--> [Internet]
```

## Use Cases and Business Requirements



## Phase 1: EVE-NG Initial Configuration

### Step 1.1: Configure EVE-NG Static IP (Optional)

Run EVE-NG first-time setup:

```bash
# Reset EVE-NG configuration
rm -f /opt/ovf/.configured
su -

# Follow the blue screen GUI to configure:
# - Management IP (e.g., 192.168.1.200/24)
# - Gateway, DNS settings
```

**Verification:**
```bash
# Check EVE-NG management interface
ip addr show pnet0
# Expected: Should show your configured management IP

# Test internet connectivity
ping -c 3 8.8.8.8
# Expected: Should succeed
```

### Step 1.2: Configure Pnet1 Interface

Create Pnet1 network configuration:

```bash
sudo vim /etc/netplan/99-pnet.yaml
```

Add configuration:
```yaml
network:
  version: 2
  bridges:
    pnet1:
      dhcp4: false
      addresses:
        - 20.0.0.1/24
      parameters:
        stp: false
        forward-delay: 0
```

Apply configuration:
```bash
# Fix file permissions
sudo chmod 600 /etc/netplan/99-pnet.yaml

# Apply changes
sudo netplan apply
```

**Verification:**
```bash
# Check pnet1 interface (will show NO-CARRIER initially)
ip addr show pnet1
# Expected: Interface should exist with IP 20.0.0.1/24 but state DOWN/NO-CARRIER

# Check bridge status
brctl show
# Expected: Should show pnet1 bridge
```

---

## Phase 2: Linux VM Configuration

### Step 2.1: Configure Linux VM Network Interfaces

Check current interfaces:
```bash
# Identify available interfaces
ip addr show
# Expected: Should see at least ens33 (or similar) and possibly ens34
```

Configure static networking:
```bash
sudo vim /etc/netplan/01-netcfg.yaml
```

Add dual-interface configuration:
```yaml
network:
  version: 2
  ethernets:
    ens33:  # Connected to PC network
      dhcp4: false
      addresses:
        - 192.168.1.17/24  # Adjust to your network
      routes:
        - to: default
          via: 192.168.1.1  # Your router IP
      nameservers:
        addresses:
          - 8.8.8.8
          - 8.8.4.4
    ens34:  # Connected to EVE-NG network
      dhcp4: false
      addresses:
        - 20.0.0.10/24
```

Apply configuration:
```bash
# Fix permissions and apply
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
```

**Verification:**
```bash
# Check both interfaces
ip addr show ens33
ip addr show ens34
# Expected: ens33 should have 192.168.1.17/24, ens34 should have 20.0.0.10/24

# Test connectivity
ping -c 3 192.168.1.1    # Gateway
ping -c 3 8.8.8.8        # Internet
# Expected: Both should succeed
```

### Step 2.2: Enable IP Forwarding and NAT

Enable IP forwarding:
```bash
# Enable IP forwarding
echo 'net.ipv4.ip_forward=1' | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

Configure NAT and forwarding rules:
```bash
# Configure iptables for NAT
sudo iptables -t nat -A POSTROUTING -o ens33 -j MASQUERADE
sudo iptables -A FORWARD -i ens34 -o ens33 -j ACCEPT
sudo iptables -A FORWARD -i ens33 -o ens34 -m state --state RELATED,ESTABLISHED -j ACCEPT

# Save rules (Ubuntu/Debian)
sudo iptables-save | sudo tee /etc/iptables/rules.v4

# Install iptables-persistent to restore rules on boot
sudo apt update && sudo apt install -y iptables-persistent
```

**Verification:**
```bash
# Check IP forwarding
cat /proc/sys/net/ipv4/ip_forward
# Expected: Should show 1

# Check NAT rules
sudo iptables -t nat -L
# Expected: Should show MASQUERADE rule for ens33

# Check forward rules
sudo iptables -L FORWARD
# Expected: Should show ACCEPT rules for ens34 <-> ens33
```

---

## Phase 3: VMware Network Configuration

### Step 3.1: Configure VMnet10 Host-Only Network

In VMware Workstation:

1. **Open Virtual Network Editor:**
   - Edit → Virtual Network Editor
   - Run as Administrator (if prompted)

2. **Create/Configure VMnet10:**
   - Select VMnet10 (or click Add Network)
   - Type: **Host-only**
   - Subnet IP: **20.0.0.0**
   - Subnet mask: **255.255.255.0**
   - **Uncheck "Use local DHCP service"**
   - Apply changes

**Verification:**
```cmd
# On Windows host, check VMware network adapters
ipconfig
# Expected: Should see VMware Network Adapter VMnet10 with 20.0.0.1
```

### Step 3.2: Configure Linux VM Network Adapters

In VMware Linux VM settings:

1. **Network Adapter 1:**
   - Connection: **NAT** or **Bridged** (for internet access)
   - Maps to: ens33

2. **Network Adapter 2:**
   - Connection: **Host-only: VMnet10**
   - Maps to: ens34

**Verification:**
After VM restart:
```bash
# Check if ens34 has connectivity
ip addr show ens34
# Expected: Should show UP state with 20.0.0.10/24

# Test host-only network
ping -c 3 20.0.0.1  # VMware host adapter
# Expected: Should succeed
```

### Step 3.3: Configure EVE-NG VM Network Adapters

In VMware EVE-NG VM settings:

1. **Network Adapter 1:**
   - Connection: **NAT** or **Bridged** (for management)
   - Maps to: eth0/pnet0

2. **Network Adapter 2 (ADD THIS):**
   - Connection: **Host-only: VMnet10**
   - Maps to: eth1

**Verification:**
After EVE-NG VM restart:
```bash
# Check for new interface
ip addr show
# Expected: Should see eth1 interface

# Check current interfaces count
ip link show | grep -E '^[0-9]+:'
# Expected: Should show at least 4 interfaces (lo, eth0, pnet0, pnet1, eth1)
```

---

## Phase 4: Bridge EVE-NG to VMnet10

### Step 4.1: Bridge eth1 to pnet1

Update EVE-NG pnet1 configuration:
```bash
sudo vim /etc/netplan/99-pnet.yaml
```

Update with bridge configuration:
```yaml
network:
  version: 2
  bridges:
    pnet1:
      interfaces: [eth1]  # Bridge eth1 (connected to VMnet10)
      dhcp4: false
      addresses:
        - 20.0.0.1/24
      parameters:
        stp: false
        forward-delay: 0
```

Apply configuration:
```bash
sudo chmod 600 /etc/netplan/99-pnet.yaml
sudo netplan apply
```

**Verification:**
```bash
# Check pnet1 status - should now be UP
ip addr show pnet1
# Expected: Should show "state UP" instead of "NO-CARRIER" or "DOWN"

# Check bridge configuration
brctl show
# Expected: Should show eth1 connected to pnet1 bridge

# Test connectivity to Linux VM
ping -c 3 20.0.0.10
# Expected: Should succeed with response from Linux VM
```

---

## Phase 5: Test with Cisco Router

### Step 5.1: Create Test Topology in EVE-NG

1. **Access EVE-NG Web Interface:**
   - Navigate to `http://your-eve-ng-ip`
   - Create new lab

2. **Add Cisco Router:**
   - Add router (e.g., Cisco IOSv, CSR1000v)
   - Right-click router → Configure
   - Connect interface to **pnet1**

### Step 5.2: Configure Cisco Router

Console into the router and configure:
```cisco
Router> enable
Router# configure terminal

# Configure interface connected to pnet1
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip address 20.0.0.20 255.255.255.0
Router(config-if)# no shutdown
Router(config-if)# exit

# Set default route through Linux VM
Router(config)# ip route 0.0.0.0 0.0.0.0 20.0.0.10

# Save configuration
Router(config)# exit
Router# write memory
```

**Verification:**
```cisco
# Check interface status
Router# show ip interface brief
# Expected: GigabitEthernet0/0 should show up/up with IP 20.0.0.20

# Test local connectivity
Router# ping 20.0.0.1
# Expected: Should succeed (EVE-NG pnet1)

Router# ping 20.0.0.10
# Expected: Should succeed (Linux VM)

# Test internet connectivity
Router# ping 8.8.8.8
# Expected: Should succeed (Internet via Linux VM NAT)

Router# ping google.com
# Expected: Should succeed with DNS resolution
```

---

## Phase 6: Verification and Troubleshooting

### Step 6.1: Complete Network Verification

**From EVE-NG server:**
```bash
# Test all network segments
ping -c 3 20.0.0.10    # Linux VM
ping -c 3 20.0.0.20    # Cisco Router
ping -c 3 192.168.1.17 # Linux VM management (if routed)
```

**From Linux VM:**
```bash
# Test EVE-NG network connectivity
ping -c 3 20.0.0.1     # EVE-NG pnet1
ping -c 3 20.0.0.20    # Cisco Router

# Check routing table
ip route show
# Expected: Should show routes for both 20.0.0.0/24 and default via 192.168.1.1

# Test NAT functionality
sudo tcpdump -i ens33 icmp &
# Then ping from router to 8.8.8.8 and watch traffic
```

**From Cisco Router:**
```cisco
# Complete connectivity test
Router# ping 20.0.0.1      # EVE-NG
Router# ping 20.0.0.10     # Linux VM
Router# ping 192.168.1.1   # Router's gateway (via Linux VM)
Router# ping 8.8.8.8       # Internet
Router# traceroute 8.8.8.8 # Should show path via 20.0.0.10

# Check routing
Router# show ip route
# Expected: Should show default route via 20.0.0.10
```

### Step 6.2: Common Troubleshooting

**If pnet1 shows NO-CARRIER:**
```bash
# Check if eth1 is properly bridged
brctl show pnet1
ip addr show eth1

# Restart networking
sudo netplan apply
sudo systemctl restart systemd-networkd
```

**If router can't reach internet:**
```bash
# On Linux VM - check IP forwarding
cat /proc/sys/net/ipv4/ip_forward  # Should be 1

# Check iptables rules
sudo iptables -t nat -L POSTROUTING
sudo iptables -L FORWARD

# Check if packets are being forwarded
sudo tcpdump -i ens34 host 20.0.0.20
sudo tcpdump -i ens33 host 20.0.0.20
```

**If VMnet10 connectivity fails:**
```bash
# Check VMware network services (Windows host)
services.msc → Look for VMware services

# Restart VMware networking
net stop vmnetdhcp
net start vmnetdhcp
```

### Step 6.3: Performance Testing

Test network performance between components:
```bash
# From Linux VM to EVE-NG
iperf3 -c 20.0.0.1 -t 10

# Monitor network interfaces during testing
watch -n 1 'cat /proc/net/dev'
```

---

## Network Summary

After successful completion:

| Component | Interface | IP Address | Purpose |
|-----------|-----------|------------|---------|
| EVE-NG | pnet0 | 192.168.1.200/24 | Management |
| EVE-NG | pnet1 | 20.0.0.1/24 | Lab network |
| Linux VM | ens33 | 192.168.1.17/24 | Internet access |
| Linux VM | ens34 | 20.0.0.10/24 | EVE-NG connectivity |
| Cisco Router | Gi0/0 | 20.0.0.20/24 | Lab interface |
| VMware Host | VMnet10 | 20.0.0.1/24 | Bridge network |

## Traffic Flow

1. **Router → Internet:** 20.0.0.20 → 20.0.0.10 (NAT) → 192.168.1.17 → 192.168.1.1 → Internet
2. **Router → EVE-NG:** 20.0.0.20 → 20.0.0.1 (via VMnet10 bridge)
3. **Management:** 192.168.1.200 (EVE-NG) ↔ 192.168.1.17 (Linux VM)

---

### Visual Architecture Diagram

![Network Architecture](./diagrams/network-architecture.svg)

*For the SVG diagram above, save the network-architecture.svg file in a `diagrams/` folder*

## Repository Structure

```
eve-ng-lab-setup/
├── README.md                 # This guide
├── diagrams/
│   ├── network-architecture.svg  # Visual network diagram
│   └── topology-overview.png     # Additional topology diagrams
├── configs/
│   ├── eve-ng/
│   │   ├── 99-pnet.yaml     # EVE-NG netplan config
│   │   └── interfaces.txt    # Interface verification commands
│   ├── linux-vm/
│   │   ├── 01-netcfg.yaml   # Linux VM netplan config
│   │   ├── iptables.rules   # NAT and forwarding rules
│   │   └── sysctl.conf      # IP forwarding settings
│   └── cisco/
│       ├── router-basic.txt # Basic router configuration
│       └── verification.txt # Router verification commands
├── scripts/
│   ├── setup-eve-ng.sh     # EVE-NG setup automation
│   ├── setup-linux-vm.sh   # Linux VM setup automation
│   └── verify-connectivity.sh # Complete verification script
└── troubleshooting/
    ├── common-issues.md     # Common problems and solutions
    └── packet-capture.md    # Network debugging guide
```

## Contributing

1. Fork this repository
2. Create feature branch (`git checkout -b feature/improvement`)
3. Test your changes thoroughly
4. Commit changes (`git commit -am 'Add improvement'`)
5. Push to branch (`git push origin feature/improvement`)
6. Create Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Support

For issues and questions:
1. Check troubleshooting section
2. Verify each step's validation commands
3. Create issue with detailed error messages and configuration outputs