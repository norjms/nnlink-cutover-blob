# eero gateway cutover - execution runbook

Built 2026-11-18. Read with `claude/home-wifi-overhaul-plan.md` (decision, final
topology, camera design, ONT notes) and `claude/network-map-2026-11-18.png` (James's map).
This runbook is the step-by-step. It is stored in the claude.ai Project so Claude can
read it during the cutover with no connection to the LAN. A second copy is in the
GitHub repo `norjms/nnlink-cutover-blob`.

## The one rule that outranks everything

**alice (192.168.1.50) and the Proxmox nodes stay hardwired to the US-XG-16 the entire
time. Never let their traffic ride the eero wireless mesh.** About 35 guests boot and run
their disks off alice NFS, including the domain controllers (nndc01 .15, nndc02 .18,
dc01 .208), AdGuard (.185/.12/.13), docker01 (.181, which runs Claude's MCP tooling and
the eero-toolkit), authentik (.173) and npmplus (.9). If alice's wired path is cut, all of
them hang on I/O until it returns. They are all on 192.168.1.0/24, so alice<->Proxmox NFS
is pure L2 switching on the XG-16 - it does not need the eero gateway and survives the
eero's ~8-minute config restarts as long as the XG-16 keeps switching. Keep that copper
path up from start to finish.

Second rule: **avoid two gateways on 192.168.1.1 at once.** The UDM is .1 today; the eero
will become .1. The UDM must vacate .1 before the eero takes it. Third rule: **bring up one
uplink at a time** when gluing a segment to an eero, or you create a bridging loop.

## Lifeline (already staged, verified 2026-11-18)

- Encrypted creds + the eero desired-state + topology: `nnlink-cutover.enc` in
  `github.com/norjms/nnlink-cutover-blob` (public, AES-256 ciphertext, useless without the
  passphrase). Fetch + decrypt proven from Claude's own environment with no LAN.
- Passphrase: held by James in 1Password. Claude never stores it.
- Decrypt: `openssl enc -d -aes-256-cbc -pbkdf2 -iter 600000 -in nnlink-cutover.enc -pass pass:'<phrase>' | tar -xf -`

## Pre-flight (before "go")

- [ ] UDM backup taken (done 0718) and a UniFi Protect export.
- [ ] eero firmware updated on all 7 nodes NOW (keeps that restart out of the window).
- [ ] UniFi OS Server VM155 (192.168.1.115) reachable; note which devices it has discovered.
- [ ] James has the SSID name and password the doorbell + 2 chimes use today (Claude cannot
      read the password; the eero must broadcast the same SSID so they rejoin).
- [ ] Cables staged: the ONT->eero run, the eero-gateway 10G run to the XG-16, the 2x 10G
      SFP+ pair (SG3452XP to XG-16), and the UDM SFP+ LAN -> SG3452XP camera link.
- [ ] James at the rack, phone in hand, ~45-60 min, tolerant of two ~8-min eero restarts.

## Current wiring (verified live 2026-11-18)

Gateway + controller today is the UDM Pro Max (Dream Machine Pro Max).

```
UDM Pro Max (gateway .1, UniFi controller)
  p10 (10G) -> US-48 .157  p49(uplink)
                 p50 -> US-XG-16 .254  p1(uplink)
                           p2  -> CRS317-1G-16S+ (MikroTik 74:4d:28:08:db:9c)
                           p9  -> alice2 .51 (10G)            <- alice NFS path
                           p10 -> alice1 .50 (10G) + DESKTOP-KHKS9MB .127
                           p12 -> ugreen2 .53 (10G)
                 p51 -> US-16-PoE .108  p17(uplink)
                           cameras p2,7,9,11,13,14,15,16  (PoE, G3/G3-bullet)
                           p1 -> AP AC Pro;  p4 -> AP IW HD
                           p3 -> US-8 .104;  p6 -> US-8 Theater .8
                           p12 -> USW Flex 2.5G .236 (p9 uplink, 100M) -> p4 AP U7 In-Wall
                 p47 -> AP U6 Lite .35 (100M)   <- doorbell + chimes live on this AP
  p3  (10G) -> USW Flex 2.5G Basement Master .7  p10(uplink)
                           p6 -> Nvidia .217;  p7 -> Sonos wired root (100M, STP must stay on)
  IW HD p3 -> eero Office Max 7 WAN (.238)   <- eero mesh currently gets internet via the UDM
```

eero "Migration" network 21760076 (all 7 nodes green, wired): Office Max 7 = gateway
(192.168.4.1), SG3452XP sits behind it (mgmt 192.168.4.120, temporary). eero LAN is
192.168.4.0/24 today - a separate island behind the UDM.

## Target wiring (James's map)

```
ONT (Nokia XS-010X-Q, 10G RJ45)
  -> eero Office Max 7 WAN (10G, static 46.110.77.225/26 gw .193)
     eero Max 7 LAN (10G) -> US-XG-16            <- eero is the gateway for 192.168.1.0/24
US-XG-16 (L2 core, adopted by UniFi OS Server VM155)
  -> alice .50/.51 (10G)           KEEP WIRED THROUGHOUT
  -> CRS317 (10G), ugreen (10G), Flex Basement Master (10G, Sonos root)
  -> SG3452XP  2x 10G SFP+ LAG (ports 49-50)
SG3452XP (Omada, behind nothing - hangs off the XG-16; clients + PoE cameras)
  -> client + camera PoE ports
  -> SFP+ 51 -> UDM Pro Max SFP+ LAN  (camera VLAN 120 only)
UDM Pro Max (cameras-only, Protect NVR, adopted-by-nobody - its own controller)
  WAN: SFP+ to the flat LAN, static 192.168.1.16
  LAN: 192.168.120.0/24 (camera VLAN), DHCP on
  -> cameras, U6 Lite (doorbell + chimes) on camera VLAN
```

## Sequence

Each phase ends with a verification gate. Claude checks live state before the next phase.
Physical cable moves are interactive: James moves one thing, Claude confirms, then the next.

### Phase 0 - prep, no outage
1. Update eero firmware (all nodes), wait for green.
2. Omada: on the SG3452XP create VLAN 120 "CAMERAS"; leave ports as-is for now.
3. Confirm VM155 is ready to adopt (reachable, licensed, sees devices).

### Phase 1 - give the eero real internet (no estate impact)
1. Move the eero gateway WAN cable from UDM IW-HD p3 to the ONT.
2. Set the eero WAN static: 46.110.77.225, mask 255.255.255.192, gw 46.110.77.193,
   DNS 1.1.1.1 / 1.0.0.1.
3. GATE: eero shows online via the ONT (eero app / Claude via eero cloud). The estate is
   untouched - still on the UDM gateway. eero LAN is still 192.168.4.x.
   - ONT ARP note: the ISP maps 46.110.77.225 to the UDM's MAC. If the eero has link but no
     internet, reboot the eero, then power-cycle the ONT, then call the ISP to clear ARP.

### Phase 2 - apply the eero 192.168.1.0/24 config while it is still isolated
The eero mesh must NOT be bridged to the 192.168.1.x L2 during this step (its backhaul is
wireless / eero-to-eero only; no eero LAN port is plugged into a main UniFi switch yet).
1. Claude (from docker01): `desired-from-unifi --exclude-mac c4:a8:16:47:ed:82` (fresh),
   then `plan`, then `apply --yes --allow-restart`.
2. This sets eero LAN = 192.168.1.0/24, pool .32-.254, DNS .11/.12/.13, 114 reservations,
   9 forwards. TWO ~8-min mesh restarts. The estate keeps running on the UDM throughout.
3. GATE: eero reports LAN 192.168.1.1, reservations/forwards present, mesh green.

### Phase 3 - the flip (short changeover; alice stays on copper)
Done quickly, in this order, to keep the dual-.1 window near zero:
1. UDM: disable DHCP on the Default network; unplug the UDM WAN; change the UDM LAN IP off
   .1 (set it to .16 now, or power down UDM routing). The 192.168.1.x L2 keeps switching on
   the XG-16 - alice/Proxmox NFS unaffected (L2, no gateway needed).
2. Connect the eero gateway LAN (10G) to the US-XG-16. The eero is now .1 for the whole L2.
3. GATE: from a wired host, gateway is the eero, DHCP leases from .32-.254, internal names
   resolve (AdGuard->DCs), internet works. Check alice still 10G, Proxmox `pvecm status`
   quorate (4 votes).

### Phase 4 - glue the rest with the eeros, finish wired runs
With the eero now the 192.168.1.x gateway, any eero LAN port is a 192.168.1.x access port
and the mesh carries segments that are not yet wired. One uplink at a time (no loops).
1. Bring up the SG3452XP 2x 10G LAG to the XG-16 (configure the LAG on both ends FIRST,
   then plug the second cable). SG mgmt returns to its reserved .198.
2. Move client/AP segments onto their target ports; let eeros mesh any gap, auto-switching
   to wired as runs complete.
3. GATE: all segments reachable; speed test from DRYAN01 near 5 Gbps; walk Wi-Fi.

### Phase 5 - adopt the UniFi switches to VM155, one at a time
Migrate by set-inform (keeps config, no factory reset) rather than forget-then-adopt where
possible. Releasing/forgetting a switch reboots it and drops everything downstream, so:
- Do leaf switches and APs first (US-8s, Flex .236, cameras' US-16-PoE last of the leaves).
- The US-XG-16 is the alice-critical core: migrate it LAST and by set-inform only. If it
  must reboot, know that alice NFS blips for the reboot window and the NFS-backed VMs hang
  until it returns (hard mounts recover). Have James ready; do it deliberately, not casually.
- Confirm each device shows "Connected" in VM155 before the next.

### Phase 6 - UDM to cameras-only (LAST)
Per `home-wifi-overhaul-plan.md` "Cameras" section:
1. Omada: VLAN 120 on SG3452XP; port 51 (to UDM) + camera ports + U6 Lite port untagged 120,
   PVID 120; the XG-16 LAG carries VLAN 1 only.
2. UDM: Default network -> 192.168.120.1/24 DHCP on; WAN static 192.168.1.16 gw .1 DNS .11;
   add eero reservation for the UDM WAN MAC at .16; repoint AD `udm` A record to .16.
3. Cable UDM SFP+ WAN to the XG-16; UDM SFP+ LAN to SG3452XP port 51.
4. Move each camera cable to a camera-VLAN port one at a time; confirm Connected in Protect.
5. Plug the U6 Lite into a camera-VLAN port (same SSID as today); confirm doorbell + both
   chimes rejoin and the chimes ring.
6. UDM firewall: allow tcp 443,7441,7447 from 192.168.1.0/24 to the gateway for local
   viewing; test the UniFi app on Wi-Fi and on cellular.
7. eero: re-run `desired-from-unifi` with `--exclude-mac` for every camera/doorbell/chime/
   U6 Lite MAC (now on the UDM side) and add the UDM WAN reservation at .16; apply.

## Verification (end)
- [ ] Wired speed test near 5 Gbps from DRYAN01.
- [ ] Walk Wi-Fi every room incl. deck, garage, basement.
- [ ] Protect: all cameras + doorbell connected; chimes ring.
- [ ] Port forwards from outside (web .9, mail .212).
- [ ] lan.nnlink.org names resolve on Wi-Fi clients.
- [ ] Proxmox: all 4 nodes quorate; no VM hung on NFS.

## Rollback
- Phase 1-3: move the ONT cable back to the UDM WAN, re-enable UDM DHCP, set UDM LAN back to
  .1, unplug the eero LAN from the XG-16. eero returns to the .4.x island. No data lost.
- Phase 5: a switch that will not re-adopt keeps forwarding on its last config; re-point its
  inform URL back, or adopt in VM155 fresh.
- Phase 6: move the ONT/camera cables back; UDM Default network back to .1; re-adopt cameras.

## Reference - full reservations and port map
The 85 UDM fixed-IP reservations and the complete per-port/MAC topology are in the encrypted
bundle (`reservations.txt`, `topology.txt`) and on ops01 at `/root/cutover/`. The eero
desired-state (114 reservations = 85 UDM + 29 live-pinned wired hosts, 9 forwards) is in
`desired-state.json` in the bundle and on the eero-toolkit volume.
