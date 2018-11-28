SYNOPSIS:
  sudo natap nat start|restart|masq
  sudo natap nat start|restart|masq enp59s0 enx000ec6fb151b
  sudo natap nat start|restart|masq enp59s0 
  sudo natap nat start|restart|masq enp59s0 enx000ec6fb151b 
  sudo natap nat start|restart|masq enp59s0 enx000ec6fb151b 
  sudo natap nat start|restart|masq wlp60s0_sta 
  sudo natap nat start|restart|masq wlp60s0_sta enx000ec6fb151b
  sudo natap nat start|restart|masq wlp60s0_sta wlp60s0_ap enx000ec6fb151b
  sudo natap nat stop
  sudo natap nat status
  sudo natap nat flush
  sudo natap ap start wlp60s0
  sudo natap ap stop wlp60s0
  sudo natap ap create wlp60s0
  sudo natap ap delete wlp60s0
  sudo natap reset

  This command must be run with root privilege. It can:
         + Create a wireless access point(AP)
         + Share Internet with local LAN by using MASQUERADE, i.e., Linux is a gateway

  Dependencies:
         + hostapd, create access point
         + isc-dhcp-server, assign ip for associcated clients
         + network tools, e.g. ifup/ifdown/ip/iw/ifconfig/iwconfig

    Important configuration files: 
         + /etc/hostapd.conf
         + /etc/dhcp/dhcpd.conf
         + /etc/default/isc-dhcp-server
         + /etc/network/interfaces
         + /usr/share/bash-completion/completions/natap***

     You should modify configuration files to get it worked for you. This is just a guide.

    My setting on Debian gateway have three sets of rules:

       * Disallow incoming connections to eth1 (the external network interface)
       * Allow outgoing packets from the LAN (via eth0 and wlan0)
       * Allow established connections to return.

 Example configuration:


/etc/network/interfaces files

# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp59s0
allow-hotplug enp59s0
iface enp59s0 inet static
    address 172.20.16.39
    netmask 255.255.255.0
    gateway 172.20.16.1
    dns-nameservers 211.167.230.100
    dns-nameservers 211.167.230.200
    dns-nameservers 8.8.8.8
    dns-nameservers 8.8.4.4
# usb-lan
auto enx000ec6fb151b
allow-hotplug enx000ec6fb151b
iface enx000ec6fb151b inet static
    address 192.20.16.1 

# wlan
# client
allow_hotplug wlp60s0_sta
iface wlp60s0_sta inet dhcp
# manual not static
iface wlp60s0_ap inet manual
    #hostapd must be started after the interface is brought-up
    post-up /usr/sbin/hostapd -B /etc/
    up ip addr add  broadcast 192.20.39.1 dev wlp60s0_ap
    up service isc-dhcp-server restart
    pre-down killall hostapd
    down ip addr flush dev wlp60s0_ap


/etc/hostapd.conf

# natap_1-0
# Debug
# Minimal configuration for test
interface=wlp60s0_ap
driver=nl80211
ssid=ainray
#check by sudo iwlist wlp60s0 channel, more refer to iwlist
channel=1

# add more
#simply means 2.4GHz band, a means 5G
hw_mode=g
wme_enabled=1
# 802.11n support
ieee80211n=1
# check it by iw list
ht_capab=[HT40+][SHORT-GI-40][DSSS_CKK-40]

# authentication and Encryption
# 1 for wpa, 2 for wep, 3 both
auth_algs=1
ignore_broadcast_ssid=0
# WPA 2
wpa=2
wpa_passphrase=235111235
wpa_key_mgmt=WPA-PSK
# wpa_pairwise for WPA, rsn_pairwise for WPA2
wpa_pairwise=TKIP
rsn_pairwise=CCMP

# more
# limit frequencies used to those allowed in the country
ieee80211d=1
# china, check by isoqurey
country_code=CN
# QoS support
wmm_enabled=1


/etc/dhcp/dhcpd.conf
# dhcpd.conf
#
# Sample configuration file for ISC dhcpd
#

# option definitions common to all supported networks...
option domain-name-servers 211.167.230.100,211.167.230.200,8.8.8.8,8.8.4.4;

default-lease-time 600;
max-lease-time 7200;

# The ddns-updates-style parameter controls whether or not the server will
# attempt to do a DNS update when a lease is confirmed. We default to the
# behavior of the version 2 packages ('none', since DHCP v2 didn't
# have support for DDNS.)
ddns-update-style none;

# If this DHCP server is the official DHCP server for the local
# network, the authoritative directive should be uncommented.
authoritative;
subnet 192.20.39.0 netmask 255.255.255.0 {
  range 192.20.39.101 192.20.39.254;
  option subnet-mask 255.255.255.0;
  option broadcast-address 192.20.39.255;
  option routers 192.20.39.1;
}


/etc/default/isc-dhcp-server

# Defaults for isc-dhcp-server (sourced by /etc/init.d/isc-dhcp-server)

# Path to dhcpd's config file (default: /etc/dhcp/dhcpd.conf).
#DHCPDv4_CONF=/etc/dhcp/dhcpd.conf
#DHCPDv6_CONF=/etc/dhcp/dhcpd6.conf

# Path to dhcpd's PID file (default: /var/run/dhcpd.pid).
#DHCPDv4_PID=/var/run/dhcpd.pid
#DHCPDv6_PID=/var/run/dhcpd6.pid

# Additional options to start dhcpd with.
#	Don't use options -cf or -pf here; use DHCPD_CONF/ DHCPD_PID instead
#OPTIONS=""

# On what interfaces should the DHCP server (dhcpd) serve DHCP requests?
#	Separate multiple interfaces with spaces, e.g. "eth0 eth1".
#INTERFACESv4=""
#interfaces="wlp60s0"
INTERFACESv4="wlp60s0_ap"
#INTERFACESv6=


/usr/share/bash-completion/completions/natap
#!/bin/bash
_natap(){
    local cur prev opts
    local interfaces=$(ls /sys/class/net)
    COMPREPLY=()
    cur=${COMP_WORDS[COMP_CWORD]}
    prev=${COMP_WORDS[$((COMP_CWORD-1))]}
    case "$prev" in
        "natap")
            opts="ap inet nat reset status"
            ;;
        "nat")
            opts="restart start stop flush masq"
            ;;
        "restart"|"start"|"masq"|"stop")
            opts=${interfaces}
            ;;
        ap)
            opts="create delete start stop"
            ;;
        inet)
            opts="start stop"
            ;;
        create|delete)
            opts="wlp60s0"
            ;;
        ${interfaces})
            opts=${interfaces}
            ;;
    esac
    COMPREPLY=( $(compgen -W "${opts}" -- $cur) )
}
complete -F _natap natap
