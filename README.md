# openfortivpn-android-build
This repo builds openfortivpn for arm64 Android devices

## Setup
1. `cd /sdcard/Download`
2. `curl -LO https://github.com/Real96/openfortivpn-android-build/releases/latest/download/pppd`
3. `curl -LO https://github.com/Real96/openfortivpn-android-build/releases/latest/download/openfortivpn`
4. `su -c 'mkdir -p /data/local/bin /data/local/etc/ppp /data/local/etc/openfortivpn /data/local/var/run'`
5. `su -c 'mv /sdcard/Download/pppd /sdcard/Download/openfortivpn /data/local/bin/'`
6. `su -c 'chmod 755 /data/local/bin/pppd /data/local/bin/openfortivpn'`

## Verify if binaries work correctly
1. `su -c '/data/local/bin/openfortivpn --version'`
2. `su -c '/data/local/bin/pppd --version'`
3. `su -c 'grep -a -o "/data/local/etc/ppp[^ ]*" /data/local/bin/pppd | head -3'`
4. `su -c 'ls -l /dev/ppp'`

## Set openfortivpn config file
1. `su -c 'vi /data/local/etc/openfortivpn/config'`
```
host = <gateway_ip>
port = <port>
username = <username>
password = <password>
set-routes = 0
set-dns = 0
pppd-use-peerdns = 0
persistent = 10
```
2. `su -c 'chmod 600 /data/local/etc/openfortivpn/config'`
3. `su -c '/data/local/bin/openfortivpn -c /data/local/etc/openfortivpn/config -v'` # this will fail, it's necessary to get the `trusted-cert` SHA-256 value
4. `su -c 'pkill -TERM openfortivpn'`
5. `su -c 'vi /data/local/etc/openfortivpn/config'` # write the `trusted-cert` SHA-256 value you got above
```
host = <gateway_IP>
port = <port>
username = <username>
password = <password>
set-routes = 0
set-dns = 0
pppd-use-peerdns = 0
persistent = 10
trusted-cert = <cert_SHA-256>
```

## Set ppp ip-up
1. `su -c 'vi /data/local/etc/ppp/ip-up'`
```
#!/bin/sh

PATH=/system/bin:/system/xbin
IP=/system/bin/ip
IFACE="$1"
TABLE=200
NETS="<subnet1 subnet2 ...>"

$IP route flush table $TABLE 2>/dev/null
$IP route add default dev "$IFACE" table $TABLE
while $IP rule del pref 9000 2>/dev/null; do :; done
for n in $NETS; do $IP rule add pref 9000 to "$n" lookup $TABLE; done
$IP link set dev "$IFACE" mtu 1354
```
2. `su -c 'chmod 755 /data/local/etc/ppp/ip-up'`

## Set ppp ip-down
1. `su -c 'vi /data/local/etc/ppp/ip-down'`
```
#!/bin/sh

PATH=/system/bin:/system/xbin
IP=/system/bin/ip
while $IP rule del pref 9000 2>/dev/null; do :; done
$IP route flush table 200 2>/dev/null
```
2. `su -c 'chmod 755 /data/local/etc/ppp/ip-down'`

## Manual start
`su -c '/data/local/bin/openfortivpn -c /data/local/etc/openfortivpn/config -v > /data/local/tmp/openfortivpn.log 2>&1' &`

## Verify working state
`su -c 'tail /data/local/tmp/openfortivpn.log'` # Expected output: `INFO:   Tunnel is up and running.`

## Verify routing
1. `su -c 'ip addr show ppp0'`
2. `su -c 'ip rule show | grep 9000'`
3. `su -c 'ip route show table 200'`

## Toggle script
1. `su -c 'vi /data/local/bin/forti-toggle.sh'`
```
#!/bin/sh

BIN=/data/local/bin
CFG=/data/local/etc/openfortivpn/config
LOG=/data/local/tmp/openfortivpn.log

start_vpn() {
  pgrep -x openfortivpn >/dev/null && return
  "$BIN/openfortivpn" -c "$CFG" >"$LOG" 2>&1 &
}

stop_vpn() {
  pkill -TERM openfortivpn

  i=0
  while [ $i -lt 10 ]; do
    pgrep -x openfortivpn >/dev/null || break
    sleep 1; i=$((i+1))
  done

  pkill -TERM pppd 2>/dev/null
  sleep 1
  while ip rule del pref 9000 2>/dev/null; do :; done
  ip route flush table 200 2>/dev/null
}

case "$1" in
  up|start|on)   start_vpn ;;
  down|stop|off) stop_vpn ;;
  *) if pgrep -x openfortivpn >/dev/null; then stop_vpn; else start_vpn; fi ;;
esac
```
2. `su -c 'chmod 700 /data/local/bin/forti-toggle.sh'`

## Usage
* Turn the VPN on: `su -c '/data/local/bin/forti-toggle.sh up'`
* Turn the VPN off: `su -c '/data/local/bin/forti-toggle.sh down'`
* Toggle the VPN: `su -c '/data/local/bin/forti-toggle.sh'`

## NOTE
`trusted-cert` pins the gateway certificate. When the company renews it, openfortivpn will fail with a validation error and print the new SHA-256: just replace the `trusted-cert` value in the config.
