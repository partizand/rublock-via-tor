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

��������� tor
-------------

    opkg install tor
	
��� ��������� ������������ ������ ������ /opt/etc/tor/torrc

���������� � ����� `user tor` �� `user admin`



