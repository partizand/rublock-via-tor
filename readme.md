Использование tor для обхода блокировок
=======================================

**Пока в процессе настройки. Работать не будет!!!**

Решение ниже позволяет использовать tor исключтельно для списка блокируемых ресурсов. При обращении к любым другим ресурсам будет использоваться привычное провайдерское соединение.

Вопрос возможности использования tor с процессором роутера остается открытым.

По мотивам https://github.com/DontBeAPadavan/rublock-via-vpn/

Требования
----------

* Прошивка с ipset (любые сборки Padavan, кроме nano),
* Развёрнутый репозиторий Entware

Установка скриптов
------------------

Обновите список пакетов

    opkg update

Установите необходимые пакеты:

    opkg install lua

Скачайте скрипт формирования конфига для dnsmasq, сделайте его исполняемым:

    mkdir /opt/lib/lua
    wget --no-check-certificate -O /opt/lib/lua/ltn12.lua https://raw.githubusercontent.com/diegonehab/luasocket/master/src/ltn12.lua
    wget --no-check-certificate -O /opt/bin/blupdate.lua https://raw.githubusercontent.com/partizand/rublock-via-tor/master/opt/bin/blupdate.lua
    chmod +x /opt/bin/blupdate.lua

Запустите скрипт на исполнение:

    blupdate.lua

В результате работы скрипта будут сформированы файлы /opt/etc/rublock.dnsmasq и /opt/etc/rublock.ips, которые будут использованы далее. Повторно запускать скрипт имеет смысл только для обновления списка заблокированных ресурсов, например, раз в месяц.

На самом деле перестройка файла rublock.ips бессмысленна без рестарта /opt/etc/init.d/S10iptables. Который на основе него делает ipset. А стартует он только по перезагрузке. Так что при изменении ip ресурса нужно будет перезагружать роутер. Или я не так понял (что скорее всего)

Установка tor
-------------

    opkg install tor
	
Для настройки конфигурации правим конфиг /opt/etc/tor/torrc

Исправляем в конце `user tor` на `user admin`

Если нужен прокси tor, раскомментируйте и исправьте IP в строке
    
	#SOCKSPort 192.168.0.1:9100 # Bind to this address:port too.

На

    SOCKSPort 192.168.1.1:9100 # Bind to this address:port too.
	
Теперь можете указывать socks proxy в браузере и программах 192.168.1.1:9100. Соединение пойдет через tor

**Внимание! Для анонимности этого не достаточно!**

*Начало не проверенного блока 1*

Однако есть ряд программ которые не умеют работать с socks протоколом. Эту проблему можно решить средствами самого tor + iptables. Для этого необходимо включить в tor режим прозрачного проксирования. В таком случае на подключенных к роутеру машинах вообще никаких настроек делать не нужно. Весь трафик будет сразу проходить через tor.
Использование прозрачного прокси в Tor

Для этого в конфигурационном файле /opt/etc/tor/torrc добавляем в конец:

	ExcludeExitNodes {RU} # Заблокировать выходные ноды из России
	AutomapHostsOnResolve 1
	TransPort 9040
	DNSPort 9053
	VirtualAddrNetwork 10.254.0.0/16  # виртуальные адреса для .onion ресурсов

Тем самым прозрачный прокси будет слушать порт 9040 (Но только для 127.0.0.1).

*Конец непровернного блока 1*

Настройка прошивки
------------------

Отредактируйте /opt/etc/init.d/S10iptables:

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

На самом деле файл может содержать другие команды которые трогать не нужно. Нужно добавить блок

	# Create new rublock ipset and fill it with IPs from list
	if [ ! -z "$(ipset --swap rublock rublock 2>&1 | grep 'given name does not exist')" ] ; then
			ipset -N rublock iphash
			for IP in $(cat /opt/etc/rublock.ips) ; do
					ipset -A rublock $IP
			done
	fi
	iptables -A PREROUTING -t mangle -m set --match-set rublock dst,src -j MARK --set-mark 1

*Начало не проверенного блока 2*	
	
На самом деле строку, которая просто помечает нужные пакеты
    
	iptables -A PREROUTING -t mangle -m set --match-set rublock dst,src -j MARK --set-mark 1

Нужно заменить на редирект на порт прокси tor. Пока не знаю как.

Для этого необходимо использовать iptables с опцией REDIRECT –to-port. Для использования этой опции необходимо дополнительно собрать и установить пакет iptables-mod-nat-extra. В противном случае вы получите ошибку вида:

    iptables v1.4.6: unknown option `--to-ports'
	
Не ясно есть ли такая опция в padavan ской прошивке. Но вроде пишут можно установить из репозитория.

Установить пакет можно тривиально выполнив

    opkg install kmod-ipt-nat-extra
	
После установки пакета необходимо “завернуть” трафик на прозрачный прокси сервер:

	iptables -t nat -A PREROUTING -p tcp --match-set rublock --dport 80 -j REDIRECT --to-ports 9040

Что означает – перебрасывать трафик хостов rublock идущий на 80-й порт на порт 9040, на котором как раз и находится прозрачный прокси.

Для перенаправления .onion ресурсов на прокси тора нужно добавить правило

	iptables -t nat -A OUTPUT -p tcp -d 10.254.0.0/16 -j REDIRECT --to-ports 9040
	
Сами .onion ресурсы должны разрешатся dns тора


*Конец непровернного блока 2*

В веб-интерфейсе роутера на странице Customization > Scripts отредактируйте поле Run After Router Started, раскоментировав две строчки:

    modprobe ip_set_hash_ip
    modprobe xt_set

На странице LAN > DHCP Server допишите в поле Custom Configuration File "dnsmasq.conf" строчку:

	conf-file=/opt/etc/rublock.dnsmasq

*Начало не проверенного блока 3*
	
Для разрешения имен .onion, туда же добавить строчку

	server=/onion/127.0.0.1#9053
	#ipset=/onion/onion # Непонятно нужно ли

*Конец непровернного блока 3*


Перегрузите роутер для того, чтобы настройки вступили в силу.

Полезные ссылки

http://rover-seti.blogspot.ru/2015/11/tor-openwrt.html

https://habrahabr.ru/post/270657/

https://geektimes.ru/post/129603/








