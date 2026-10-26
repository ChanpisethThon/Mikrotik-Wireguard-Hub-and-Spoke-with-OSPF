# Mikrotik Wireguard Hub and Spoke with OSPF

For 1 HQ + 3 sites, a clean MikroTik design is to use WireGuard as the tunnel and OSPF to dynamically advertise the LAN networks over the WireGuard interfaces.

- Mikrotik HQ is the hub
- Mikrotik WH01, WH02, WH03 are the spokes
- Many users or site
- OSPF dynamic route auto update when topology change or update
- Each user has their own Wireguard key pair
- Disable one user or site by disabling that user's Wireguard interface

based of the network structure:

| Router | LAN               | WireGuard      |
| ------ | ----------------- | -------------- |
| HQ     | `192.168.1.0/24`  | `10.10.1.1/30` |
| Site 1 | `192.168.10.0/24` | `10.10.1.0/30` |
| Site 2 | `192.168.20.0/24` | `10.10.2.0/30` |
| Site 3 | `192.168.30.0/24` | `10.10.3.0/30` |

- For 1 HQ + 3 sites, using a separate /30 WireGuard subnet per site is cleaner.

**WireGuard addressing:**

| Tunnel | HQ IP | Site IP | Subnet |
| - | - | - | - |
| HQ ↔ Site 1 | 10.10.1.1/30 | 10.10.1.2/30 | 10.10.1.0/30 |
| HQ ↔ Site 2 | 10.10.2.1/30 | 10.10.2.2/30 | 10.10.2.0/30 |
| HQ ↔ Site 3 | 10.10.3.1/30 | 10.10.3.2/30 | 10.10.3.0/30 |

This gives you three independent point-to-point WireGuard networks.

**Topology:**

```markdown
                    INTERNET
                       |
                 Public IP / DDNS
                       |
                  ┌───────────┐
                  │    HQ     │
                  │ MikroTik  │
                  │10.10.1.1  |
                  |10.10.2.1  |
                  |10.10.3.1  │
                  └─────┬─────┘
                    WG  │
             ┌──────────┼──────────┐
             │          │          │
          10.10.1.2  10.10.2.2  10.10.3.2
             │          │          │
        ┌────▼───┐ ┌────▼───┐ ┌────▼───┐
        │ Site 1 │ │ Site 2 │ │ Site 3 │
        │  .10.  │ │  .20.  │ │  .30.  │
        └────────┘ └────────┘ └────────┘
```

OSPF will automatically learn:

```markdown
HQ:
192.168.10.0/24
192.168.20.0/24
192.168.30.0/24

Site 1:
192.168.1.0/30
192.168.20.0/24
192.168.30.0/24

Site 2:
192.168.1.0/30
192.168.10.0/24
192.168.30.0/24

Site 3:
192.168.1.0/30
192.168.10.0/24
192.168.20.0/24
```

recommend using the loopback as the OSPF router ID and a stable address for management/routing.

**Addressing plan:**

|Router|Loopback|WireGuard P2P|LAN|
|-|-|-|-|
|HQ|192.168.255.1/32|10.10.1.1/30, 10.10.2.1/30, 10.10.3.1/30|192.168.1.0/24|
|Site 1|192.168.255.2/32|10.10.1.2/30|192.168.10.0/24|
|Site 2|192.168.255.3/32|10.10.2.2/30|192.168.20.0/24|
|Site 3|192.168.255.4/32|10.10.3.2/30|192.168.30.0/24|

## HQ WireGuard

**On HQ:**

You can use one **WireGuard interface** with three peers, while assigning the three /30 addresses to that interface:

```routeros
/interface/wireguard
add name=wg-sites listen-port=51820 mtu=1420

/ip/address
add address=10.10.1.1/30 interface=wg-sites comment="WG Site1"
add address=10.10.2.1/30 interface=wg-sites comment="WG Site2"
add address=10.10.3.1/30 interface=wg-sites comment="WG Site3"
```

Create a key pair:

```routeros
/interface/wireguard/print
```

Take the **public key** from HQ.

Then add the three peers:

