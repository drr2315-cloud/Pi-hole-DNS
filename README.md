# Pi-hole-DNS
DNS Server built using raspberry pi zero W
A self-hosted DNS filtering and remote access solution built on a Raspberry Pi Zero W, serving as the DNS and DHCP infrastructure for my entire home network. This project replaced my router's stock DHCP service, blocks ads and trackers for every device on the network, and provides secure remote access (with ad-blocking included) from anywhere via WireGuard.

This README documents not just the working configuration, but the failures along the way and how each one was isolated and resolved — because the troubleshooting was the real project.


Architecture Overview

                        Internet
                           │
                  ┌────────┴────────┐
                  │  Verizon CR1000B │  ← Routing, Wi-Fi, NAT only
                  │  (DHCP disabled) │     UDP 51820 forwarded to Pi
                  └────────┬────────┘
                           │
                  ┌────────┴────────┐
                  │ Raspberry Pi     │  192.168.1.157 (static)
                  │ Zero W           │
                  │                  │
                  │  • Pi-hole       │  ← DNS filtering + DHCP server
                  │  • WireGuard     │  ← VPN (PiVPN), clients get
                  │  • DuckDNS cron  │     Pi-hole DNS through tunnel
                  └────────┬────────┘
                           │
              All home devices (leases from Pi-hole,
              DNS filtered network-wide)

   Remote devices ──WireGuard tunnel──► home network
   (ad-blocking follows the phone onto cellular)

Stack

ComponentRoleRaspberry Pi Zero WHost (single-core ARMv6, 512MB RAM — sufficient for DNS/DHCP workloads)Raspberry Pi OS Lite (32-bit)Headless OS, managed entirely over SSHPi-holeDNS sinkhole + DHCP server for the LANPiVPN / WireGuardSelf-hosted VPN, UDP 51820DuckDNS + cronDynamic DNS so the VPN endpoint survives public IP changesMicro-USB OTG Ethernet adapterWired uplink (ASIX chipset, driver-free on Raspberry Pi OS)

Key Configuration Decisions

Wired over wireless. The Pi serves DNS for the whole network, so it runs on Ethernet via a micro-USB OTG adapter rather than the Zero W's 2.4GHz Wi-Fi. DNS infrastructure shouldn't depend on radio conditions.

True static IP on the device, not just a router reservation. The Pi originally held a DHCP reservation — until the Pi became the DHCP server, creating a chicken-and-egg dependency on its own service at boot. Fixed by configuring a static address directly on the Pi (nmtui: 192.168.1.157/24, gateway 192.168.1.1, DNS 127.0.0.1), making the Pi's addressing independent of any DHCP server, including itself.

DHCP pool moved out of the server's address space. Pi-hole's DHCP range was shrunk to 192.168.1.100–150 so the pool can never collide with the server's own address.

Pi-hole as DHCP server. The Verizon CR1000B does not expose a LAN-side DNS override in its DHCP settings, so the router's DHCP was disabled and Pi-hole took over lease assignment — the standard workaround for locked-down ISP routers, with the bonus of per-client visibility in the Pi-hole dashboard.

VPN clients inherit Pi-hole DNS. PiVPN was configured to hand the Pi-hole address to WireGuard clients, so ad-blocking follows my phone onto cellular.

Unattended upgrades enabled. Security patching happens automatically.

Setup Summary


Image the SD card — Raspberry Pi Imager with OS customization: hostname, user account, SSH enabled pre-boot (fully headless provisioning, no monitor ever attached)
Static addressing — nmtui, static IP outside the DHCP pool
Pi-hole install — curl -sSL https://install.pi-hole.net | bash, bound to eth0
DHCP migration — enable Pi-hole DHCP first, then disable the router's (no gap with no DHCP authority)
PiVPN install — curl -L https://install.pivpn.io | bash, WireGuard on UDP 51820
Dynamic DNS — DuckDNS update script + cron entry (*/5 * * * *)
Port forward — UDP 51820 → 192.168.1.157 on the router
Client enrollment — pivpn add + pivpn -qr, scanned from the WireGuard mobile app
Config backup — Pi-hole Teleporter export stored off-device


Verification Testing


ipconfig /all on LAN clients confirms leases and DNS served by the Pi
Dashboard query log confirms network-wide filtering across all client devices
VPN validated from cellular only (testing from inside the LAN can false-fail due to hairpin NAT limitations on consumer routers)
Pi-hole query log confirms VPN clients resolving through the tunnel from the 10.63.76.0/24 VPN subnet
Remote admin access to the Pi-hole dashboard verified over the tunnel
Reboot testing confirms static addressing and all services recover without intervention



Troubleshooting Log (The Real Curriculum)

Incident 1: The seven-blink boot failure

