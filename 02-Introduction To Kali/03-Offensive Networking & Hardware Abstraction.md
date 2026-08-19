## 1. WIRELESS STACK MANIPULATION & RFMON STATE ENGINEERING

### Managed vs. Monitor Mode (RFMON) Subsystem Mechanics

The IEEE 802.11 wireless stack in Linux operates across two primary operational modes at the MAC layer: **Managed Mode** (`NL80211_IFTYPE_STATION`) and **Monitor Mode** (`NL80211_IFTYPE_MONITOR`).

```
+-----------------------------------------------------------------------------------+
| USER SPACE: airmon-ng, iw, tshark, custom C injection engine                      |
+-----------------------------------------------------------------------------------+
                                   │
                                   ▼ Generic Netlink (NETLINK_GENERIC)
+-----------------------------------------------------------------------------------+
| KERNEL USER INTERFACE: net/wireless/nl80211.c                                     |
|  - Parses NL80211_CMD_SET_INTERFACE, NL80211_CMD_SET_CHANNEL                       |
+-----------------------------------------------------------------------------------+
                                   │
                                   ▼
+-----------------------------------------------------------------------------------+
| CORE WIRELESS SUBSYSTEM: net/wireless/core.c (cfg80211)                           |
|  - Validates regulatory domain compliance (CRDA)                                  |
|  - Coordinates channel selection and hardware state locking                       |
+-----------------------------------------------------------------------------------+
                                   │
                                   ▼
+-----------------------------------------------------------------------------------+
| MAC SOFTWARE LAYER: net/mac80211/ (mac80211)                                      |
|  - ieee80211_rx_monitor(): Constructs radiotap headers for all MPDUs              |
|  - ieee80211_tx_prepare(): Overrides association checks during frame injection    |
+-----------------------------------------------------------------------------------+
                                   │
                                   ▼ Driver Callbacks (struct ieee80211_ops)
+-----------------------------------------------------------------------------------+
| HARDWARE DRIVER / PHY LAYER: (e.g., drivers/net/wireless/realtek/rtl8812au)      |
|  - Sets hardware MAC filters, disables auto-ACKs, forces promiscuous RF capture  |
+-----------------------------------------------------------------------------------+

```

#### Operational Subsystem Comparison

* **Managed Mode (`NL80211_IFTYPE_STATION`):** The driver configures physical hardware filters to accept strictly unicast frames matching the interface's local MAC address (`net_device->dev_addr`) or broadcast/multicast frames belonging to the associated Basic Service Set Identifier (BSSID). Frames failing address validation are discarded by hardware or low-level driver ring buffers. The hardware MAC controller automatically handles Layer 2 ACKs, retransmissions, and frame sequence numbering.
* **Monitor Mode (`NL80211_IFTYPE_MONITOR`):** Hardware MAC address filtering is disabled. The wireless card captures all 802.11 MAC Protocol Data Units (MPDUs) passing across the physical medium regardless of BSSID, destination MAC, or CRC state. Automatic hardware-level ACK generation is disabled to prevent frame collisions during passive analysis. Captured frames are prepended with `radiotap` metadata before being delivered to user-space `AF_PACKET` sockets.

#### Kernel State Machine Transitions (`NL80211_IFTYPE_STATION` $\rightarrow$ `NL80211_IFTYPE_MONITOR`)

1. **User Request Processing:** User space sends an `NL80211_CMD_SET_INTERFACE` command over a `NETLINK_GENERIC` socket (`net/wireless/nl80211.c`).
2. **Subsystem Validation:** `nl80211_set_interface()` verifies that the underlying hardware (`struct wiphy`) supports monitor mode by checking `wiphy->interface_modes`.
3. **Core Reconfiguration:** `cfg80211_change_iface()` (`net/wireless/core.c`) locks the `cb_lock` mutex and brings the interface down internally if active.
4. **Driver Callback Invocation:** The kernel triggers the driver-specific `.change_virtual_intf` or `.add_virtual_intf` operation registered in `struct ieee80211_ops`.
5. **Hardware MAC Reconfiguration:** In `mac80211`, `ieee80211_reconfig_vif()` executes. It disables hardware-level BSSID matching, configures RX packet filters to retain control, management, and corrupted (bad FCS) frames, and installs `ieee80211_rx_monitor()` as the frame reception pipeline.