```routeros
/interface/wireguard/peers
add interface=wg-sites \
    public-key="SITE1_PUBLIC_KEY" \
    allowed-address=10.10.1.2/32 \
    responder=yes \
    comment="Site1"

add interface=wg-sites \
    public-key="SITE2_PUBLIC_KEY" \
    allowed-address=10.10.2.2/32 \
    responder=yes \
    comment="Site2"

add interface=wg-sites \
    public-key="SITE3_PUBLIC_KEY" \
    allowed-address=10.10.3.2/32 \
    responder=yes \
    comment="Site3"
```

If the sites initiate the connection to HQ, you can leave `endpoint-address` and `endpoint-port` empty on the HQ side.

**Note:**

- HQ is the central WireGuard server and the 3 sites initiate connections to HQ, enable responder on the HQ peers.
- At sites, Don't enable responder in wireguard peer if the sites are the ones initiating the tunnel.
- persistent-keepalive=25 in wireguard peer at the sites is especially useful if the sites are behind NAT or have changing WAN IPs.

```markdown
             HQ
       WireGuard Responder
              ↑
       waits for connections
          /     |     \
         /      |      \
      Site1   Site2   Site3
       ↑        ↑       ↑
   initiates initiates initiates
```

## Site 1

```routeros
/interface/wireguard
add name=wg-hq listen-port=51820 mtu=1420

/ip/address
add address=10.10.1.2/24 interface=wg-hq comment="WG to HQ"
```

Add HQ as peer:

```routeros
/interface/wireguard/peers
add interface=wg-hq \
    public-key="HQ_PUBLIC_KEY" \
    endpoint-address=d8550dd96e2a.sn.mynetname.net \
    endpoint-port=51820 \
    allowed-address=10.10.1.1/32 \
    persistent-keepalive=25
```

Your Site 1 LAN:

```routeros
192.168.10.0/24
```

## Site 2

```routeros
/interface/wireguard
add name=wg-hq listen-port=51820 mtu=1420

/ip/address
add address=10.10.2.2/30 interface=wg-hq comment="WG to HQ"
```

Peer:

```routeros
/interface/wireguard/peers
add interface=wg-hq \
    public-key="HQ_PUBLIC_KEY" \
    endpoint-address=d8550dd96e2a.sn.mynetname.net \
    endpoint-port=51820 \
    allowed-address=10.10.2.1/32 \
    persistent-keepalive=25
```

LAN:

```markdown
192.168.20.0/24
```

## Site 3

```routeros
/interface/wireguard
add name=wg-hq listen-port=51820 mtu=1420

/ip/address
add address=10.10.3.2/30 interface=wg-hq comment="WG to HQ"
```

Peer:

```routeros
/interface/wireguard/peers
add interface=wg-hq \
    public-key="HQ_PUBLIC_KEY" \
    endpoint-address=d8550dd96e2a.sn.mynetname.net \
    endpoint-port=51820 \
    allowed-address=10.10.3.1/32 \
    persistent-keepalive=25
```

LAN:

```markdown
192.168.30.0/24
```

## Important WireGuard firewall rule

On **HQ**, allow WireGuard from the Internet:

```routeros
/ip/firewall/filter
add chain=input \
    action=accept \
    protocol=udp \
    dst-port=51820 \
    comment="Allow WireGuard"
```

On all routers, allow the WireGuard interface:

```routeros
/ip/firewall/filter
add chain=input \
    action=accept \
    in-interface=wg-sites \
    comment="Allow WireGuard traffic"
```

On sites, use:

```routeros
/ip/firewall/filter
add chain=input \
    action=accept \
    in-interface=wg-hq \
    comment="Allow WireGuard traffic"
```

For HQ forwarding between LANs:

```routeros
/ip/firewall/filter
add chain=forward \
    action=accept \
    in-interface=wg-sites \
    comment="Allow VPN forwarding"
```

and on sites:

```routeros
/ip/firewall/filter
add chain=forward \
    action=accept \
    in-interface=wg-hq \
    comment="Allow VPN forwarding"
```

Put these **before your drop rules.**

## Create loopback interfaces

MikroTik doesn't have a special `loopback` interface like some vendors. The usual RouterOS method is a bridge with no ports.

**HQ:**

```routeros
/interface/bridge
add name=lo protocol-mode=none comment="Loopback"

/ip/address
add address=192.168.255.1/32 interface=lo comment="OSPF Loopback"
```

**Site 1:**

