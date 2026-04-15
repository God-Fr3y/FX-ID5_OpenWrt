# FX-ID5 LTE Modem Setup

OpenWrt 23.05.x · MediaTek MT7628 · Asrmicro LTE

---

## Part 1 — Connect to Main Router (Temporary, for Package Installation)

Before installing packages, bridge the device to your main router for internet access.

### Step 1 — Setup the admin password of OpenWrt and enable the Wi-Fi
Connect your computer using LAN cable to the LAN port of the router then open a browser and go to 192.168.1.1

Set a strong admin password
```sh
passwd
```

Check the AP interface
```sh
uci show wireless
```
Enable the AP, set to AP mode, and set a strong password
```sh
uci set wireless.default_radio0.disabled='0'
uci commit wireless
wifi up
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='Pass@1234'
uci commit wireless
wifi reload
``` 

### Step 2 — Change the LAN IP
Set the OpenWrt LAN IP to an unused address in your main router's subnet.

```sh
uci set network.lan.ipaddr='192.168.1.2'
uci commit network
/etc/init.d/network restart
```
Now unplug the cable from your computer and connect the it from OpenWrt LAN port to Main Router LAN port

### Step 3 — Disable DHCP on LAN and set Gateway and DNS

Your main router handles DHCP, so disable it here to avoid conflicts. SSH into it with your new OpenWrt LAN IP

```sh
uci set dhcp.lan.ignore='1'
uci commit dhcp
uci set network.lan.gateway='192.168.1.1'
uci set network.lan.dns='192.168.1.1'
service dnsmasq restart
```

Or via LuCI: **Network → Interfaces → br-lan → Edit → DHCP Server tab → check "Ignore interface"**

### Step 4 — Connect the Cable

> **Warning:** Connect main router's **LAN** port → OpenWrt's **LAN** port. Do NOT use the WAN port — it will cause a routing conflict.

### Step 5 — Test Connectivity

```sh
ping -c 4 8.8.8.8     # should be 0% packet loss
ping -c 4 google.com  # should resolve and respond
```

> If `8.8.8.8` works but DNS doesn't: `uci show network.lan.dns` — make sure it points to your main router.

---

## Part 2 — Install Packages

### Step 6 — Update Package Lists

```sh
opkg update
```

> Note: OpenWrt 23.05 uses `opkg`. The `apk` package manager is only in 24.x snapshot builds.

### Step 7 — Install USB Modem Drivers

The Asrmicro modem presents as RNDIS/CDC-NCM over USB.

```sh
opkg install kmod-usb-net-rndis   # RNDIS — exposes modem as usb0
opkg install kmod-usb-net-cdc-ncm # CDC-NCM — alternative USB networking
opkg install kmod-usb-acm         # CDC-ACM — AT command port (/dev/ttyACM0)
opkg install kmod-usb-serial      # optional, useful for debugging
```

### Step 8 — Load Kernel Modules

```sh
modprobe rndis_host
modprobe cdc_acm
```

Persist across reboots:

```sh
echo 'rndis_host' >> /etc/modules.d/rndis
echo 'cdc_acm'    >> /etc/modules.d/cdc-acm
```

### Step 9 — Verify Modem Detection

```sh
dmesg | grep -i 'usb\|rndis\|cdc\|ttyACM' | tail -20
ls /dev/ttyACM*   # expected: /dev/ttyACM0
ifconfig usb0     # expected: inet addr in carrier's network
```

### Step 10 — Install picocom

```sh
opkg install picocom
```

---

## Part 3 — Unlock the Modem

### Step 11 — Open AT Command Interface

```sh
picocom -b 115200 /dev/ttyACM0
# if no response, try: picocom -b 57600 /dev/ttyACM0
```

Type `AT` and press Enter. Expected response: `OK`. If not, try a different baud rate or `/dev/ttyACM1`.

### Step 12 — Remove SIM Lock

Inside the picocom session:
AT+CLCK="PN",0,"3@P#fT&30aTrs4L"

Expected: `OK`  
If `ERROR`: modem may already be unlocked, or the code doesn't apply to your unit.

> **Warning:** This unlock code is specific to this modem batch. Do not use on other devices. Repeated failed attempts may permanently lock the modem.

Exit picocom: `Ctrl+A` then `Ctrl+X`

### Step 13 — Verify Network Registration

Back in picocom:
AT+CREG?   # +CREG: 0,1  (1=home, 5=roaming)
AT+CEREG?  # +CEREG: 0,1  (LTE registration)
AT+CSQ     # +CSQ: 18,0   (signal quality; 99 = no signal)

---

## Part 4 — Configure wwan Interface

### Step 14 — Add wwan Interface

Via SSH:

```sh
uci set network.wwan=interface
uci set network.wwan.proto='dhcp'
uci set network.wwan.device='usb0'
uci commit network

# assign to wan firewall zone
uci add_list firewall.@zone[1].network='wwan'
uci commit firewall

service network restart
service firewall restart
```

Or via LuCI: **Network → Interfaces → Add new interface**
- Name: `wwan`, Protocol: `DHCP client`, Device: `usb0`
- Under Firewall Settings tab: assign to `wan` zone

### Step 15 — Verify LTE Connection

```sh
ifconfig usb0         # expect inet addr:100.x.x.x (carrier NAT is normal)
route -n              # wwan/usb0 should be default route
ping -c 4 -I usb0 8.8.8.8
```

---

## Part 5 — Switch to Standalone LTE Mode

### Step 16 — Disconnect from Main Router

Unplug the cable. OpenWrt will now route through LTE (usb0) as default WAN.

### Step 17 — Re-enable DHCP on LAN

```sh
uci delete dhcp.lan.ignore
uci commit dhcp
service dnsmasq restart
```

### Step 18 — Final Check

```sh
ping -c 4 8.8.8.8
ping -c 4 google.com
ip route show   # confirm usb0 is default route
```

---

## Interface Summary

| Interface | Role | IP / Details | Notes |
|-----------|------|--------------|-------|
| br-lan | LAN bridge | 192.168.1.2 (AP mode) / 192.168.1.1 (standalone) | Bridges eth0.1 + phy0-ap0 |
| usb0 | LTE WAN | 100.x.x.x (carrier NAT) | Asrmicro modem via RNDIS |
| phy0-ap0 | Wi-Fi AP | Bridged to br-lan | MT7628 802.11n 2.4 GHz |
| eth0.1 | LAN VLAN | Bridged to br-lan | Physical LAN port |
| eth0.2 | WAN VLAN | Unused | Physical WAN port (LTE replaces) |