#### Radiotap Header Encapsulation

When operating in monitor mode, `mac80211` constructs a `radiotap` header (`struct ieee80211_radiotap_header`) prior to pushing the frame up through `netif_rx()`. This structure prepends raw physical-layer transmission metadata:

```c
/* Standard Radiotap Header Format passed to AF_PACKET Sockets */
struct ieee80211_radiotap_header {
    uint8_t  it_version;     /* Set to 0 */
    uint8_t  it_pad;         /* Alignment padding */
    uint16_t it_len;         /* Total length of radiotap header + extended fields */
    uint32_t it_present;     /* Bitmask of present fields (TSFT, Flags, Rate, Channel, RSSI) */
} __attribute__((packed));

```

The hardware driver populates present fields using physical PHY registers:

* **`IEEE80211_RADIOTAP_TSFT`:** 64-bit Microsecond Time Synchronization Function timer readout.
* **`IEEE80211_RADIOTAP_FLAGS`:** Frame integrity status (FCS errors, encrypted frame status, bad PLCP).
* **`IEEE80211_RADIOTAP_RATE`:** Physical data rate in units of 500 Kbps.
* **`IEEE80211_RADIOTAP_CHANNEL`:** Frequency in MHz (`channel.freq`) paired with channel flags (2GHz, 5GHz, OFDM, CCK).
* **`IEEE80211_RADIOTAP_DBM_ANTSIGNAL`:** Received Signal Strength Indicator (RSSI) expressed in dBm.

---

### Under the Hood of `airmon-ng` & `iw`

#### Interference Process Termination Mechanics

The utility `airmon-ng check kill` identifies and terminates user-space daemons that hold active netlink socket handles or manage system network interfaces.

| Daemon | Interfere Mechanism with RFMON & Frame Injection |
| --- | --- |
| **`NetworkManager`** | Continuously issues `NL80211_CMD_TRIGGER_SCAN` requests to identify managed networks, overriding manual channel lock settings and forcing the physical PHY off its target frequency. |
| **`wpa_supplicant`** | Re-asserts EAPOL state machines, issues authentication/association requests, and forces the driver out of monitor mode back to `NL80211_IFTYPE_STATION` state. |
| **`dhclient` / `systemd-resolved**` | Continuously attempts DHCP lease acquisition and link state renewal over the interface, generating unwanted IP-layer traffic and re-initializing dynamic network link parameters. |
| **`iwd`** | Directly conflicts with `cfg80211` state variables by continually maintaining internal BSS scan indexes and interface state parameters. |

#### Low-Level Interface Manipulation via `iw` (nl80211) vs Legacy `iwconfig` (WEXT)

Legacy `iwconfig` utilities communicate via the obsolete Wireless Extensions (WEXT) `ioctl` interface (`/proc/net/wireless`), which uses fixed structural parameters that cannot support modern 802.11ac/ax/be channel configurations or complex netlink events. Modern interface control relies strictly on `iw` via the `nl80211` generic netlink family.

```bash
# 1. Bring the wireless interface down to modify MAC state variables
sudo ip link set dev wlan0 down

# 2. Reconfigure interface type to Monitor Mode via nl80211
sudo iw dev wlan0 set type monitor

# 3. Re-enable the network interface link
sudo ip link set dev wlan0 up

# 4. Lock physical PHY to a specific 2.4 GHz channel (e.g., Channel 6 / 2437 MHz)
sudo iw dev wlan0 set channel 6

# 5. Lock physical PHY to a 5 GHz channel with 80 MHz channel width
sudo iw dev wlan0 set channel 36 80MHz

```

---

### Channel Hopping Locks & Frequency Selection

#### Channel Hopping Implementation Architecture

User-space wireless sniffers implement channel hopping by executing a cyclic timer loop that issues `NL80211_CMD_SET_CHANNEL` netlink commands.

