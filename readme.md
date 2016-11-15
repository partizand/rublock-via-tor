������������� tor ��� ������ ����������
=======================================

���� � �������� ���������. �������� �� �����!!!

������� ���� ��������� ������������ tor ������������ ��� ������ ����������� ��������. ��� ��������� � ����� ������ �������� ����� �������������� ��������� ������������� ����������.

�� ������� https://github.com/DontBeAPadavan/rublock-via-vpn/

����������
----------

* �������� � ipset (����� ������ Padavan, ����� nano),
* ���������� ����������� Entware

��������� ��������
------------------

�������� ������ �������

    opkg update

���������� ����������� ������:

    opkg install lua

�������� ������ ������������ ������� ��� dnsmasq, �������� ��� �����������:

    mkdir /opt/lib/lua
    wget --no-check-certificate -O /opt/lib/lua/ltn12.lua https://raw.githubusercontent.com/diegonehab/luasocket/master/src/ltn12.lua
    wget --no-check-certificate -O /opt/bin/blupdate.lua https://raw.githubusercontent.com/partizand/rublock-via-tor/master/opt/bin/blupdate.lua
    chmod +x /opt/bin/blupdate.lua

��������� ������ �� ����������:

    blupdate.lua

� ���������� ������ ������� ����� ������������ ����� /opt/etc/rublock.dnsmasq � /opt/etc/rublock.ips, ������� ����� ������������ �����. �������� ��������� ������ ����� ����� ������ ��� ���������� ������ ��������������� ��������, ��������, ��� � �����.
�� ����� ���� ����������� ����� rublock.ips ������������ ��� �������� /opt/etc/init.d/S10iptables. ������� �� ������ ���� ������ ipset. � �������� �� ������ �� ������������. ��� ��� ��� ��������� ip ������� ����� ����� ������������� ������. ��� � ��� ��� ����� (��� ������ �����)

��������� tor
-------------

    opkg install tor
	
��� ��������� ������������ ������ ������ /opt/etc/tor/torrc

���������� � ����� `user tor` �� `user admin`

���� ����� ������ tor, ���������������� � ��������� IP � ������
    
	#SOCKSPort 192.168.0.1:9100 # Bind to this address:port too.

��

    SOCKSPort 192.168.1.1:9100 # Bind to this address:port too.
	
������ ������ ��������� socks proxy � �������� � ���������� 192.168.1.1:9100. ���������� ������ ����� tor

**��������! ��� ����������� ����� �� ����������!**

*������ �� ������������ ����� 1*

������ ���� ��� �������� ������� �� ����� �������� � socks ����������. ��� �������� ����� ������ ���������� ������ tor + iptables. ��� ����� ���������� �������� � tor ����� ����������� �������������. � ����� ������ �� ������������ � ������� ������� ������ ������� �������� ������ �� �����. ���� ������ ����� ����� ��������� ����� tor.
������������� ����������� ������ � Tor

��� ����� � ���������������� ����� /opt/etc/tor/torrc ��������� � �����:

AutomapHostsOnResolve 1
TransPort 9040
DNSPort 9053

��� ����� ���������� ������ ����� ������� ���� 9040 (�� ������ ��� 127.0.0.1).

*����� ������������� ����� 1*

��������� ��������
------------------

�������������� /opt/etc/init.d/S10iptables:

	#!/bin/sh

	case "$1" in
	start|update)
			# add iptables custom rules
			echo "firewall started"
			[ -d '/opt/etc' ] || exit 0
			# Create new rublock ipset and fill it with IPs from list
			if [ ! -z "$(ipset --swap rublock rublock 2>&1 | grep 'given name does not exist')" ] ; then
					ipset -N rublock iphash
					for IP in $(cat /opt/etc/rublock.ips) ; do
							ipset -A rublock $IP
					done
			fi
			iptables -A PREROUTING -t mangle -m set --match-set rublock dst,src -j MARK --set-mark 1
			;;
	stop)
			# delete iptables custom rules
			echo "firewall stopped"
			;;
	*)
			echo "Usage: $0 {start|stop|update}"
			exit 1
			;;
	esac

�� ����� ���� ���� ����� ��������� ������ ������� ������� ������� �� �����. ����� �������� ����

	# Create new rublock ipset and fill it with IPs from list
	if [ ! -z "$(ipset --swap rublock rublock 2>&1 | grep 'given name does not exist')" ] ; then
			ipset -N rublock iphash
			for IP in $(cat /opt/etc/rublock.ips) ; do
					ipset -A rublock $IP
			done
	fi
	iptables -A PREROUTING -t mangle -m set --match-set rublock dst,src -j MARK --set-mark 1

*������ �� ������������ ����� 2*	
	
�� ����� ���� ������, ������� ������ �������� ������ ������
    
	iptables -A PREROUTING -t mangle -m set --match-set rublock dst,src -j MARK --set-mark 1

����� �������� �� �������� �� ���� ������ tor. ���� �� ���� ���.

��� ����� ���������� ������������ iptables � ������ REDIRECT �to-port. ��� ������������� ���� ����� ���������� ������������� ������� � ���������� ����� iptables-mod-nat-extra. � ��������� ������ �� �������� ������ ����:

    iptables v1.4.6: unknown option `--to-ports'
	
�� ���� ���� �� ����� ����� � padavan ���� ��������. �� ����� ����� ����� ���������� �� �����������.

���������� ����� ����� ���������� ��������

    opkg install kmod-ipt-nat-extra
	
����� ��������� ������ ���������� ����������� ������ �� ���������� ������ ������:

iptables -t nat -A PREROUTING -p tcp --match-set rublock --dport 80 -j REDIRECT --to-ports 9040

��� �������� � ������������� ������ ������ rublock ������ �� 80-� ���� �� ���� 9040, �� ������� ��� ��� � ��������� ���������� ������.

*����� ������������� ����� 2*

� ���-���������� ������� �� �������� Customization > Scripts �������������� ���� Run After Router Started, ��������������� ��� �������:

    modprobe ip_set_hash_ip
    modprobe xt_set











