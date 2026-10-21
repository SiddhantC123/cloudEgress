# CloudEgress – Home-to-Cloud VPN Gateway (Raspberry Pi ⇄ AWS EC2)

I built a site-to-cloud VPN that sends my home network's internet traffic through an encrypted **WireGuard** tunnel to an **AWS EC2** instance, which acts as the internet exit point. A **Raspberry Pi** on my home LAN is the local gateway.

Devices at home reach the internet through AWS, so websites and services see the EC2 **Elastic IP** instead of my home ISP's IP address.

**Tech:** WireGuard · AWS EC2 · Raspberry Pi · Linux routing · policy routing (`ip rule`, fwmark) · iptables / netfilter · NAT (MASQUERADE) · conntrack

---

## Architecture

```text
 ┌──────────────────────────── Home Network ────────────────────────────┐
 │                                                                      │
 │  [Laptop / Phone / Other LAN devices]                                │
 │          │  default gateway = Raspberry Pi                           │
 │          ▼                                                           │
 │  [Raspberry Pi – LAN gateway]                                        │
 │     eth0 : 192.168.1.50   (static LAN IP)                            │
 │     wg0  : 10.0.0.2/24    (tunnel IP)                                │
 │          │                                                           │
 │  [Home router] ── NAT ── Home ISP public IP                          │
 └──────────┼───────────────────────────────────────────────────────────┘
            │
            │  WireGuard: encrypted UDP 51820 over the public internet
            ▼
 ┌──────────────────────────── AWS (VPC) ───────────────────────────────┐
 │  [EC2 – cloud egress gateway]                                        │
 │     wg0  : 10.0.0.1/24        (tunnel IP)                            │
 │     ens5 : private VPC IP  ←→  Elastic IP (public)                   │
 │          │  iptables MASQUERADE (SNAT to EC2 IP)                     │
 └──────────┼───────────────────────────────────────────────────────────┘
            ▼
     [Public Internet]  ← sees the Elastic IP, not my home IP
```

| Component | Role | Tunnel IP |
|---|---|---|
| AWS EC2 (Ubuntu) | WireGuard "server", NAT, internet exit | `10.0.0.1` |
| Raspberry Pi (Raspberry Pi OS) | WireGuard "client", LAN gateway, NAT into tunnel | `10.0.0.2` |
| Home LAN devices | Use the Pi as their gateway | — |