```c
/* Conceptual Channel Hopping Loop using Netlink Socket Commands */
void channel_hop_loop(struct nl_msg *msg, struct nl_sock *sk, int *channels, int count) {
    int index = 0;
    while (1) {
        int target_freq = channels[index];
        
        /* Reconstruct Netlink message to set channel frequency */
        nla_put_u32(msg, NL80211_ATTR_WIPHY_FREQ, target_freq);
        nla_put_u32(msg, NL80211_ATTR_CHANNEL_WIDTH, NL80211_CHAN_WIDTH_20);
        
        /* Send NL80211_CMD_SET_CHANNEL request to kernel */
        nl_send_auto(sk, msg);
        
        index = (index + 1) % count;
        usleep(100000); /* Hop dwell time: 100 milliseconds */
    }
}

```

#### Spectrum Frequency & Width Specifications

* **2.4 GHz Band:** 14 channels (2412 MHz – 2484 MHz), spaced 5 MHz apart (except Channel 14). Standard operational widths are 20 MHz or 40 MHz (HT40+/HT40-).
* **5 GHz Band:** UNII-1 through UNII-3 (5180 MHz – 5825 MHz). Supports 20 MHz, 40 MHz (VHT40), 80 MHz (VHT80), and 160 MHz (VHT160) channel widths.
* **6 GHz Band (Wi-Fi 6E/7):** UNII-5 through UNII-8 (5955 MHz – 7115 MHz). Supports up to 320 MHz contiguous channel bandwidths (`NL80211_CHAN_WIDTH_320`).

#### Resolving Driver Frequency Locks & Freeze Conditions

When a wireless driver freezes due to kernel lockup or invalid CRDA state transitions during channel hopping, execute the following sequence to unbind and reset the hardware device without rebooting:

```bash
# Identify the PCI device address of the target wireless adapter
lspci -D | grep -i network
# Output: 0000:03:00.0 Network controller: Realtek Semiconductor Corp...

# Forcefully unbind the driver from the physical PCI bus
echo "0000:03:00.0" | sudo tee /sys/bus/pci/drivers/rtl8812au/unbind

# Wait for kernel hardware tear-down completion
sleep 1

# Re-bind the physical device to trigger a clean driver re-initialization
echo "0000:03:00.0" | sudo tee /sys/bus/pci/drivers/rtl8812au/bind

```

---

## 2. INTERFACE & IDENTITY CONTROL, MAC SPOOFING & NETWORK NAMESPACES

### MAC Address Spoofing & Kernel Interface State

#### `struct net_device` State Management in Kernel Memory

Every network interface registered within the Linux kernel is represented by a `struct net_device` instance (`include/linux/netdevice.h`). MAC address resolution is maintained via two distinct fields inside this structure:

* **`dev_addr`:** Pointer to the active, operational MAC address currently utilized by the kernel stack to frame outgoing Layer 2 headers and filter incoming frames.
* **`perm_addr`:** Read-only memory array storing the permanent Burned-In Address (BIA) extracted from the physical device's EEPROM or internal flash memory during driver probe initialization (`register_netdevice()`).

```c
/* Kernel driver MAC update handler execution (net/core/dev.c) */
int dev_set_mac_address(struct net_device *dev, struct sockaddr *sa, struct netlink_ext_ack *extack) {
    const struct net_device_ops *ops = dev->netdev_ops;
    
    /* Verify if the underlying driver provides a MAC modification callback */
    if (!ops->ndo_set_mac_address) {
        return -EOPNOTSUPP;
    }
    
    /* Ensure interface is down if the driver does not support live updates */
    if (netif_running(dev) && !(dev->priv_flags & IFF_LIVE_ADDR_CHANGE)) {
        return -EBUSY;
    }
    
    /* Execute driver callback to write new MAC to dev->dev_addr and hardware registers */
    return ops->ndo_set_mac_address(dev, sa);
}

```

#### ARP Cache & Switch Port Table Poisoning Mitigation

When modifying `dev_addr`, network switches continue routing unicast frames to the old MAC until the switch's CAM (Content Addressable Memory) table updates its port association.

To force immediate upstream switch update upon address rotation, the system must broadcast an Unsolicited (Gratuitous) ARP frame:

```bash
# 1. Bring down the interface link to prevent race conditions with NetworkManager
sudo ip link set dev eth0 down

# 2. Modify operational MAC address in kernel memory
sudo ip link set dev eth0 address 00:11:22:33:44:55

# 3. Re-enable the interface link
sudo ip link set dev eth0 up

# 4. Transmit Gratuitous ARP to update upstream switch port MAC tables
sudo arping -c 3 -A -I eth0 192.168.1.100

```

