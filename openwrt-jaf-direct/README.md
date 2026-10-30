# Direct Jio ODU to OpenWrt with 464XLAT

This guide connects a Jio outdoor unit (ODU) directly to OpenWrt, without the
Jio indoor unit (IDU), and provides IPv4 service over Jio's IPv6-only access
network using 464XLAT.

The reference system is a GL.iNet Flint 2 / GL-MT6000 running OpenWrt 25.12.6.
Linux device names, switch topology, VLAN ID, IDU identity values, and the
NAT64 prefix must be verified on each installation.

## Choose the ODU port first: direct Ethernet or DSA

OpenWrt can expose physical Ethernet connectors in two different ways. This
choice determines the device name used by every later command.

### Direct Ethernet netdev, such as `eth1`

`eth1` is a Linux Ethernet device backed directly by a MAC controller. It is
not a universal name for "WAN" and must not be copied merely because it worked
on the reference router. On the Flint 2, `eth1` was identified as an
independent, non-bridge device by correlating cable link changes with
`ip link` output. The Jio ODU then obtained DHCPv6 successfully through VLAN
1015 on `eth1`.

When the router exposes a suitable direct port, this is the preferred path:

- it is already separate from the LAN bridge;
- it does not require removing a switch port from `br-lan`;
- VLAN device ordering is simpler; and
- the DSA-specific CLAT refresh hook is normally unnecessary.

The resulting tagged device is `eth1.1015`.

### DSA switch port, such as `lan4`

DSA (Distributed Switch Architecture) exposes individual ports of the
router's Ethernet switch as Linux devices such as `lan1`, `lan2`, or `lan4`.
These ports usually belong to `br-lan`. To use one for the ODU, it must first
be removed from the LAN bridge and then used as the parent of a VLAN device.

The reference router now uses the ODU permanently on `lan4`, producing
`lan4.1015`. This path is useful when the direct Ethernet netdev is not mapped
to the desired connector or a particular switch port must be used. It has one
extra consideration: on the tested Flint 2, CLAT could start before the global
Jio DHCPv6 address was fully available after a cold boot. The supplied
hotplug hook corrects that ordering issue.

Do not configure both paths for the same ODU. Set `JIO_PARENT` to the device
that actually follows the ODU cable:

```sh
# Direct Ethernet example
JIO_PARENT='eth1'

# DSA example: use this instead when the ODU is connected to lan4
# JIO_PARENT='lan4'

JIO_VLAN_ID='1015'
JIO_VLAN_DEVICE="${JIO_PARENT}.${JIO_VLAN_ID}"
printf 'ODU parent=%s VLAN device=%s\n' "$JIO_PARENT" "$JIO_VLAN_DEVICE"
```

### Identify the correct device instead of guessing

Start from a LAN or Wi-Fi SSH session that will remain reachable while the ODU
cable is moved:

```sh
ubus call system board
ip -br link
ip -d link show
uci show network | grep -E "\.name='br-lan'|\.ports="
```

Run a link monitor, unplug and reconnect only the ODU cable, and note which
device changes state:

```sh
ip monitor link
```

In another session, inspect the candidates:

```sh
ip -br link show eth1
ip -br link show lan4
readlink -f /sys/class/net/eth1/device 2>/dev/null
readlink -f /sys/class/net/lan4/device 2>/dev/null
```

If `ethtool` is installed, carrier state provides another confirmation:

```sh
ethtool eth1 | grep -E 'Link detected|Speed|Duplex'
ethtool lan4 | grep -E 'Link detected|Speed|Duplex'
```

A `lanN` candidate listed in the `br-lan` ports is a DSA switch port. An
independent `ethN` device outside that bridge is a direct netdev. Select the
device whose carrier follows the ODU cable; labels and numbering vary between
targets and firmware builds.

## Packet path and required values

```text
IPv4 application or LAN client
        |
        v
wanxlat zone: wanb / 464-wanb
        | NAT46 / CLAT
        v
IPv6 destination under the provider NAT64 prefix
        |
        v
wan zone: wanb6 / eth1.1015 or lan4.1015
        |
        v
Jio NAT64 -> IPv4 Internet
```

The reference values were:

```text
Jio VLAN:            1015
Direct VLAN device:  eth1.1015
DSA VLAN device:     lan4.1015
Native IPv6 iface:   wanb6
CLAT iface:          wanb
CLAT runtime device: 464-wanb
NAT64 prefix:        64:ff9b::/96
```

