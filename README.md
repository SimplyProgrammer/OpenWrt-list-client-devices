# OpenWrt-list-client-devices
Lightweight OpenWrt script to show all client devices no matter the connection type (Wired, Wireless, DHCP or Static).

This can be considered a programmatic and better way to LuCi's Status > Routing - IP Neighbors

## Example output
```
IP Address                MAC Address       Vendor (MAC)              Hostname                 Iface          Method     State
-----------------------------------------------------------------------------------------------------------------------------
10.10.123.1               e8:39:35:ed:d6:30 Hewlett Packard           Hp                       Pi (eth4)      DHCP       STALE
10.10.123.4               28:70:4e:32:36:5e Ubiquiti Inc              USW-Flex-Mini            Pi (eth4)      DHCP       DELAY
10.10.2.100               00:13:1c:21:ca:84 LiteTouch, Inc.           *                        Pa (eth1)      DHCP       STALE
10.10.2.4                 b0:6e:bf:bb:5c:61 ASUSTek COMPUTER INC.     PC                       Pa (eth1)      DHCP       REACHABLE
169.254.216.15            b0:6e:bf:bb:5c:61 ASUSTek COMPUTER INC.     PC                       Pa (eth1)      DHCP       STALE
10.10.5.123               a6:0d:e5:cc:21:aa -                         iPad                     - (wlan2)      DHCP       STALE
192.168.0.1               9c:53:22:de:10:43 TP-Link Systems Inc       -                        wan (eth0)     Static/?   REACHABLE
192.168.0.10              00:13:1b:4a:a0:a9 -                         -                        wan (eth0)     Static/?   STALE
192.168.0.238             -                 -                         -                        wan (eth0)     Static/?   FAILED
```
Note that Vendor might not be displayed properly due the the request limits on https://api.macvendors.com when run for first couple of times...

## Install
You need to have basic commands (ip, grep, awk, sed etc...) available.
Also bash (if you do not have it already):
```
opkg update
opkg install bash
```
Switch to `bash`
```
cat << 'THE_END' > /bin/list-clients && chmod +x /bin/list-clients && sed -i -e 's/\t//g' -e '/^[[:space:]]*$/d' -e '/^# /d' /bin/list-clients && grep -qxF '/bin/list-clients' /etc/sysupgrade.conf || echo '/bin/list-clients' >> /etc/sysupgrade.conf
#!/bin/sh

LEASES="/tmp/dhcp.leases"
ETHS="/etc/ethers"
MAC_VENDORS="/etc/mac-vendors.db"

# Mac vendor cahce
MAC_CACHE=$(cat "$MAC_VENDORS" 2>/dev/null)

# Print header
printf "%-28s %-17s %-34s %-25s %-20s %-10s %-10s\n" "IP Addr" "MAC Addr" "Vendor (MAC)" "Hostname" "Iface" "Method" "State"
printf "%s\n" "----------------------------------------------------------------------------------------------------------------------------------------------------"

elap=0

ip neigh show | sort | while read -r l; do
	ip=$(echo "$l" | awk '{print $1}')
	ifc=$(echo "$l" | sed -n 's/.* dev \([^ ]*\).*/\1/p')
	mac=$(echo "$l" | sed -n 's/.* lladdr \([^ ]*\).*/\1/p')
	state=$(echo "$l" | awk '{print $NF}')

	[ -z "$ip" ] && ip="-"
	[ -z "$ifc" ] && ifc="-"
	[ -z "$state" ] && state="-"

	# Hostname & Method detection
	host=$(awk -v mac="$mac" '$2 == mac {print $4}' "$LEASES" 2>/dev/null)
	meth="Static/?"
	if [ -n "$host" ]; then
		meth="DHCP"
	elif grep -qi "^[[:space:]]*$mac[[:space:]]" "$ETHS" 2>/dev/null; then
		meth="Static"
		[ -z "$host" ] && host=$(awk -v mac="$mac" 'BEGIN{IGNORECASE=1} $1==mac {print $2}' "$ETHS" 2>/dev/null)
	fi
	[ -z "$host" ] && host="-"

	# Mac vend lookup with local cache
	vend="-"
	pref=$(echo "$mac" | tr '[:lower:]' '[:upper:]' | cut -d: -f1-3)

	[ -n "$pref" ] && vend=$(echo "$MAC_CACHE" | grep -i "^$pref=" | head -n1 | cut -d= -f2-)

	if [ -z "$vend" ] && [ "$pref" != "---" ] && [ "$elap" -lt 4 ]; then
		t0=$(date +%s)
		resp=$(wget -qO- "https://api.macvendors.com/$mac" 2>/dev/null)
		elap=$(($(date +%s) - t0))

		if [ -n "$resp" ]; then
			vend="$resp"
			echo "$pref=$vend" >> "$MAC_VENDORS"
			MAC_CACHE="${MAC_CACHE}${MAC_CACHE:+
}$pref=$vend"
		fi
	fi
	[ -z "$mac" ] && mac="-"
	[ -z "$vend" ] && vend="-"

	owrtIfc=$(uci show network 2>/dev/null | grep "$ifc" | cut -d. -f2 | cut -d= -f1 | head -n1)
	[ -z "$owrtIfc" ] && owrtIfc="-"

	printf "%-28s %-17s %-34.34s %-25.25s %-20s %-10s %-10s\n" "$ip" "$mac" "$vend" "$host" "$owrtIfc ($ifc)" "$meth" "$state"
done

THE_END
```

Now you can simply run it as `list-clients` and add it as a Custom command (System > Custom commands) to LuCi!

Note that command might not work properly if not run with root privileges.
Also, the `list-clients` itself does not require bash to run so after you are done with the install script, you can uninstall bash if you want...