---

### Network Namespace Isolation (`ip netns`)

Network namespaces (`CLONE_NEWNET`) isolate network state across processes.

```
+-----------------------------------------------------------------------------------+
| DEFAULT NAMESPACE (PID 1 Context)                                                 |
|  - Interfaces: lo, eth0, wlan0                                                    |
|  - Routing Table: Default Gateway -> 192.168.1.1                                  |
|  - Netfilter: Standard Host Firewall Rules                                        |
+-----------------------------------------------------------------------------------+
                                         │
                                         │ Move Physical Device (wlan1)
                                         ▼
+-----------------------------------------------------------------------------------+
| ISOLATED OPERATIONAL NAMESPACE: ops_ns                                            |
|  - Interfaces: lo, wlan1 (Physical Adapter), veth_ops0                            |
|  - Routing Table: Completely Isolated (VPN or Custom Direct Route)                 |
|  - Netfilter: Isolated / Empty Table Context                                      |
|  - Procfs Scope: /proc/net Isolated                                              |
+-----------------------------------------------------------------------------------+

```

#### Subsystem Boundary Isolation Scope

1. **Network Interfaces:** Physical (`wlan1`) and virtual (`veth`) interfaces assigned to a namespace are visible exclusively to processes executing inside that namespace context.
2. **Routing Tables:** Independent Forwarding Information Base (FIB) rules, default gateways, and custom table identifiers.
3. **Socket & Port Spaces:** Binding to TCP port 80 inside a namespace does not conflict with TCP port 80 bound in the default root namespace.
4. **Netfilter / Firewall Rules:** Completely independent `iptables` and `nftables` chains and hooks.
5. **`/proc/net` Pseudofs:** Process readouts for `/proc/net/dev`, `/proc/net/tcp`, and `/proc/net/arp` reflect only namespace-bound resources.

#### Step-by-Step Namespace Provisioning Script

```bash
# 1. Create a new network namespace named 'ops_ns'
sudo ip netns add ops_ns

# 2. Instantiate a virtual ethernet pair (veth) for inter-namespace communication
sudo ip link add veth_root type veth peer name veth_ops0

# 3. Move the peer veth interface into the 'ops_ns' namespace
sudo ip link set veth_ops0 netns ops_ns

# 4. Move a physical secondary wireless card (wlan1) directly into the operational namespace
sudo ip link set wlan1 netns ops_ns

# 5. Configure IP addressing inside the isolated namespace
sudo ip netns exec ops_ns ip addr add 10.200.1.2/24 dev veth_ops0
sudo ip addr add 10.200.1.1/24 dev veth_root

# 6. Bring up loopback and target interfaces inside the namespace
sudo ip netns exec ops_ns ip link set dev lo up
sudo ip netns exec ops_ns ip link set dev veth_ops0 up
sudo ip netns exec ops_ns ip link set dev wlan1 up
sudo ip link set dev veth_root up

# 7. Set default routing path inside the isolated namespace
sudo ip netns exec ops_ns ip route add default via 10.200.1.1

# 8. Execute an isolated interactive shell inside the clean operational environment
sudo ip netns exec ops_ns /bin/bash

```

---

### IP Routing Tables & Policy-Based Routing

Standard Linux routing relies on a single default table. **Policy-Based Routing (PBR)** enables multi-homed traffic routing driven by source IP addresses, incoming interface bindings, or netfilter firewall marks (`fwmark`).

```
                              [ Incoming Packet / Socket Output ]
                                               │
                                               ▼
                                  [ FIB Rule Database (ip rule) ]
                                               │
                       ┌───────────────────────┴───────────────────────┐
                       │                                               │
                       ▼ (Match: fwmark 0x10)                          ▼ (Match: Default Fallback)
          [ Table 200: VPN_Table ]                            [ Table 254: Main Table ]
                       │                                               │
                       ▼                                               ▼
          (Route out via tun0 / VPN)                      (Route out via Default Gateway)

```

#### Multi-Homed Routing Table Provisioning