Keys and public IPs are redacted in this README. The examples in [How It Works](#how-it-works) use documentation address ranges:

| Thing | Address |
|---|---|
| Laptop | `192.168.1.20` |
| Pi eth0 / wg0 | `192.168.1.50` / `10.0.0.2` |
| Pi's WireGuard UDP port | `41000` (random, since the Pi has no `ListenPort`) |
| Home router public IP | `192.0.2.44` |
| EC2 private IP (ens5) | `172.31.5.10` |
| EC2 Elastic IP | `198.51.100.7` |
| EC2 wg0 | `10.0.0.1` |
| Website | `203.0.113.10:443` |

---

## Design Decisions

- **WireGuard over IPsec/OpenVPN:** small codebase, modern cryptography (Curve25519, ChaCha20-Poly1305), runs in the Linux kernel, and needs only one UDP port.
- **The Pi initiates the connection:** my home network sits behind the ISP's NAT and its public IP can change. The Pi connects *out* to a fixed Elastic IP, so I didn't need any port forwarding on the home router. `PersistentKeepalive` keeps the router's NAT mapping open so EC2 can always reach the Pi.
- **NAT on both nodes:** the Pi masquerades LAN traffic to its tunnel IP, so EC2 only has to know about `10.0.0.2`. This kept EC2's config to a single peer address. A routed (non-NAT) design would instead add the home subnet to EC2's `AllowedIPs`.
- **Elastic IP:** gives EC2 a fixed public IP, so the Pi's `Endpoint` doesn't break when the instance restarts.
- **EC2 source/destination check left enabled:** EC2 masquerades all tunnel traffic to its own IP, so every packet it sends into the VPC has its own address as the source. The check would only need disabling in a routed design.
- **Full tunnel (`0.0.0.0/0`) on the Pi:** every IPv4 destination goes through AWS. `wg-quick`'s policy routing handles this without breaking access to the local LAN (explained in [Routing](#2-routing-how-traffic-gets-into-the-tunnel)).

---

## Implementation

### 1. AWS side

- **Instance:** Ubuntu Server LTS on a `t3.micro` (free-tier eligible; WireGuard is very lightweight).
- **Elastic IP:** allocated and associated with the instance.
- **Security group:**

| Type | Protocol | Port | Source | Purpose |
|---|---|---|---|---|
| SSH | TCP | 22 | my home IP `/32` | Admin access |
| Custom UDP | UDP | 51820 | `0.0.0.0/0` | WireGuard |

I left UDP 51820 open to all sources because my home IP can change. WireGuard silently ignores any packet not signed by a known peer key, so the open port doesn't reveal the service to scanners.

- **Public interface:** on current Ubuntu AMIs the internet-facing interface is **`ens5`**, not `eth0`, so the NAT rule on EC2 uses `ens5`:

```bash
ip route show default
# default via 172.31.0.1 dev ens5 proto dhcp ...
```

### 2. Raspberry Pi preparation

- Updated the system and installed `iptables`, since recent Raspberry Pi OS releases use nftables and don't always ship the `iptables` command:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install iptables -y
```

- Gave the Pi a **static LAN IP** (`192.168.1.50`), because LAN devices use it as their gateway and it must not change. Raspberry Pi OS Bookworm uses NetworkManager:

```bash
sudo nmcli connection modify "Wired connection 1" \
  ipv4.method manual \
  ipv4.addresses 192.168.1.50/24 \
  ipv4.gateway 192.168.1.1 \
  ipv4.dns "1.1.1.1"
sudo nmcli connection up "Wired connection 1"
```

### 3. WireGuard keys

On both nodes I installed WireGuard and generated a key pair readable only by root:

```bash
sudo apt install wireguard -y
sudo -i
cd /etc/wireguard
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
exit
```

Each node's `[Peer]` section holds the **other** node's public key. Private keys never leave the machine they were generated on and are excluded from this repository.

### 4. IP forwarding

Both nodes route packets for other hosts, so I enabled IPv4 forwarding persistently on each:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-wireguard.conf
sudo sysctl --system
```

I used a file in `/etc/sysctl.d/` because newer Debian-based releases no longer read `/etc/sysctl.conf` by default.

### 5. EC2 configuration

`/etc/wireguard/wg0.conf` on EC2:

```ini
[Interface]
Address    = 10.0.0.1/24
ListenPort = 51820
PrivateKey = <EC2_PRIVATE_KEY>

# On tunnel start:
#   1-2. Allow forwarding into and out of the tunnel
#   3.   Masquerade (SNAT) tunnel traffic leaving to the internet as the EC2's own IP
PostUp   = iptables -A FORWARD -i wg0 -j ACCEPT
PostUp   = iptables -A FORWARD -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostUp   = iptables -t nat -A POSTROUTING -s 10.0.0.0/24 -o ens5 -j MASQUERADE

# On tunnel stop: remove the same rules
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT
PostDown = iptables -D FORWARD -o wg0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s 10.0.0.0/24 -o ens5 -j MASQUERADE

[Peer]
# Raspberry Pi
PublicKey  = <PI_PUBLIC_KEY>
AllowedIPs = 10.0.0.2/32
```

What each part does:

- `Address` – EC2's tunnel IP. It also creates the route `10.0.0.0/24 dev wg0`, which sends replies for the Pi into the tunnel.
- No `Endpoint` – EC2 learns the Pi's current public address from the Pi's first authenticated packet.
- `FORWARD -i wg0 -j ACCEPT` – lets traffic coming out of the tunnel be forwarded to the internet.
- `FORWARD -o wg0 ... --ctstate RELATED,ESTABLISHED` – lets only *replies* back into the tunnel.
- `MASQUERADE -s 10.0.0.0/24 -o ens5` – rewrites the source of tunnel traffic to EC2's IP so internet replies return to EC2. Limited to the tunnel subnet so EC2's own traffic isn't touched.
- `AllowedIPs = 10.0.0.2/32` – traffic **to** `10.0.0.2` goes to the Pi, and traffic **from** the Pi is only accepted if its inner source is `10.0.0.2`.

### 6. Raspberry Pi configuration

`/etc/wireguard/wg0.conf` on the Pi:

```ini
[Interface]
Address    = 10.0.0.2/24
PrivateKey = <PI_PRIVATE_KEY>

# Allow LAN devices' traffic to be forwarded into the tunnel, and replies back out
PostUp   = iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT
PostUp   = iptables -A FORWARD -i wg0 -o eth0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
# Rewrite LAN source IPs to the Pi's tunnel IP (10.0.0.2) before entering the tunnel
PostUp   = iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
# Clamp TCP MSS to the tunnel MTU so large TCP packets from LAN devices aren't dropped
PostUp   = iptables -t mangle -A FORWARD -o wg0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

PostDown = iptables -D FORWARD -i eth0 -o wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -o eth0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -o wg0 -j MASQUERADE
PostDown = iptables -t mangle -D FORWARD -o wg0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu

[Peer]
# AWS EC2
PublicKey           = <EC2_PUBLIC_KEY>
Endpoint            = <AWS_ELASTIC_IP>:51820
AllowedIPs          = 0.0.0.0/0
PersistentKeepalive = 25
```

What each part does:

- `AllowedIPs = 0.0.0.0/0` – every IPv4 destination is reached through EC2. Because this covers the whole internet, `wg-quick` also sets up policy routing so the tunnel's own encrypted packets still leave through the LAN instead of looping into the tunnel.
- LAN access keeps working: `wg-quick` adds a `suppress_prefixlength 0` rule, so specific routes such as `192.168.1.0/24` still win. SSH to the Pi from the LAN is unaffected.
- `PersistentKeepalive = 25` – keeps the home router's NAT mapping alive so EC2 can always reach the Pi.
- `MASQUERADE -o wg0` – lets every LAN device use the tunnel while EC2 only ever sees `10.0.0.2`.
- `TCPMSS --clamp-mss-to-pmtu` – WireGuard's MTU (1420) is smaller than Ethernet's (1500). Without clamping, large responses don't fit through the tunnel and pages can hang halfway.

Both config files are `chmod 600` and root-owned.

### 7. Bringing the tunnel up

I started EC2 first, then the Pi, and enabled both at boot:

```bash
sudo systemctl enable --now wg-quick@wg0
```

I configured the Pi from inside the home network: bringing up a full tunnel changes how the Pi routes replies, which can drop an SSH session coming from outside.

### 8. Routing LAN devices through the Pi

With the tunnel working for the Pi itself, I pointed LAN devices at the Pi by setting their **gateway** to `192.168.1.50`. Visiting `https://ifconfig.me` from those devices then showed the Elastic IP instead of my home IP.

If the Pi or the tunnel goes down, devices using it as their gateway lose internet access, so I kept the router as a fallback gateway I can switch back to.

---

## Verification

| Check | Command | Result |
|---|---|---|
| Handshake established | `sudo wg show` | `latest handshake: … seconds ago`, transfer counters increasing |
| Tunnel reachable | `ping -c 3 10.0.0.1` (from the Pi) | replies from EC2's tunnel IP |
| Traffic exits through AWS | `traceroute -n 8.8.8.8` | first hop is `10.0.0.1`, followed by AWS hops |
| Public IP changed | `curl ifconfig.me` | the Elastic IP, not my home IP |
| NAT and forwarding active | `sudo iptables -t nat -L POSTROUTING -n -v` and `sudo iptables -L FORWARD -n -v` on EC2 | counters increasing on `MASQUERADE` and `ACCEPT` |
| LAN devices tunnelled | `https://ifconfig.me` in a browser on a LAN device | the Elastic IP |

`sudo wg show` on the Pi (keys and IPs redacted):


interface: wg0
  public key: ...
  listening port: ...
  fwmark: 0xca6c

peer: <EC2_PUBLIC_KEY>
  endpoint: <AWS_ELASTIC_IP>:51820
  allowed ips: 0.0.0.0/0
  latest handshake: 12 seconds ago
  transfer: 4.21 KiB received, 6.87 KiB sent
  persistent keepalive: every 25 seconds


---

## How It Works

This is how a packet actually moves through the system, using the example addresses from [Architecture](#architecture).

There are **two packets** for every request:

- **The letter** – the real packet, e.g. laptop → website.
- **The envelope** – WireGuard's encrypted UDP packet between the Pi and EC2, carrying the letter sealed inside. This is all the internet sees.

### 1. What wg-quick builds on the Pi

When `wg-quick up wg0` runs on the Pi, it executes these commands:

[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.0.0.2/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
[#] wg set wg0 fwmark 51820
[#] ip -4 route add 0.0.0.0/0 dev wg0 table 51820
[#] ip -4 rule add not fwmark 51820 table 51820
[#] ip -4 rule add table main suppress_prefixlength 0
[#] sysctl -q net.ipv4.conf.all.src_valid_mark=1
[#] nft -f /dev/fd/63          (or iptables-restore if nft is not installed)
[#] iptables ... (the PostUp lines)


| Command | What it creates | Effect on packets |
|---|---|---|
| `ip link add wg0 type wireguard` | Virtual interface wg0 | "Sending out wg0" means handing the packet to WireGuard to be encrypted |
| `wg setconf wg0 ...` | Keys, `Endpoint`, `AllowedIPs = 0.0.0.0/0` | WireGuard's peer table: every IPv4 address → EC2 |
| `ip -4 address add 10.0.0.2/24 dev wg0` | Address on wg0 + route `10.0.0.0/24 dev wg0` | MASQUERADE uses `10.0.0.2` as the new source |
| `ip link set mtu 1420 up dev wg0` | wg0 up, max packet 1420 | Leaves room for the envelope's headers |
| `wg set wg0 fwmark 51820` | WireGuard marks every envelope with `0xca6c` | Routing can tell envelopes apart from normal traffic |
| `ip -4 route add 0.0.0.0/0 dev wg0 table 51820` | Separate routing table: everything → wg0 | Unused until a rule points to it |
| `ip -4 rule add not fwmark 51820 table 51820` | Rule 32765: packets **without** the mark use table 51820 | Normal traffic goes into the tunnel |
| `ip -4 rule add table main suppress_prefixlength 0` | Rule 32764: use specific routes in main, ignore main's default route | LAN and tunnel subnets keep working |
| `sysctl ... src_valid_mark=1` | Reverse-path check considers marks | See [CONNMARK](#6-connmark-keeping-the-tunnels-own-replies-valid) |
| `nft -f ...` | CONNMARK save/restore rules + an anti-spoofing rule | See [CONNMARK](#6-connmark-keeping-the-tunnels-own-replies-valid) |

On EC2, `wg-quick` only creates the interface, address, MTU and the PostUp rules. It adds no fwmark or policy routing, because EC2's peer has `AllowedIPs = 10.0.0.2/32`, not `0.0.0.0/0`.

### 2. Routing: how traffic gets into the tunnel

Linux routing has two levels:

1. **`ip rule`** – a list checked top to bottom that says **which routing table** to look in.
2. **The routing table** – the most specific matching route picks the interface.

The Pi's routing state with the tunnel up:

```text
$ ip rule
0:      from all lookup local
32764:  from all lookup main suppress_prefixlength 0
32765:  not from all fwmark 0xca6c lookup 51820
32766:  from all lookup main
32767:  from all lookup default

$ ip route show table main
default via 192.168.1.1 dev eth0
10.0.0.0/24 dev wg0
192.168.1.0/24 dev eth0

$ ip route show table 51820
default dev wg0
```

**A packet from the laptop to a website** (`192.168.1.20 → 203.0.113.10`, no mark):

| Rule | Check | Result |
|---|---|---|
| 32764 main, suppress default | Only main's default route matches, and it's ignored | next rule |
| 32765 not fwmark 0xca6c | Packet has no mark → applies | table 51820 → **out wg0** |

**WireGuard's own envelope** (`192.168.1.50 → 198.51.100.7`, mark `0xca6c`):

| Rule | Check | Result |
|---|---|---|
| 32764 main, suppress default | Only default matches, ignored | next rule |
| 32765 not fwmark 0xca6c | Packet **has** the mark → skipped | next rule |
| 32766 main | `default via 192.168.1.1 dev eth0` | **out eth0** to the router |

Without the mark, the envelope would hit rule 32765, go back into wg0, be encrypted again, and loop forever.

**A reply to the laptop** (`→ 192.168.1.20`):

| Rule | Check | Result |
|---|---|---|
| 32764 main, suppress default | `192.168.1.0/24 dev eth0` matches and is specific | **out eth0** |

Rule 32765 is never reached, so LAN traffic is never pulled into the tunnel.

### 3. AllowedIPs: how a peer is chosen

Routing chooses the **interface**. Inside wg0, `AllowedIPs` chooses the **peer**. It reads as *"these addresses live behind this peer"* and is used in both directions:

| Direction | Field checked | Pi (`0.0.0.0/0` → EC2) | EC2 (`10.0.0.2/32` → Pi) |
|---|---|---|---|
| **Sending** | the letter's **destination** | `203.0.113.10` ∈ `0.0.0.0/0` → encrypt for EC2 | `10.0.0.2` ∈ `10.0.0.2/32` → encrypt for the Pi |
| **Receiving** | the letter's **source**, after decrypting | `203.0.113.10` ∈ `0.0.0.0/0`, owner EC2 ✓ accept | `10.0.0.2` ∈ `10.0.0.2/32`, owner Pi ✓ accept |

Consequences:

- If no peer's AllowedIPs contains a destination, WireGuard drops the packet (`Required key not available`).
- The Pi's `0.0.0.0/0` is what lets **replies from any website** be accepted from EC2.
- EC2 only accepts `10.0.0.2` from the Pi, which is why the Pi must masquerade LAN addresses to `10.0.0.2` **before** encryption.
- With several peers on one interface, the **most specific** matching range wins, and each range can belong to only one peer.

### 4. Conntrack and NAT: how replies find their way back

#### The problem

The kernel handles each packet on its own. After the Pi rewrites the laptop's address to `10.0.0.2`, the website's reply arrives as:

```text
203.0.113.10:443 → 10.0.0.2:50000
```

Nothing in this packet says "laptop". The Pi has to **remember** what it changed. That memory is **conntrack**.

#### What switches conntrack on

Conntrack is part of the kernel and starts tracking as soon as a rule needs it. On the Pi, either of these is enough:

```bash
iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE                                        # NAT has to remember rewrites
iptables -A FORWARD -i wg0 -o eth0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT   # needs connection state
```

(plus wg-quick's CONNMARK rules). On EC2, its MASQUERADE and `--ctstate` rules do the same. Once on, conntrack tracks **every** packet on the machine (TCP, UDP and ICMP), not only the ones the rules mention. It has to, because it sees each packet on arrival, before routing has decided whether NAT will apply.

#### What an entry holds

Each conversation gets one entry with two lines:

```text
original: how the first packet looked
reply   : how a reply is expected to look
```

| A packet matches… | Conntrack labels it |
|---|---|
| no entry | **NEW** – a new entry is created |
| an entry, and no reply has been seen yet | **NEW** |
| an entry that has seen packets in both directions | **ESTABLISHED** |
| an error message about an existing entry (e.g. ICMP "packet too big") | **RELATED** |

#### One conversation, step by step (on the Pi)

**1. The laptop's first packet arrives on eth0**

```text
192.168.1.20:50000 → 203.0.113.10:443
```

No entry matches, so conntrack creates one and labels the packet **NEW**:

```text
original: 192.168.1.20:50000 → 203.0.113.10:443
reply   : 203.0.113.10:443   → 192.168.1.20:50000
```

**2. MASQUERADE runs (only because the label is NEW)**

The nat table is only checked for NEW connections. At POSTROUTING:

```bash
iptables -t nat -A POSTROUTING -o wg0 -j MASQUERADE
```

changes the source and **rewrites the entry's reply line**:

```text
packet:   10.0.0.2:50000 → 203.0.113.10:443

original: 192.168.1.20:50000 → 203.0.113.10:443
reply   : 203.0.113.10:443   → 10.0.0.2:50000      ← written by MASQUERADE
```

Before choosing the port, MASQUERADE checks that no other entry already expects replies to `10.0.0.2:50000`. If another device were using it, a different port would be chosen so the two conversations stay distinguishable.

**3. The reply comes back through EC2 and the tunnel**

```text
203.0.113.10:443 → 10.0.0.2:50000
```

It matches the entry's **reply line**, so conntrack:

- labels it **ESTABLISHED** (a reply has now been seen), and
- reverses the rewrite: destination → `192.168.1.20:50000`.

This happens **before routing**, so routing sees the laptop's address rather than the Pi's own `10.0.0.2`.

**4. Every later packet**

The label is ESTABLISHED, so the nat table is **not** checked again. Conntrack applies the recorded rewrite to every outgoing packet and reverses it on every reply. MASQUERADE runs **once per conversation**.

When the conversation ends, the entry is removed after a short timeout.

#### Where conntrack is used across the system

| Machine | Entry | Purpose |
|---|---|---|
| Pi | laptop ⇄ website | Undo the Pi's MASQUERADE on replies |
| Pi | Pi ⇄ EC2 envelope (UDP) | Store WireGuard's mark (see CONNMARK) |
| Home router | Pi ⇄ EC2 envelope | The router's own NAT: its public port means "the Pi". `PersistentKeepalive = 25` stops this entry from expiring |
| EC2 | Pi ⇄ EC2 envelope | Tracking only |
| EC2 | `10.0.0.2` ⇄ website | Undo EC2's MASQUERADE on replies |

### 5. FORWARD rules: what is allowed through

MASQUERADE and conntrack decide **addresses**. FORWARD rules decide **permission**: whether a packet may pass through the machine at all.

On the Pi:

```bash
iptables -A FORWARD -i eth0 -o wg0 -j ACCEPT
iptables -A FORWARD -i wg0 -o eth0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

| Part | Meaning |
|---|---|
| `-i eth0 -o wg0` | from the home LAN into the tunnel |
| `-i wg0 -o eth0` | from the tunnel into the home LAN |
| `--ctstate RELATED,ESTABLISHED` | only packets belonging to (or about) a conversation already started from the LAN |

So: **out – anything; in – only replies.**

| Packet from the tunnel into the LAN | Conntrack label | Result (with FORWARD policy DROP) |
|---|---|---|
| Website's reply to the laptop | ESTABLISHED | allowed |
| ICMP "packet too big" about the laptop's connection | RELATED | allowed (needed for large transfers) |
| A new connection attempt toward a LAN device (e.g. from a compromised EC2) | NEW | **blocked** |

EC2 follows the same idea: `-i wg0 -j ACCEPT` lets tunnel traffic out, and `-o wg0 ... --ctstate RELATED,ESTABLISHED` lets only replies back in.

These rules only block anything when the FORWARD chain's default policy is DROP; on Raspberry Pi OS and Ubuntu the default is ACCEPT (see [Security](#security)).

### 6. CONNMARK: keeping the tunnel's own replies valid

`wg-quick` adds two rules on the Pi that aren't in `wg0.conf`:

```bash
iptables -t mangle -I POSTROUTING -m mark --mark 0xca6c -p udp -j CONNMARK --save-mark
iptables -t mangle -I PREROUTING -p udp -j CONNMARK --restore-mark
```

Linux has a built-in anti-spoofing check (the reverse-path filter). For every arriving packet it asks: *"if I sent something back to this sender, would it leave through the interface this packet came in on?"*

| Arriving packet | Came in on | Linux would reply via | Result in strict mode |
|---|---|---|---|
| Fake packet claiming to be from `10.0.0.1` | eth0 | wg0 | dropped (correctly) |
| EC2's envelope, **no mark** | eth0 | wg0 (full tunnel: everything goes via wg0) | dropped (wrongly) |
| EC2's envelope, **mark restored** | eth0 | eth0 (marked packets skip rule 32765) | allowed |

The mark only exists inside the Pi, so EC2's reply envelope arrives without it. `--save-mark` stores the outgoing envelope's mark in its conntrack entry, and `--restore-mark` copies it back onto the reply.

On Raspberry Pi OS the reverse-path filter is usually in *loose* mode, where the tunnel would work even without these rules; `wg-quick` adds them so full-tunnel setups also work in *strict* mode.

### 7. MTU and MSS: why packets are made smaller

- **MTU** – the largest **whole packet** an interface can send (headers + data). Ethernet and the internet: **1500**.
- **MSS** – the largest amount of **data only** in one TCP packet. MSS = MTU − 20 (IP header) − 20 (TCP header).

A maximum-size packet leaving the Pi through the tunnel:

| Layer | Bytes | Running total |
|---|---|---|
| Data | 1380 | 1380 (MSS) |
| + TCP header | 20 | 1400 |
| + inner IP header | 20 | **1420** (wg0 MTU – the letter) |
| + WireGuard header | 16 | 1436 |
| + authentication tag | 16 | 1452 |
| + UDP header | 8 | 1460 |
| + outer IP header | 20 | **1480** (the envelope, fits in 1500) |

`wg-quick` sets wg0's MTU to 1500 − **80** = 1420. The envelope needs only 60 bytes over IPv4, but 80 also covers an IPv6 outer header.

**Why the MSS clamp is needed:** LAN devices have a 1500 MTU, so they advertise MSS 1460 when opening a connection. Without the clamp, websites would send 1500-byte packets, which become 1560-byte envelopes – too large for the internet's 1500 – and get fragmented or dropped between AWS and home. The symptom is that small requests work but pages load halfway and hang.

```bash
iptables -t mangle -A FORWARD -o wg0 -p tcp --tcp-flags SYN,RST SYN -j TCPMSS --clamp-mss-to-pmtu
```

This rewrites the MSS in each connection's opening packet as it enters wg0: 1420 − 40 = **1380**. Websites then send at most 1420-byte packets, and every envelope stays at 1480 bytes.

For uploads, if a LAN device sends a 1500-byte packet, the Pi replies with ICMP "packet too big, use 1420" and the device sends smaller packets (path MTU discovery).

### 8. Full packet walkthrough

One request from the laptop to `203.0.113.10:443` and its reply, with the tunnel already up.

**Outbound**

| # | Where | What acts | Result |
|---|---|---|---|
| 1 | Laptop | Gateway = Pi | Packet sent to the Pi: `192.168.1.20:50000 → 203.0.113.10:443` |
| 2 | Pi | Conntrack | New entry, label NEW |
| 3 | Pi | Rule 32765 → table 51820 | Out wg0 |
| 4 | Pi | `TCPMSS --clamp-mss-to-pmtu`, `-i eth0 -o wg0 -j ACCEPT` | MSS capped at 1380 (first packet only); allowed |
| 5 | Pi | `-t nat -o wg0 -j MASQUERADE` | Source → `10.0.0.2`; recorded in the entry |
| 6 | Pi | WireGuard, `AllowedIPs = 0.0.0.0/0` | Peer EC2; letter sealed in envelope `192.168.1.50:41000 → 198.51.100.7:51820`, mark `0xca6c` |
| 7 | Pi | Mark skips rule 32765 → main | Envelope out eth0 to the router |
| 8 | Pi | `CONNMARK --save-mark` | Mark stored in the envelope's conntrack entry |
| 9 | Home router | Router's NAT + conntrack | Envelope source → `192.0.2.44:61234` |
| 10 | AWS edge | Security group, Elastic IP mapping | Envelope destination → `172.31.5.10` |
| 11 | EC2 | WireGuard, `AllowedIPs = 10.0.0.2/32` | Decrypted; inner source `10.0.0.2` accepted; Pi's address learned; letter appears on wg0 |
| 12 | EC2 | Conntrack | New entry, label NEW |
| 13 | EC2 | Main table default route | Out ens5 |
| 14 | EC2 | `-i wg0 -j ACCEPT` | Allowed |
| 15 | EC2 | `-t nat -s 10.0.0.0/24 -o ens5 -j MASQUERADE` | Source → `172.31.5.10`; recorded in the entry |
| 16 | AWS edge | Elastic IP mapping | Source → `198.51.100.7`; website receives it |

**Return**

| # | Where | What acts | Result |
|---|---|---|---|
| 17 | AWS edge | Elastic IP mapping | `203.0.113.10:443 → 172.31.5.10:50000` |
| 18 | EC2 | Conntrack (reply line matches) | Label ESTABLISHED; destination → `10.0.0.2:50000` |
| 19 | EC2 | Route `10.0.0.0/24 dev wg0` | Out wg0 |
| 20 | EC2 | `-o wg0 --ctstate RELATED,ESTABLISHED -j ACCEPT` | Allowed |
| 21 | EC2 | WireGuard, `AllowedIPs = 10.0.0.2/32` | Peer Pi; sealed; sent to the Pi's learned address `192.0.2.44:61234` |
| 22 | AWS edge | Elastic IP mapping | Envelope source → `198.51.100.7` |
| 23 | Home router | Conntrack (reply line matches) | Envelope destination → `192.168.1.50:41000` |
| 24 | Pi | `CONNMARK --restore-mark` | Mark restored; delivered to WireGuard |
| 25 | Pi | WireGuard, `AllowedIPs = 0.0.0.0/0` | Decrypted; inner source `203.0.113.10` accepted; letter appears on wg0 |
| 26 | Pi | Conntrack (reply line matches) | Label ESTABLISHED; destination → `192.168.1.20:50000` |
| 27 | Pi | Rule 32764, route `192.168.1.0/24 dev eth0` | Out eth0 |
| 28 | Pi | `-i wg0 -o eth0 --ctstate RELATED,ESTABLISHED -j ACCEPT` | Allowed |
| 29 | Laptop | — | Reply received from `203.0.113.10:443` |

Every later packet follows the same path, except the MSS clamp no longer applies and neither MASQUERADE rule is checked: conntrack applies the recorded rewrites.

---

## Debugging and Observability

Commands I use to see each mechanism in action.

**On the Pi:**

```bash
# Routing decisions
ip rule
ip route show table 51820
ip route get 203.0.113.10 from 192.168.1.20 iif eth0   # → dev wg0 table 51820
ip route get 198.51.100.7 mark 0xca6c                   # → via 192.168.1.1 dev eth0
ip route get 192.168.1.20                               # → dev eth0

# WireGuard peer table and state
sudo wg show wg0 allowed-ips
sudo wg show wg0 latest-handshakes

# The same request seen three ways (run together, then load a page on a LAN device)
sudo tcpdump -ni eth0 host 192.168.1.20 and port 443   # the letter from the laptop
sudo tcpdump -ni wg0 port 443                          # the letter after NAT, inside the tunnel
sudo tcpdump -ni eth0 udp port 51820                   # only encrypted envelopes

# Conntrack entries appearing live
sudo conntrack -E -p tcp --dport 443
sudo conntrack -L -p udp --dport 51820                 # the envelope entry, shows mark=51820

# Rule counters
sudo iptables -t nat -L POSTROUTING -n -v              # rises once per connection, not per packet
sudo iptables -L FORWARD -n -v
sudo iptables -t mangle -L -n -v
```

**On EC2:**

```bash
sudo wg show wg0 endpoints                    # my home public IP:port, learned automatically
ip route get 10.0.0.2                         # → dev wg0
sudo tcpdump -ni ens5 udp port 51820          # encrypted envelopes
sudo tcpdump -ni wg0 port 443                 # 10.0.0.2 ↔ website
sudo tcpdump -ni ens5 tcp port 443            # 172.31.5.10 ↔ website
sudo conntrack -E -p tcp --dport 443
```

**Failure modes and what causes them:**

| Symptom | Cause |
|---|---|
| No `latest handshake` in `wg show` | UDP 51820 blocked, wrong `Endpoint`, or public keys swapped |
| Handshake OK, but `ping 10.0.0.1` fails | `AllowedIPs` wrong on EC2 |
| Handshake OK, but LAN devices get nothing | Pi not masquerading, so EC2 drops inner sources other than `10.0.0.2` |
| Ping works, but no internet | EC2's NAT rule on the wrong interface (`eth0` instead of `ens5`) |
| `Required key not available` | Destination not covered by any peer's `AllowedIPs` |
| Some sites hang or load partially | Missing MSS clamp |
| Tunnel dies after idle periods | Home router's NAT entry expired – needs `PersistentKeepalive` |
| `resolvconf: command not found` | `DNS =` line in `wg0.conf` without `openresolv` installed |

---

## Security

- Private keys and both `wg0.conf` files are `chmod 600` and root-owned, and are excluded from this repository via `.gitignore`.
- SSH on EC2 is restricted to my home IP with key-based authentication.
- WireGuard only responds to packets authenticated with a known peer key, so the open UDP port is invisible to scanners.
- EC2 is the exit point for my home traffic, so I keep it patched.
- **Default-deny forwarding (next hardening step):** the FORWARD rules only restrict traffic when the chain's default policy is DROP. Adding this to the Pi's `wg0.conf` makes it allow only "LAN → tunnel" and "replies → LAN":

  ```ini
  PostUp   = iptables -P FORWARD DROP
  PostDown = iptables -P FORWARD ACCEPT
  ```

  With this in place, even a compromised EC2 instance couldn't open new connections into my home LAN: its packets would be labelled NEW and match no rule.
- IPv6 is not tunnelled. Devices with IPv6 may reach some sites directly; forcing everything through the tunnel would mean disabling IPv6 on the LAN or extending the configs with IPv6 addresses and `::/0`.

`.gitignore`:

```gitignore
privatekey
publickey
*.key
wg0.conf
```

---

## Future Work: Multiple Exit Servers

Every forwarded packet involves two decisions:

| Decision | Made by |
|---|---|
| Which interface (eth0, wg0, wg1…) | Routing: `ip rule` + routing tables |
| Which peer on that interface | WireGuard: `AllowedIPs` |

**Split by destination – two peers on one wg0.** Each range can belong to only one peer, and the most specific match wins:

```ini
[Peer]
# EC2-A: default exit
PublicKey           = <EC2_A_PUBLIC_KEY>
Endpoint            = <EC2_A_ELASTIC_IP>:51820
AllowedIPs          = 0.0.0.0/0
PersistentKeepalive = 25

[Peer]
# EC2-B: only one specific network
PublicKey           = <EC2_B_PUBLIC_KEY>
Endpoint            = <EC2_B_ELASTIC_IP>:51820
AllowedIPs          = 10.0.0.3/32, 52.95.0.0/16
PersistentKeepalive = 25
```

| Destination | Goes to |
|---|---|
| `203.0.113.10` | EC2-A (only `0.0.0.0/0` matches) |
| `52.95.10.5` | EC2-B (`/16` beats `/0`) |
| `10.0.0.3` | EC2-B (`/32` beats `/0`) |

Giving both peers `0.0.0.0/0` doesn't load-balance: the last peer in the file takes the range.

**Split by device – two full exits.** Two interfaces (wg0 → EC2-A, wg1 → EC2-B), each with `AllowedIPs = 0.0.0.0/0` and `Table = off`, with routing choosing the interface by source address:

```bash
ip route add default dev wg0 table 100
ip route add default dev wg1 table 200
ip rule add lookup main suppress_prefixlength 0 priority 100   # LAN and tunnel subnets always win
ip rule add from 192.168.1.20 lookup 100 priority 200           # laptop → EC2-A
ip rule add from 192.168.1.30 lookup 200 priority 201           # phone  → EC2-B
```

The Pi's own envelopes (source `192.168.1.50`) match none of the per-device rules and leave via the main table, so no fwmark is needed. Automatic failover between exits would need a health-check script that rewrites these rules.

---

## What I Learned

- **Policy routing:** how `ip rule`, separate routing tables, fwmark and `suppress_prefixlength 0` let `wg-quick` send all traffic through a tunnel without routing the tunnel's own packets into itself, while keeping the LAN reachable
- **WireGuard cryptokey routing:** how `AllowedIPs` selects the peer by destination when sending and validates the source when receiving
- **Netfilter packet flow:** the order of PREROUTING → routing → FORWARD → POSTROUTING, and why NAT has to happen before WireGuard encrypts the packet
- **Conntrack:** what switches it on, how it tracks every connection, how MASQUERADE runs once per connection and records its rewrite, how replies are un-NATed before routing, and how NEW / ESTABLISHED / RELATED states drive stateful firewall rules
- **Stateful filtering:** using `--ctstate RELATED,ESTABLISHED` so the tunnel side can only send replies into the home network
- **CONNMARK and reverse-path filtering:** why a full-tunnel setup needs the tunnel's mark restored on its own replies
- **MTU and MSS:** the byte-level cost of encapsulation, why wg0 uses 1420, and how MSS clamping prevents oversized packets from being dropped
- **NAT traversal:** why the side behind NAT has to initiate the connection and send keepalives
- **AWS networking:** security groups, Elastic IPs and source/destination checks for a self-managed VPN gateway