You also need your own IDU WAN MAC. The tested service required that MAC and a
hardware-style DHCPv6 client ID derived from it. Never use identity values or
leases from another subscriber.

## Determine the NAT64 prefix before removing the IDU

`64:ff9b::/96` is the RFC 6052 well-known NAT64 prefix.

1. Check the IDU's WAN, CLAT, 464XLAT, or NAT64 status page and record the prefix including its prefix length.

## Prerequisites and packages

- OpenWrt 25 with `apk` rather than `opkg`.
- Root SSH access through a LAN or Wi-Fi path unaffected by the ODU changes.
- The verified ODU parent device, VLAN ID, IDU WAN MAC, and NAT64 prefix.

```sh
apk update
apk add 464xlat kmod-nat46 curl tcpdump

apk info -e 464xlat
apk info -e kmod-nat46
apk info -e firewall4
command -v curl
command -v tcpdump
```

Ensure the repositories match the running OpenWrt release and kernel.

## Record and inspect the starting state

```sh
ubus call system board
ip -br link
ip -4 addr
ip -4 route show default
uci export network >"/tmp/network-before-jio-$(date +%s).uci"
uci export firewall >"/tmp/firewall-before-jio-$(date +%s).uci"
uci changes network
uci changes firewall
```

Resolve unrelated staged UCI edits before continuing.

## Build the DHCPv6 client ID

Use the WAN MAC printed or reported by your own Jio IDU:

```sh
JIO_IDU_WAN_MAC='aa:bb:cc:dd:ee:ff'
JIO_DHCPV6_CLIENT_ID="00030001$(printf '%s' "$JIO_IDU_WAN_MAC" | tr -d ':')"
printf 'MAC=%s\nCLIENT_ID=%s\n' \
        "$JIO_IDU_WAN_MAC" "$JIO_DHCPV6_CLIENT_ID"
```

The client ID is `0003 0001` followed by the six-byte MAC. On the reference
connection, a UUID-style DUID received no DHCPv6 Advertise.

## Prepare a DSA port when applicable

Skip this section for a direct device such as `eth1`.

Find the UCI device section for `br-lan` and verify that the selected DSA port
is currently a member:

```sh
JIO_PARENT='lan4'
JIO_LAN_BRIDGE_SECTION="$(
        uci show network |
                sed -n "s/^\(network\.[^.]*\)\.name='br-lan'$/\1/p"
)"
printf 'LAN bridge section: %s\n' "$JIO_LAN_BRIDGE_SECTION"
uci get "${JIO_LAN_BRIDGE_SECTION}.ports"
```

Stop if the bridge section is empty or ambiguous. Keep SSH on Wi-Fi or a
different LAN port, then remove only the chosen ODU port:

```sh
uci del_list "${JIO_LAN_BRIDGE_SECTION}.ports=${JIO_PARENT}"
```

## Create VLAN 1015 explicitly

OpenWrt can infer an 802.1Q device from a name such as `eth1.1015`, but an
explicit UCI device section makes the parent, VLAN ID, name, and spoofed MAC
unambiguous. Use it for either port model:

```sh
JIO_PARENT='eth1'                 # or lan4
JIO_VLAN_ID='1015'
JIO_VLAN_DEVICE="${JIO_PARENT}.${JIO_VLAN_ID}"
JIO_IDU_WAN_MAC='aa:bb:cc:dd:ee:ff'

uci set network.jio_odu_vlan='device'
uci set network.jio_odu_vlan.type='8021q'
uci set network.jio_odu_vlan.ifname="$JIO_PARENT"
uci set network.jio_odu_vlan.vid="$JIO_VLAN_ID"
uci set network.jio_odu_vlan.name="$JIO_VLAN_DEVICE"
uci set network.jio_odu_vlan.macaddr="$JIO_IDU_WAN_MAC"
uci set network.jio_odu_vlan.dadtransmits='0'
```

The MAC can instead be assigned to a dedicated parent device section, as was
done during the initial `eth1` test. Do not create competing MAC overrides on
both parent and VLAN. Confirm the pending configuration with:

```sh
uci changes network
uci show network.jio_odu_vlan
```

The device appears in `ip link` only after the configuration has been applied.

## Configure native IPv6 and CLAT

Set the values discovered from your service:

