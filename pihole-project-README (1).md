# pihole-dns-sinkhole

Self-hosted DNS sinkhole on Kali Linux that blocks ads, trackers, and malicious domains network-wide — explored as a hands-on dive into DNS-layer security and centralized logging.

A self-hosted DNS sinkhole that blocks ads, trackers, and known malicious domains across every device on my home network — set up as a hands-on project to understand DNS-layer security and network-wide filtering, independent of any single browser or device.

## What it does

Pi-hole sits between every device on the network and the internet at the DNS resolution layer. When a device asks "what's the IP address for this domain?", Pi-hole checks the request against curated blocklists before answering:

- **Blocked domain** → Pi-hole returns a non-answer (`0.0.0.0` / NXDOMAIN), so the ad/tracker request fails before it ever loads
- **Clean domain** → Pi-hole forwards the request to an upstream resolver (Cloudflare/Quad9) and returns the real answer

Because this happens at the DNS level rather than per-browser, it protects **every device on the network** — phones, laptops, smart TVs, IoT devices — without installing anything on each one individually.

## Why I built this

I wanted hands-on experience with:
- DNS as an attack surface and a defensive control point, not just "the thing that turns names into IPs"
- Centralized logging — Pi-hole's query log is effectively a mini-SIEM for DNS traffic, letting me see exactly what every device on my network is contacting
- Lightweight self-hosted infrastructure, as a contrast to heavier security tooling (SIEM platforms, IDS/IPS) I've worked with elsewhere

## Installation walkthrough

**Environment:** Kali Linux, running as the DNS server/host on the network — a deliberate choice to reuse existing infrastructure rather than spin up a dedicated Raspberry Pi.

### Step 1 — Run the official installer

```bash
curl -sSL https://install.pi-hole.net | bash
```

<img width="657" height="551" alt="image" src="https://github.com/user-attachments/assets/39d504d8-97e6-4f61-b432-15e5d4aa14bd" />
[Pi-hole installer start — root/sudo check]

Before doing anything else, the installer runs a safety check — notably refusing to proceed without root privileges, and prompting for `sudo` even when invoked as a normal user:

```
[i] Root user check
[i] Script called with non-root privileges
    The Pi-hole requires elevated privileges to install and run
    Please check the installer for any concerns regarding this requirement
    Make sure to download this script from a trusted source

[✓] Sudo utility check
[✓] Root user check
```

This exists because Pi-hole needs to bind to port 53 (the standard DNS port) and modify system-level DNS configuration — both of which require root. A good concrete example of *why* a tool demands elevated privileges, rather than treating `sudo` as a reflexive prefix.

From here, the installer walks through an interactive setup: choosing an upstream DNS provider (Cloudflare/Quad9), selecting blocklists, and configuring the network interface to use.

### Step 2 — Installation completes with connection details

![Pi-hole installation complete — DNS and dashboard details]

```
[i] You may now configure your devices to use the Pi-hole as their DNS server
[i] Pi-hole DNS (IPv4): 192.168.100.143
[i] Pi-hole DNS (IPv6): fd7a:115c:a1e0::b01:48de
[i] If you have not done so already, the above IP should be set to static.
[i] View the web interface at http://pi.hole/admin or http://192.168.100.143/admin

[i] Web Interface password: [redacted — generated automatically, changeable via 'pihole setpassword']
[i] This can be changed using 'pihole setpassword'

[✓] Installation complete!
```

Two things worth noting here: Pi-hole auto-generates a random admin password at install time rather than defaulting to something predictable like `admin/admin` — a small but meaningful security default worth appreciating, especially compared to some IoT/appliance devices that don't bother. The install log itself is also saved (`/etc/pihole/install.log`), useful for troubleshooting if anything goes wrong later.

**Network configuration:** router's DHCP/DNS settings then updated to point all client devices at `192.168.100.143`, so filtering applies automatically across the whole network without touching each device individually.

### Step 3 — Dashboard, live and filtering

<img width="1264" height="621" alt="image" src="https://github.com/user-attachments/assets/20f0a6a6-5528-4889-900e-b74e53fbe4d9" />
[Pi-hole dashboard showing live stats]

Logging into `http://192.168.100.143/admin` with the generated password shows the live dashboard — hostname `kali`, status **Active**:

| Metric | Value |
|---|---|
| Total Queries | 1,804 |
| Queries Blocked | 7 |
| Percentage Blocked | 0.4% |
| Domains on Blocklists | 78,216 |
| Active Clients | 2 |

The **Total queries** graph shows real traffic bursts throughout the day, correlating with actual browsing activity on the network — confirming Pi-hole is genuinely intercepting and logging every DNS request, not just sitting idle.

### If managing from a Windows machine

A few PowerShell commands are useful for testing and verifying the DNS setup from a Windows client on the same network:

```powershell
# Confirm which DNS server your machine is actually using
Get-DnsClientServerAddress

# Manually point this machine at the Pi-hole for testing (before changing it at the router level)
Set-DnsClientServerAddress -InterfaceAlias "Wi-Fi" -ServerAddresses ("<pihole-ip>")

# Clear the local DNS cache so new queries actually go through Pi-hole
Clear-DnsClientCache

# Test that a known ad domain resolves to Pi-hole's blocked response
Resolve-DnsName doubleclick.net -Server <pihole-ip>
```

That last command is the real proof-of-work: a successful block shows `0.0.0.0` or an NXDOMAIN-style failure instead of a real IP, confirming Pi-hole is actually intercepting and filtering the query rather than the client silently falling back to another DNS server.

## Results

- **1,804** total DNS queries processed, **7 blocked** (0.4%) in early testing — low block percentage reflects the short monitoring window so far, expected to rise with normal daily usage and additional blocklists
- **78,216** domains loaded across blocklists, actively checked against every query
- **2 active clients** on the network routing through Pi-hole
- Query volume visibly spikes during active browsing periods, confirming real-time traffic is flowing through and being logged correctly

## What I learned

- How DNS sinkholing works as a security control, and why it's a meaningfully different approach from browser-level ad blocking
- Reading and interpreting DNS query logs to spot unusual device behavior — a skill that transfers directly to SOC-style log analysis, just scoped to DNS instead of security events
- Basic Linux service management and networking configuration on a lightweight headless system

## Tech stack

`Pi-hole` · `Linux` · `DNS` · `Docker` *(if applicable)*

---

*Part of my ongoing cybersecurity learning journey toward a SOC analyst role. See my other projects: [mini security log dashboard / SIEM project link]*
