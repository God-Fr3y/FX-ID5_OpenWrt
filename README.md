# FX-ID5 LTE Modem Setup

OpenWrt 25.05.x · MediaTek MT7628 · Asrmicro LTE

---

## Download Firmware

https://firmware-selector.openwrt.org/?version=25.12.2&target=ramips%2Fmt76x8&id=mediatek_mt7628an-eval-board

---

## Part 1 — Connect to Main Router (Temporary, for Package Installation)

Before installing packages, bridge the device to your main router for internet access.

### Step 1 — Set Admin Password and Enable Wi-Fi

Connect your computer to the FX-ID5 using a LAN cable, then SSH in:

```sh
ssh root@192.168.1.1
passwd
```

Check the wireless interface:

```sh
uci show wireless
```

Enable Wi-Fi, set AP mode, and set a password:

```sh
uci set wireless.default_radio0.disabled='0'
uci set wireless.default_radio0.encryption='psk2'
uci set wireless.default_radio0.key='Pass@1234'
uci commit wireless
wifi up
```

Disconnect the LAN cable and connect your computer to the OpenWrt Wi-Fi.

### Step 2 — Disable DHCP, Change LAN IP, Set Gateway and DNS

Set the LAN IP to an unused address in your main router's subnet.

Disable DHCP first:

```sh
uci set dhcp.lan.ignore='1'
uci commit dhcp
service dnsmasq restart
```

Change the LAN IP, gateway, and DNS:

```sh
uci set network.lan.ipaddr='192.168.1.2'
uci set network.lan.netmask='255.255.255.0'
uci set network.lan.gateway='192.168.1.1'
uci set network.lan.dns='192.168.1.1'
uci commit network
service network restart
```

> You will lose the SSH session after `service network restart` — this is expected. Reconnect to the OpenWrt Wi-Fi and SSH back in at the new IP: `ssh root@192.168.1.2`

### Step 3 — Connect the Cable

> **Warning:** Connect the main router's **LAN** port → OpenWrt's **LAN** port. Do NOT use the WAN port — it will cause a routing conflict.

### Step 4 — Test Connectivity

```sh
ping -c 4 8.8.8.8     # should be 0% packet loss
ping -c 4 google.com  # should resolve and respond
```

> If `8.8.8.8` works but DNS doesn't: run `uci show network.lan.dns` and confirm it points to your main router's IP.

---

## Part 2 — Install Packages

### Step 5 — Update System and Install USB Modem Drivers

```sh
apk update && apk upgrade
```

The Asrmicro modem presents as RNDIS/CDC-NCM over USB:

```sh
apk add kmod-usb-net-rndis    # RNDIS — exposes modem as usb0
apk add kmod-usb-net-cdc-ncm  # CDC-NCM — alternative USB networking
apk add kmod-usb-acm          # CDC-ACM — AT command port (/dev/ttyACM0)
apk add kmod-usb-serial       # optional, useful for debugging
```

### Step 6 — Load Kernel Modules

```sh
modprobe rndis_host
modprobe cdc_acm
```

Persist across reboots:

```sh
echo 'rndis_host' >> /etc/modules.d/rndis
echo 'cdc_acm'    >> /etc/modules.d/cdc-acm
```

### Step 7 — Verify Modem Detection

```sh
dmesg | grep -i 'usb\|rndis\|cdc\|ttyACM' | tail -20
ls /dev/ttyACM*   # expected: /dev/ttyACM0
ifconfig usb0     # expected: inet addr in the carrier's network
```

### Step 8 — Install picocom

```sh
apk add picocom
```

---

## Part 3 — Unlock the Modem

### Step 9 — Open AT Command Interface

```sh
picocom -b 57600 /dev/ttyACM0
```

Type `AT` and press Enter. Expected response: `OK`. If no response, check `ls /dev/ttyACM*` for the correct device node.

### Step 10 — Remove SIM Lock

Inside the picocom session:
AT+CLCK="PN",0,"3@P#fT&30aTrs4L"

Expected: `OK`  
If `ERROR`: the modem may already be unlocked, or this code does not apply to your unit.

> **Warning:** This unlock code is specific to this modem batch. Do not use it on other devices. Repeated failed attempts may permanently lock the modem.

Exit picocom: `Ctrl+A` then `Ctrl+X`

### Step 11 — Verify Network Registration

Back in picocom:
AT+CREG?   # +CREG: 0,1  (1=home, 5=roaming)
AT+CEREG?  # +CEREG: 0,1  (LTE registration)
AT+CSQ     # +CSQ: 18,0   (signal quality; 99 = no signal)

---

## Part 4 — Configure wwan Interface

### Step 12 — Add wwan Interface

```sh
uci set network.wwan=interface
uci set network.wwan.proto='dhcp'
uci set network.wwan.device='usb0'
uci commit network
```

Assign to the wan firewall zone:

```sh
uci add_list firewall.@zone[1].network='wwan'
uci commit firewall
service network restart
service firewall restart
```

### Step 13 — Verify LTE Connection

```sh
ifconfig usb0          # expect inet addr:100.x.x.x (carrier NAT is normal)
route -n               # wwan/usb0 should appear as the default route
ping -c 4 -I usb0 8.8.8.8
```

---

## Part 5 — Switch to Standalone LTE Mode

### Step 14 — Disconnect from Main Router

Unplug the LAN cable. OpenWrt will now route all traffic through LTE (usb0) as the default WAN.

### Step 15 — Re-enable DHCP on LAN

```sh
uci delete dhcp.lan.ignore
uci commit dhcp
service dnsmasq restart
```

### Step 16 — Final Check

```sh
ping -c 4 8.8.8.8
ping -c 4 google.com
ip route show   # confirm usb0 is the default route
```

---

## Interface Summary

| Interface | Role       | IP / Details             | Notes                      |
|-----------|------------|--------------------------|----------------------------|
| br-lan    | LAN bridge | 192.168.1.2 (AP mode)    | Bridges eth0.1 + phy0-ap0  |
| usb0      | LTE WAN    | 100.x.x.x (carrier NAT)  | Asrmicro modem via RNDIS   |
| phy0-ap0  | Wi-Fi AP   | Bridged to br-lan        | MT7628 802.11n 2.4 GHz     |
| eth0.1    | LAN VLAN   | Bridged to br-lan        | Physical LAN port          |
| eth0.2    | WAN VLAN   | Unused                   | Physical WAN port          |