```sh
JIO_VLAN_DEVICE='eth1.1015'       # or lan4.1015
JIO_DHCPV6_CLIENT_ID='00030001aabbccddeeff'
JIO_NAT64_PREFIX='64:ff9b::/96'

uci set network.wanb6='interface'
uci set network.wanb6.device="$JIO_VLAN_DEVICE"
uci set network.wanb6.proto='dhcpv6'
uci set network.wanb6.reqaddress='force'
uci set network.wanb6.reqprefix='no'
uci set network.wanb6.sourcefilter='0'
uci set network.wanb6.metric='2048'
uci set network.wanb6.multipath='off'
uci set network.wanb6.iface_464xlat='0'
uci set network.wanb6.clientid="$JIO_DHCPV6_CLIENT_ID"

uci set network.wanb='interface'
uci set network.wanb.proto='464xlat'
uci set network.wanb.tunlink='wanb6'
uci set network.wanb.ip6prefix="$JIO_NAT64_PREFIX"
uci set network.wanb.metric='2048'
uci set network.wanb.multipath='off'
```

Do not set a manual `ip6addr` on `wanb`. Netifd generates the CLAT address and
records it in `/tmp/464-wanb-anycast`.

Review and apply:

```sh
uci changes network
uci show network.jio_odu_vlan
uci show network.wanb6
uci show network.wanb
uci commit network
service network reload
```

A network reload can interrupt SSH. Reconnect over the unaffected LAN path.

## DSA-only CLAT startup hook

Skip this section when using the direct `eth1` path unless testing proves the same source-prefix race exists.

On the validated `lan4.1015` cold boot, DHCPv6 and firewall4 recovered, but `wanb` could start before the global Jio address was published and select a
LAN ULA as its CLAT source. 

Create a conditional hook for auto refresh at `/etc/hotplug.d/iface/95-jio-clat-refresh`

```
#!/bin/sh

# A DSA VLAN device can bring wanb6 and its dependent static 464xlat interface up in the same netifd transaction. In that case 464xlatcfg may select the router's LAN ULA before the Jio DHCPv6 address is available. Refresh CLAT only when its /64 differs from the active Jio /64.

[ "$INTERFACE" = "wanb6" ] || exit 0

case "$ACTION" in
	ifup)
		;;
	ifupdate)
		[ "$IFUPDATE_ADDRESSES" = "1" ] || exit 0
		;;
	*)
		exit 0
		;;
esac

(
	lock_dir="/var/run/jio-clat-refresh.lock"
	mkdir "$lock_dir" 2>/dev/null || exit 0
	trap 'rmdir "$lock_dir"' EXIT HUP INT TERM

	# Let netifd finish publishing the DHCPv6 address and initial CLAT state.
	sleep 2

	wan6_address="$(
		ubus call network.interface.wanb6 status 2>/dev/null |
			jsonfilter -e '@["ipv6-address"][0].address'
	)"
	clat_address="$(cat /tmp/464-wanb-anycast 2>/dev/null)"

	[ -n "$wan6_address" ] && [ -n "$clat_address" ] || {
		logger -t jio-clat-refresh "Skipped: wanb6 or CLAT address is unavailable"
		exit 0
	}

	wan6_prefix="$(printf '%s\n' "$wan6_address" | cut -d: -f1-4)"
	clat_prefix="$(printf '%s\n' "$clat_address" | cut -d: -f1-4)"

	[ "$wan6_prefix" = "$clat_prefix" ] && exit 0

	logger -t jio-clat-refresh "Refreshing wanb after DHCPv6 prefix change"
	ifdown wanb
	ifup wanb
) &

exit 0

```

The hook runs only for `wanb6` address events, waits briefly, compares the Jio
and generated CLAT prefixes, and restarts only `wanb` when they differ. Its
runtime lock prevents overlapping refreshes. The `/etc/sysupgrade.conf` entry
causes the hook to be included in the configuration backup restored by
`sysupgrade`.

When migrating an already-running router, invoke it once after `wanb6` is up:

```sh
ACTION=ifup INTERFACE=wanb6 \
        /etc/hotplug.d/iface/95-jio-clat-refresh
sleep 4
```

## Verify native IPv6 and NAT64 before CLAT

Use the actual VLAN device in every command:

```sh
JIO_VLAN_DEVICE='eth1.1015'       # or lan4.1015

ubus call network.interface.wanb6 status
ip -6 addr show dev "$JIO_VLAN_DEVICE"
ip -6 route show default
ping -6 -I "$JIO_VLAN_DEVICE" -c 3 -W 2 2405:200:800::11
ping -6 -I "$JIO_VLAN_DEVICE" -c 3 -W 2 64:ff9b::101:101
```

Expected results are a global IPv6 lease, an IPv6 default route through Jio,
native IPv6 replies, and NAT64-prefixed replies. If DHCPv6 receives no
Advertise, investigate the physical port, carrier, VLAN, IDU WAN MAC, and
hardware-style client ID before touching CLAT or firewall rules.

## Verify CLAT creation

```sh
ubus call network.interface.wanb status
ip -d link show 464-wanb
cat /tmp/464-wanb-anycast
lsmod | grep nat46
ip -6 rule show
ip -6 route show table prelocal
nft -a list chain inet fw4 srcnat | grep 464-wanb
```

Expected results:

- `wanb` is up with `l3_device` `464-wanb`;
- `464-wanb` is UP, POINTOPOINT, and NOARP;
- `nat46` is loaded;
- `/tmp/464-wanb-anycast` contains a generated CLAT address; and
- firewall4 has IPv4 SNAT for `464-wanb` to `192.0.0.1`.

The generated address need not appear as a conventional address on the VLAN
device. On the DSA path, it should use the same provider prefix as the current
global `wanb6` address, not the LAN ULA.

## Create an isolated firewall zone for CLAT

NAT46 reinjects translated IPv6 packets as forwarded traffic arriving from
`464-wanb`. If `wanb` shares the normal `wan` zone, firewall4 can drop that
WAN-to-WAN forwarding. Standard ICMPv6 exceptions can make ping work while
TCP and UDP still fail.

wan, wanxlat will be the two zones for wan. If you do not care about backup internet or multiwan, you can just update the existing wan for both and do self forward. If you want to keep another port for normal DHCP Clients then its better to separate the zones.

Create the CLAT zone, LAN forwarding, and a rule scoped to the discovered
NAT64 prefix:

```sh
JIO_NAT64_PREFIX='64:ff9b::/96'

uci set firewall.wanxlat='zone'
uci set firewall.wanxlat.name='wanxlat'
uci set firewall.wanxlat.input='REJECT'
uci set firewall.wanxlat.output='ACCEPT'
uci set firewall.wanxlat.forward='DROP'
uci set firewall.wanxlat.masq='1'
uci set firewall.wanxlat.mtu_fix='1'
uci add_list firewall.wanxlat.network='wanb'

uci set firewall.lan_wanxlat='forwarding'
uci set firewall.lan_wanxlat.src='lan'
uci set firewall.lan_wanxlat.dest='wanxlat'

uci set firewall.allow_464xlat_nat64='rule'
uci set firewall.allow_464xlat_nat64.name='Allow-464XLAT-NAT64'
uci set firewall.allow_464xlat_nat64.src='wanxlat'
uci set firewall.allow_464xlat_nat64.dest='wan'
uci set firewall.allow_464xlat_nat64.device='464-wanb'
uci set firewall.allow_464xlat_nat64.family='ipv6'
uci set firewall.allow_464xlat_nat64.proto='all'
uci set firewall.allow_464xlat_nat64.dest_ip="$JIO_NAT64_PREFIX"
uci set firewall.allow_464xlat_nat64.target='ACCEPT'
```

Validate and apply:

```sh
uci changes firewall
fw4 check
fw4 print | grep -F -B 3 -A 5 \
        -e 'Allow-464XLAT-NAT64' \
        -e 'ubus:wanb[464xlat]' \
        -e 'accept_to_wanxlat'
uci commit firewall
service firewall reload
```

The rendered critical rule should resemble:

```nft
iifname "464-wanb" ip6 daddr 64:ff9b::/96 counter jump accept_to_wan
```

## End-to-end validation

```sh
JIO_VLAN_DEVICE='eth1.1015'       # or lan4.1015

fw4 network wanb
fw4 network wanb6
fw4 device 464-wanb
fw4 device "$JIO_VLAN_DEVICE"

ping -I 464-wanb -c 3 -W 2 1.1.1.1
curl -4 -v --interface 464-wanb \
        --connect-timeout 8 --max-time 15 https://1.1.1.1/
curl -4 -v --interface 464-wanb \
        --connect-timeout 8 --max-time 15 https://one.one.one.one/
ping -6 -I "$JIO_VLAN_DEVICE" -c 3 -W 2 2405:200:800::11
nft -a list chain inet fw4 forward_wanxlat
ip -4 route show default
```