```routeros
/interface/bridge
add name=lo protocol-mode=none comment="Loopback"

/ip/address
add address=192.168.255.2/32 interface=lo comment="OSPF Loopback"
```

**Site 2:**

```routeros
/interface/bridge
add name=lo protocol-mode=none comment="Loopback"

/ip/address
add address=192.168.255.3/32 interface=lo comment="OSPF Loopback"
```

**Site 3:**

```routeros
/interface/bridge
add name=lo protocol-mode=none comment="Loopback"

/ip/address
add address=192.168.255.4/32 interface=lo comment="OSPF Loopback"
```

## OSPF on HQ

Since you're using RouterOS 7.x, use the `/routing ospf` configuration.

First:

```routeros
/routing/ospf/instance
add name=ospf-instance router-id=192.168.255.1
```

Then advertise HQ LAN:

```routeros
/routing/ospf/area
add name=backbone instance=ospf-instance area-id=0.0.0.0
```

Add interfaces:

```routeros
/routing/ospf/interface-template
add area=backbone interfaces=wg-sites type=ptp

add area=backbone networks=192.168.1.0/24 passive
add area=backbone networks=192.168.255.1/32 passive
```

The important part is:

```markdown
wg-sites → OSPF neighbor
192.168.1.0/24 → advertise HQ LAN
```

## OSPF Site 1

```routeros
/routing/ospf/instance
add name=ospf-instance router-id=192.168.255.2
```

```routeros
/routing/ospf/area
add name=backbone instance=ospf-instance area-id=0.0.0.0
```

```routeros
/routing/ospf/interface-template
add area=backbone interfaces=wg-hq type=ptp

add area=backbone networks=192.168.10.0/24 passive
add area=backbone networks=192.168.255.2/32 passive
```

## OSPF Site 2

Use a different router ID:

```routeros
/routing/ospf/instance
add name=ospf-instance router-id=192.168.255.3
```

```routeros
/routing/ospf/area
add name=backbone instance=ospf-instance area-id=0.0.0.0
```

```routeros
/routing/ospf/interface-template
add area=backbone interfaces=wg-hq type=ptp

add area=backbone networks=192.168.20.0/24 passive
add area=backbone networks=192.168.255.3/32 passive
```

## OSPF Site 3

```routeros
/routing/ospf/instance
add name=ospf-instance router-id=192.168.255.4
```

```routeros
/routing/ospf/area
add name=backbone instance=ospf-instance area-id=0.0.0.0
```

```routeros
/routing/ospf/interface-template
add area=backbone interfaces=wg-hq type=ptp

add area=backbone networks=192.168.30.0/24 passive
add area=backbone networks=192.168.255.4/32 passive
```

**Note:**

- **ospf-instance router-id** can be `main`, because router dynamic take ip from `lo` interface.
- The passive setting means OSPF advertises the network but does not try to form an OSPF neighbor on the loopback.

**Final addressing:**

| Router | Interface  | IP                 |
| ------ | ---------- | ------------------ |
| HQ     | `lo`       | `192.168.255.1/32` |
| HQ     | `wg-site1` | `10.10.1.1/30`     |
| HQ     | `wg-site2` | `10.10.2.1/30`     |
| HQ     | `wg-site3` | `10.10.3.1/30`     |
| Site 1 | `lo`       | `192.168.255.2/32` |
| Site 1 | `wg-hq`    | `10.10.1.2/30`     |
| Site 2 | `lo`       | `192.168.255.3/32` |
| Site 2 | `wg-hq`    | `10.10.2.2/30`     |
| Site 3 | `lo`       | `192.168.255.4/32` |
| Site 3 | `wg-hq`    | `10.10.3.2/30`     |

**Topology:**

```markdown
                    HQ
                  Loopback
                192.168.255.1/32
                       |
          +------------+------------+
          |            |            |
       WG /30       WG /30       WG /30
          |            |            |
  10.10.1.0/30   10.10.2.0/30   10.10.3.0/30
          |            |            |
          |            |            |
       Site 1        Site 2       Site 3
       .2/30         .2/30        .2/30
          |            |            |
    Loopback       Loopback      Loopback
   .255.2/32      .255.3/32     .255.4/32
```

## Check OSPF neighbors

On HQ:

```routeros
/routing/ospf/neighbor/print
```