Symptom: Pi refused to boot; ACT LED blinked 7 times in a repeating pattern ("kernel not found"). Persisted across two different SD cards — one budget card, one genuine SanDisk — with a verified-complete boot partition.

Investigation: Cheap SD media was the initial suspect (a real and common failure mode) and was replaced. When the identical failure survived the card swap, the suspect list collapsed to what was common across all attempts. Root cause: the board was a Pi Zero W (32-bit ARMv6), but every flash had used the 64-bit OS image — which contains only kernel8.img. The Zero W's firmware looks for the 32-bit kernel.img, which doesn't exist on that image. Both cards had been flawlessly written with an OS the hardware could never boot.

Lesson: Verify the hardware inventory before troubleshooting the stack above it. When a symptom survives a component swap, the swapped component was never the cause.

Incident 2: Imaging from a legacy macOS system

Flashing was initially done from a 2013-era iMac on macOS High Sierra, which produced a chain of distinct failures — each with a different root cause behind a similar-looking error:

ErrorActual causeFixPermission denied on writeSD adapter's physical lock switchSlide the lockunmount dissented by PID 0Kernel/Spotlight holding the volumediskutil unmountDisk forceOperation not permittedTypo'd device path (/dev/drdisk2) — dd tried to create a file inside /devRead the error precisely; fix the pathInput/output error at byte 0Raw device path (rdisk) failing through the aging USB stackWrite via the buffered device insteadWrite "completed" but card left in factory stateBuffered write cache: dd returned while gigabytes were still flushing in the background; unplugging the reader discarded the cached dataAlways sync after buffered writes and wait for it to return; eject properly

Lesson: A progress indicator (or a completed dd) is not data on disk. Write caching means the job isn't done until the flush is done — the same reason "safe to remove hardware" exists on Windows. The workflow was ultimately migrated to Raspberry Pi Imager on a modern PC, which adds a byte-level verify pass after writing.

Incident 3: Production outage — the DHCP single point of failure

Symptom: Pi rebooted and hung mid-boot; every device in the house lost internet. Clients whose DHCP renewals failed during the outage self-assigned 169.254.x.x addresses — at which point they couldn't even reach the router's admin page to restore service.

Response:


Restored client access by assigning a temporary manual IP to a workstation (static 192.168.1.50, gateway 192.168.1.1) to reach the router
Re-enabled the router's DHCP as an emergency rollback, restoring the household while the Pi was diagnosed offline
Triaged the Pi by LED behavior; checked for undervoltage with vcgencmd get_throttled (returned 0x0 — power supply acquitted)
Root-caused the addressing fragility to the DHCP reservation chicken-and-egg problem (see Key Configuration Decisions) and converted the Pi to true static addressing
Performed a planned reboot test with the rollback safety net still in place before returning DHCP duty to the Pi


Mitigations now in place: device-level static IP, Teleporter config backups, reduced logging verbosity to limit SD wear, and a documented rollback procedure (router DHCP re-enable + lease renewal).

Lesson: A planned change with a rollback path and an unplanned outage can be the same technical event with completely different risk profiles. Also: when you run the only DHCP server, you run production.

Incident 4: VPN handshake failures

Symptom: WireGuard client stuck in a handshake-retry loop; tunnel never established.

Investigation (layered):


Server: pivpn -d self-checks all green; ss -ulnp confirmed WireGuard listening on 51820/udp — server acquitted
Name resolution: nslookup confirmed DuckDNS resolving to the correct public IP — dynamic DNS acquitted
Client config: endpoint verified correct in the phone profile
Path: prepared a tcpdump -i eth0 udp port 51820 capture on the Pi to establish ground truth on whether packets were arriving at all


The tunnel began working before the capture ran — most plausibly delayed application of the router's port-forward rule. Logged as resolved; root cause unconfirmed (suspected delayed rule application), which is sometimes the honest answer.

Lesson: Debug in layers with a distinct test per layer (service → DNS → config → packet path), and packet capture is the ground truth beneath all configuration. Also, test VPNs from outside the network — hairpin NAT makes inside-the-LAN tests unreliable on consumer routers.


Skills Demonstrated


Linux server administration (headless Debian, systemd services, cron, nmtui/NetworkManager)
DNS and DHCP architecture — scopes, leases, reservations vs. device-level static addressing, migration between DHCP authorities with zero-gap cutover
VPN deployment — WireGuard key-based peering, NAT traversal, port forwarding, dynamic DNS
Network troubleshooting methodology — OSI-layered isolation, packet capture, reading LED diagnostic codes, distinguishing similar errors with different root causes
Storage forensics — partition tables, write caching behavior, raw vs. buffered device I/O, media verification
Operational practice — rollback planning, config backups, post-incident review, documentation