```bash
# 1. Register a custom routing table alias in /etc/iproute2/rt_tables
echo "200 ops_vpn_table" | sudo tee -a /etc/iproute2/rt_tables

# 2. Add default route to the custom table targeting a VPN tunnel interface (tun0)
sudo ip route add default via 10.8.0.1 dev tun0 table ops_vpn_table

# 3. Define PBR rule forcing traffic marked with fwmark 0x10 through the custom table
sudo ip rule add fwmark 0x10 table ops_vpn_table

# 4. Define PBR rule forcing traffic originating from a specific subnet through the custom table
sudo ip rule add from 192.168.100.0/24 table ops_vpn_table

# 5. Mark specific application traffic using nftables for selective VPN routing
sudo nft add table ip mangle
sudo nft add chain ip mangle output { type route hook output priority -150 \; }
sudo nft add rule ip mangle output skuid "kali" meta mark set 0x10

# 6. Flush route cache to enforce instantly
sudo ip route flush cache

```

---

## 3. RAW SOCKETS, PROMISCUOUS MODE & LAYER 2/3 FRAME INJECTION

### Interface Binding & Promiscuous Mode Operations

#### `IFF_PROMISC` Socket Flags Mechanics

Setting the promiscuous mode flag (`IFF_PROMISC`) on a network device instructs the physical Network Interface Card (NIC) to disable its hardware address filtering engine.

```
+-----------------------------------------------------------------------------------+
| RAW INGRESS FRAME (Destination MAC: AA:BB:CC:DD:EE:FF)                            |
+-----------------------------------------------------------------------------------+
                                   │
                                   ▼
+-----------------------------------------------------------------------------------+
| KERNEL INGRESS PIPELINE: net/core/dev.c (__netif_receive_skb_core)                |
|                                                                                   |
|  if (skb->pkt_type == PACKET_OTHERHOST) {                                         |
|      if (dev->flags & IFF_PROMISC) {                                              |
|          /* Retain frame and pass to raw packet taps (AF_PACKET) */              |
|          deliver_to_raw_sockets(skb);                                             |
|      } else {                                                                     |
|          /* Discard frame at Layer 2 boundary */                                 |
|          kfree_skb(skb);                                                          |
|      }                                                                            |
|  }                                                                                |
+-----------------------------------------------------------------------------------+

```

When `IFF_PROMISC` is enabled via `SIOCSIFFLAGS` `ioctl` calls, `dev->promiscuity` counter increments. All valid link-layer frames reaching the hardware MAC are accepted and delivered to packet taps registered with `ptype_all` (`net/packet/af_packet.c`).

---

### Layer 2 Raw Frame Crafting in C

The following complete, compilable C application demonstrates low-level `AF_PACKET` socket creation, interface index resolution using `if_nametoindex()`, explicit binding via `struct sockaddr_ll`, Ethernet frame assembly (`struct ethhdr`), and raw frame injection onto the physical wire.