The expected firewall mapping is:

```text
wanb        -> wanxlat
wanb6       -> wan
464-wanb    -> wanxlat
ODU VLAN    -> wan
```

## Bounded concurrent packet capture

A capture is useful only when a matching test runs at the same time. Replace
the VLAN device before executing this bounded script:

```sh
JIO_VLAN_DEVICE='eth1.1015'       # or lan4.1015
c1=/tmp/clat-tcp.txt
c2=/tmp/vlan-tcp.txt

tcpdump -ni 464-wanb -c 80 -vv 'tcp port 443' >"$c1" 2>&1 & p1=$!
tcpdump -ni "$JIO_VLAN_DEVICE" -c 80 -vv \
        'ip6 and tcp' >"$c2" 2>&1 & p2=$!

cleanup() {
        kill "$p1" "$p2" 2>/dev/null
        wait "$p1" "$p2" 2>/dev/null
        rm -f "$c1" "$c2"
}
trap cleanup EXIT HUP INT TERM

sleep 2
curl -4 -v --interface 464-wanb \
        --connect-timeout 8 --max-time 15 https://1.1.1.1/
sleep 2
kill "$p1" "$p2" 2>/dev/null
wait "$p1" "$p2" 2>/dev/null

printf '%s\n' '=== 464-wanb ==='
cat "$c1"
printf '%s\n' "=== $JIO_VLAN_DEVICE ==="
cat "$c2"
```

Healthy TCP shows the IPv4 flow on `464-wanb`, translated IPv6 on the ODU
VLAN, a returning SYN-ACK, and bidirectional TLS/HTTP traffic.

## Troubleshooting by the first failed layer

### The expected physical device does not gain carrier

Re-run `ip monitor link` while reconnecting the ODU. Verify the actual port
mapping rather than renaming interfaces. For DSA, also confirm the selected
port is no longer in `br-lan`.

### VLAN device is absent or has the wrong parent

```sh
uci show network.jio_odu_vlan
ip -d link show eth1.1015          # or lan4.1015
```

The detailed link output must show VLAN protocol 802.1Q, ID 1015, and the
correct parent.

### No DHCPv6 lease

```sh
ubus call network.interface.wanb6 status
logread | grep -Ei 'odhcp6c|dhcpv6|wanb6' | tail -n 100
tcpdump -ni eth1.1015 -c 30 -vv 'udp port 546 or udp port 547'
```

Use `lan4.1015` instead when applicable. If Solicit leaves but no Advertise
returns, recheck VLAN 1015, the IDU WAN MAC, and the hardware-style client ID.
If no Solicit leaves, inspect interface configuration and netifd logs.

### Native IPv6 works but the NAT64 test fails

Reconfirm the NAT64 prefix from the working IDU or provider DNS64. Check the
IPv6 route to the synthesized destination and capture it on the ODU VLAN:

```sh
ip -6 route get 64:ff9b::101:101
tcpdump -ni eth1.1015 -c 20 -vv 'host 64:ff9b::101:101'
```

### NAT64 works but CLAT is absent

```sh
apk info -e 464xlat
apk info -e kmod-nat46
lsmod | grep nat46
ubus call network.interface.wanb status
logread | grep -Ei 'nat46|464xlat|wanb' | tail -n 100
```

Missing nat46 debugfs or sysfs parameter files alone does not prove failure.

### CLAT ping works but TCP times out

Inspect `fw4 network wanb`, `forward_wanxlat`, the scoped NAT64 allow counter,
and both sides of a concurrent curl. IPv4 SYNs on `464-wanb` with no translated
IPv6 on the ODU VLAN usually point to firewall zone forwarding or the NAT46
path. Translated IPv6 leaving the VLAN without a reply points upstream.

### DSA path works for IPv6 but CLAT fails after boot

```sh
ubus call network.interface.wanb6 status
cat /tmp/464-wanb-anycast
logread | grep -F jio-clat-refresh
ls -l /etc/hotplug.d/iface/95-jio-clat-refresh
```

If the native Jio address is global but the CLAT address uses a LAN ULA,
invoke the hook. When both prefixes already match, the hook exits without
restarting `wanb`.