You should eventually see:

```routeros
192.168.255.2  Full
192.168.255.3  Full
192.168.255.4  Full
```

That's what you want.

**Topology of OSPF neighbors should be:**

```markdown
HQ 192.168.255.1
 ├── Site 1 192.168.255.2
 │     WG: 10.10.1.0/30
 │
 ├── Site 2 192.168.255.3
 │     WG: 10.10.2.0/30
 │
 └── Site 3 192.168.255.4
       WG: 10.10.3.0/30
```

And routes will be dynamically learned:

```markdown
HQ
  → 192.168.10.0/24 via 10.10.1.2
  → 192.168.20.0/24 via 10.10.2.2
  → 192.168.30.0/24 via 10.10.3.2
```

## Check routes

On HQ:

```routeros
/ip/route/print
```

You should see OSPF routes similar to:

```markdown
DAo  192.168.10.0/24
DAo  192.168.20.0/24
DAo  192.168.30.0/24
```

On Site 1:

```markdown
DAo  192.168.1.0/24
DAo  192.168.20.0/24
DAo  192.168.30.0/24
```

One important correction about `allowed-address`

For a simple **HQ-and-three-site hub-and-spoke WireGuard design**, don't put all remote LANs in WireGuard's `allowed-address` on the HQ peers.

For example, HQ Site 1 peer should normally be:

```routeros
allowed-address=10.10.1.2/32
```

Site 1's peer to HQ can be:

```routeros
allowed-address=10.10.1.1/32
```

Then **OSPF controls the actual LAN routes.**

That keeps the responsibilities clean:

```markdown
WireGuard
   ↓
Creates encrypted tunnel
   ↓
10.10.1.0/30
10.10.2.0/30
10.10.3.0/30
   ↓
OSPF
   ↓
Advertises LAN networks
   ↓
192.168.1.0/24
192.168.10.0/24
192.168.20.0/24
192.168.30.0/24
```

**Recommended final design:**

```markdown
                   HQ
             192.168.1.0/24
                    |
             WG 10.10.1.1
             WG 10.10.2.1
             WG 10.10.3.1
              /     |     \
             /      |      \
           WG       WG       WG
         1.2        2.2      3.2
          |          |        |
       Site 1      Site 2   Site 3
      .10.0/24    .20.0/24 .30.0/24

              OSPF AREA 0
```

For your MikroTik WireGuard + OSPF, 1 HQ + 3 sites, I recommend /30, not /31.

|Overview|`/30`|`/31`|
|-|-|-|
|Usable IPs|2|2|
|Network/broadcast|Yes|No|
|MikroTik support|Excellent|Supported, but more specialized|
|Troubleshooting|Easier|Slightly less intuitive|
|OSPF point-to-point|Excellent|Excellent|
|Conventional P2P design|✅||
|IP efficiency||✅|
|My recommendation|✅**/30**||

**Use /30 like this:**

```markdown
HQ ↔ Site 1
10.10.1.0/30
```

```markdown
HQ    10.10.1.1
Site1 10.10.1.2
HQ ↔ Site 2
10.10.2.0/30
```

```markdown
HQ    10.10.2.1
Site2 10.10.2.2
HQ ↔ Site 3
10.10.3.0/30
```

```markdown
HQ    10.10.3.1
Site3 10.10.3.2
```

Why I prefer /30 for your setup

You only have **3 site-to-site tunnels**, so saving two IP addresses per tunnel with `/31` doesn't provide much practical benefit.

With /30, your configuration is very obvious:

```markdown
10.10.1.1 ←→ 10.10.1.2
10.10.2.1 ←→ 10.10.2.2
10.10.3.1 ←→ 10.10.3.2
```

Then OSPF runs over those WireGuard point-to-point links.

- This is a good setup for your environment because if **Site 2's LAN changes**, you don't have to manually create a static route on HQ; OSPF can advertise the new route.

- If you give me your actual HQ LAN + Site 1/2/3 LAN subnets and each router's current WireGuard interface name, I can give you the exact copy/paste RouterOS 7.23.1 commands for all 4 MikroTik routers, including firewall, NAT bypass, OSPF, and failover.

- use `/30 + OSPF + WireGuard`. It is simple, easy to troubleshoot, and leaves room for future changes.