```c
/*
 * Compiled with: gcc -O2 -Wall raw_l2_inject.c -o raw_l2_inject
 * Execution: sudo ./raw_l2_inject eth0
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <sys/ioctl.h>
#include <arpa/inet.h>
#include <net/if.h>
#include <net/ethernet.h>
#include <linux/if_packet.h>

#define PAYLOAD_LEN 64

int main(int argc, char *argv[]) {
    if (argc < 2) {
        fprintf(stderr, "Usage: %s <interface>\n", argv[0]);
        exit(EXIT_FAILURE);
    }

    const char *iface_name = argv[1];

    /* 1. Instantiate AF_PACKET raw socket catching all protocol types */
    int sock_fd = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    if (sock_fd < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* 2. Resolve interface index from interface string name */
    unsigned int if_index = if_nametoindex(iface_name);
    if (if_index == 0) {
        perror("if_nametoindex");
        close(sock_fd);
        exit(EXIT_FAILURE);
    }

    /* 3. Allocate buffer for total frame (Ethernet Header + Payload) */
    size_t frame_len = sizeof(struct ethhdr) + PAYLOAD_LEN;
    uint8_t *frame_buffer = calloc(1, frame_len);
    if (!frame_buffer) {
        perror("calloc");
        close(sock_fd);
        exit(EXIT_FAILURE);
    }

    /* 4. Construct Ethernet Link-Layer Header */
    struct ethhdr *eth = (struct ethhdr *)frame_buffer;
    
    /* Target Destination MAC: Broadcast (FF:FF:FF:FF:FF:FF) */
    memset(eth->h_dest, 0xFF, ETH_ALEN);
    
    /* Source MAC: Arbitrary Spoofed MAC (00:11:22:33:44:55) */
    eth->h_source[0] = 0x00;
    eth->h_source[1] = 0x11;
    eth->h_source[2] = 0x22;
    eth->h_source[3] = 0x33;
    eth->h_source[4] = 0x44;
    eth->h_source[5] = 0x55;
    
    /* Ethernet Type: Custom / Experimental (0x88B5) */
    eth->h_proto = htons(0x88B5);

    /* 5. Populate Payload Buffer with Pattern */
    uint8_t *payload = frame_buffer + sizeof(struct ethhdr);
    memset(payload, 0xA5, PAYLOAD_LEN);

    /* 6. Populate Link-Layer Socket Address Structure */
    struct sockaddr_ll socket_address;
    memset(&socket_address, 0, sizeof(struct sockaddr_ll));
    socket_address.sll_family   = AF_PACKET;
    socket_address.sll_protocol = htons(ETH_P_ALL);
    socket_address.sll_ifindex  = if_index;
    socket_address.sll_halen    = ETH_ALEN;
    memset(socket_address.sll_addr, 0xFF, ETH_ALEN);

    /* 7. Inject Raw Layer 2 Frame onto the wire */
    ssize_t bytes_sent = sendto(sock_fd, frame_buffer, frame_len, 0,
                                (struct sockaddr *)&socket_address, sizeof(socket_address));
    
    if (bytes_sent < 0) {
        perror("sendto");
    } else {
        printf("[+] Successfully injected %ld byte Layer 2 frame on interface %s (Index: %d)\n",
               bytes_sent, iface_name, if_index);
    }

    /* Cleanup resources */
    free(frame_buffer);
    close(sock_fd);
    return 0;
}

```

---

### Layer 3 Raw Sockets & Packet Synthesis

When injecting Layer 3 packets using `AF_INET` raw sockets, setting the `IP_HDRINCL` socket option informs the kernel that the application provides its own custom IP header.

#### Raw Socket Ingress & Egress Data Paths

```
                                +---------------------------+
                                |  USER SPACE APPLICATION   |
                                |  Constructs Custom IP &   |
                                |     TCP Header Buffer     |
                                +---------------------------+
                                              │
                                              ▼ sendto()
+-----------------------------------------------------------------------------------+
| KERNEL LAYER 3 (IP SUBSYSTEM): net/ipv4/raw.c (raw_sendmsg)                       |
|                                                                                   |
|  - Validates IP_HDRINCL option status                                            |
|  - Bypasses local netfilter raw / mangle output chains                            |
|  - Skips internal IP header auto-generation                                       |
+-----------------------------------------------------------------------------------+
                                              │
                                              ▼
+-----------------------------------------------------------------------------------+
| KERNEL LAYER 2 NEIGHBOR SUBSYSTEM: net/core/dev.c (dev_queue_xmit)                |
|  - Resolves destination MAC via local ARP cache                                   |
|  - Appends hardware Ethernet header                                               |
|  - Pushes frame buffer directly to network interface transmit queue (TX Ring)     |
+-----------------------------------------------------------------------------------+
                                              │
                                              ▼
                                 +-------------------------+
                                 | HARDWARE NIC TRANSMIT   |
                                 +-------------------------+

```

#### Internet Checksum Routine Implementation

The IP and TCP header checksum fields require manual 1's complement summation over 16-bit words.

$$\text{Checksum} = \sim \left( \sum_{i=1}^{N} \text{Word}_i \pmod{2^{16}-1} \right)$$

```c
/* Standard 1's Complement Internet Checksum Algorithm */
uint16_t compute_ip_checksum(uint16_t *addr, size_t len) {
    uint32_t sum = 0;
    while (len > 1) {
        sum += *addr++;
        len -= 2;
    }
    /* Add left-over byte if odd length */
    if (len > 0) {
        sum += *(uint8_t *)addr;
    }
    /* Fold 32-bit sum to 16 bits */
    while (sum >> 16) {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }
    return (uint16_t)(~sum);
}

```

