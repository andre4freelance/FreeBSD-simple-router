# FreeBSD SIMPLE ROUTER: DHCP Client, DHCP Server, NAT, DNS Cache

## 1. DHCP Client (WAN)
Edit `/etc/rc.conf`:
```conf
ifconfig_vtnet0="DHCP"
```

Restart networking:
```sh
service netif restart vtnet0
```

Cek IP:
```sh
ifconfig vtnet0
```

---

## 2. DHCP Server (LAN)
### Install ISC DHCP server
```sh
pkg install isc-dhcp44-server
```

### Konfigurasi interface di `/etc/rc.conf`
```conf
ifconfig_vtnet1="inet 192.168.10.1/24"
dhcpd_enable="YES"
dhcpd_ifaces="vtnet1"
```

### Konfigurasi DHCP server
File: `/usr/local/etc/dhcpd.conf`
```conf
default-lease-time 600;
max-lease-time 7200;
authoritative;

subnet 192.168.10.0 netmask 255.255.255.0 {
    range 192.168.10.100 192.168.10.200;
    option routers 192.168.10.1;
    option domain-name-servers 192.168.10.1;
}
```

### Jalankan DHCP server
```sh
service isc-dhcpd start
tail -f /var/db/dhcpd/dhcpd.leases
```

---

## 3. NAT (PF)
Aktifkan di `/etc/rc.conf`:
```conf
pf_enable="YES"
pf_rules="/etc/pf.conf"
gateway_enable="YES"
```

File: `/etc/pf.conf`
```pf
ext_if="vtnet0"    # WAN
int_if="vtnet1"    # LAN

set skip on lo

# NAT LAN → Internet
nat on $ext_if from $int_if:network to any -> ($ext_if)

# Allow LAN keluar
pass out on $ext_if from $int_if:network to any keep state

# Allow LAN masuk
pass in on $int_if
```

Reload rules:
```sh
service pf restart
pfctl -s nat
pfctl -ss
```

---

## 4. DNS Cache (Unbound)
Aktifkan di `/etc/rc.conf`:
```conf
local_unbound_enable="YES"
```

File: `/var/unbound/forward.conf`
```conf
forward-zone:
    name: "."
    forward-addr: 103.140.188.80
    forward-addr: 103.140.189.189
    forward-addr: 103.169.239.239
```

File: `/var/unbound/lan-zones.conf`
```conf
server:
    interface: 192.168.10.1
    access-control: 192.168.10.0/24 allow
    unblock-lan-zones: yes
    insecure-lan-zones: yes
```

Restart Unbound:
```sh
service local_unbound restart
```

### Test dari LAN client
```sh
nslookup google.com 192.168.10.1
```

---
## 5. Final version rc.conf
`/etc/rc.conf`:
```conf
[rc.conf](./rc.conf)
```

✅ Dengan konfigurasi ini:
- WAN: DHCP client aktif
- LAN: DHCP server berjalan
- NAT: LAN bisa akses internet via WAN
- DNS: FreeBSD cache resolver untuk LAN