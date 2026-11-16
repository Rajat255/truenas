# TrueNAS Setup Progress

## Status
- [x] Installed TrueNAS from USB drive onto spare laptop
- [x] First reboot succeeded, console setup menu reached
- [ ] Get the NAS onto the network (blocked: no Ethernet, WiFi only)
- [ ] Log into web UI from another device
- [ ] Create storage pool
- [ ] Set up shares / apps

## Issue log

### 2026-11-07: No network, only WiFi available
TrueNAS does not support WiFi. There is no WiFi option in the console menu or web UI,
and the wireless drivers are left out on purpose. The NAS needs a wired link of some kind.

Options under consideration:
1. WiFi-to-Ethernet bridge (travel router in repeater/bridge mode, or a mesh node with a LAN port). Best long-term fix.
2. USB-to-Ethernet adapter, if the laptop has no Ethernet port (used together with option 1).
3. Phone USB tethering. Temporary only: the NAS ends up on the phone's private subnet, so other devices can't reach it.

Update: phone USB tethering connected. TrueNAS sees the tethered interface, but it has no
working connection yet. Next step is to turn on DHCP on that interface (console option 1),
then apply, test, and persist the change.

Update: `ip -br addr` shows the tethering interface as DOWN. Next step is to check whether
it's turned off in software (it has no config yet) or has no physical link to the phone
(NO-CARRIER).

Update: the interface now shows state UNKNOWN with IP 10.189.6.203/24. The link works.
UNKNOWN is normal for USB tethering. Remaining steps: make the DHCP config persistent
(console option 1), then open http://10.189.6.203 in the phone's browser.

Update: want to reach the web UI from the Mac. The Mac is on home WiFi, and the NAS is on the
phone's private USB tethering subnet, so they can't reach each other directly.
Options: (a) put the Mac on the same phone's hotspot and test whether the phone routes between
hotspot and USB clients, (b) Tailscale on the NAS and the Mac (needs a pool for apps),
(c) WiFi-to-Ethernet bridge hardware (permanent fix).

Note: the phone and the Mac being on the same WiFi doesn't help. The phone hides the NAS behind
NAT and drops new connections that come in from WiFi to the USB tethering subnet.
Workaround: run a port forwarder on the phone (Termux + socat) so that
http://<phone-wifi-ip>:8080 forwards to 10.189.6.203:80.

Update: Mac connected to the phone's hotspot, but http://10.21.202.68/ doesn't load. The NAS
address changed (it was 10.189.6.203), because Android picks a new random subnet each time
tethering restarts. Next: confirm the NAS's current IP, then ping it from the Mac. If the ping
fails, the phone is isolating hotspot clients from USB clients, so use Termux + socat.

Result: Mac on the hotspot is 10.215.191.222/24, with the phone as gateway at 10.215.191.171.
The NAS is on 10.21.202.x, a different subnet. Ping from Mac to 10.21.202.68 got 100% loss.
The phone doesn't pass traffic between hotspot and USB clients, so the hotspot option fails.
Next: Termux + socat on the phone, then open http://10.215.191.171:8080 from the Mac.

Open questions:
- Does the laptop have a built-in Ethernet port?
- Is there a spare router, travel router, or mesh node available?
- What does the console screen show for "The web user interface is at:"?