#### Custom TCP/IP Packet Synthesis Code Snippet

```c
#include <sys/socket.h>
#include <netinet/ip.h>
#include <netinet/tcp.h>
#include <arpa/inet.h>

void inject_custom_tcp_syn(int sock_fd, struct sockaddr_in *target) {
    char packet[sizeof(struct iphdr) + sizeof(struct tcphdr)];
    memset(packet, 0, sizeof(packet));

    struct iphdr *iph = (struct iphdr *)packet;
    struct tcphdr *tcph = (struct tcphdr *)(packet + sizeof(struct iphdr));

    /* Populate Custom IP Header (struct iphdr) */
    iph->ihl      = 5;
    iph->version  = 4;
    iph->tos      = 0;
    iph->tot_len  = htons(sizeof(packet));
    iph->id       = htons(54321);
    iph->frag_off = 0;
    iph->ttl      = 64;
    iph->protocol = IPPROTO_TCP;
    iph->saddr    = inet_addr("192.168.1.50"); /* Spoofed Source IP */
    iph->daddr    = target->sin_addr.saddr;
    iph->check    = compute_ip_checksum((uint16_t *)iph, sizeof(struct iphdr));

    /* Populate Custom TCP Header (struct tcphdr) */
    tcph->source  = htons(12345);
    tcph->dest    = target->sin_port;
    tcph->seq     = htonl(1000);
    tcph->ack_seq = 0;
    tcph->doff    = 5; /* 20 bytes header size */
    tcph->syn     = 1; /* Set SYN Flag */
    tcph->window  = htons(5840);
    tcph->check   = 0; /* Pseudo-header checksum calculated separately */

    /* Enable IP_HDRINCL option on socket */
    int one = 1;
    setsockopt(sock_fd, IPPROTO_IP, IP_HDRINCL, &one, sizeof(one));

    /* Send Synthesized Packet */
    sendto(sock_fd, packet, iph->tot_len, 0, (struct sockaddr *)target, sizeof(*target));
}

```

---

## 4. SOURCES & REFERENCES

1. **Linux Kernel Core Network Device Subsystem:**
* Path: `net/core/dev.c`
* URL: [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/dev.c](https://www.google.com/search?q=https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/core/dev.c)
* *Relevance:* Implementation details for `__netif_receive_skb_core()`, `dev_set_mac_address()`, and promiscuous mode processing.


2. **Linux Kernel Wireless Subsystem Configuration Source:**
* Path: `net/wireless/core.c` & `net/wireless/nl80211.c`
* URL: [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/wireless/](https://www.google.com/search?q=https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/wireless/)
* *Relevance:* Mechanics of `cfg80211`, `nl80211` generic netlink command handlers, and monitor mode state machine transitions.


3. **Linux Kernel Packet Socket Subsystem Source:**
* Path: `net/packet/af_packet.c`
* URL: [https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/packet/af_packet.c](https://www.google.com/search?q=https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/tree/net/packet/af_packet.c)
* *Relevance:* Low-level socket creation, `sockaddr_ll` structure handling, and `ptype_all` packet tap hooks.


4. **Linux Network Namespaces Manual Page (`ip-namespace(8)`):**
* Author: Linux IPRoute2 Maintainers
* URL: [https://man7.org/linux/man-pages/man8/ip-netns.8.html](https://man7.org/linux/man-pages/man8/ip-netns.8.html)
* *Relevance:* Commands and isolation primitives for network namespace management.


5. **Packet Socket Manual Page (`packet(7)`):**
* Author: Linux Man-Pages Project
* URL: [https://man7.org/linux/man-pages/man7/packet.7.html](https://man7.org/linux/man-pages/man7/packet.7.html)
* *Relevance:* Specifications for `AF_PACKET`, `SOCK_RAW`, `SOCK_DGRAM`, and physical ring buffer parameters.


6. **IEEE 802.11-2020 Standard Specification:**
* Organization: IEEE Computer Society
* URL: [https://standards.ieee.org/ieee/802.11/7028/](https://standards.ieee.org/ieee/802.11/7028/)
* *Relevance:* Definitions of 802.11 MAC frame formats, radiotap headers, and physical channel specifications.