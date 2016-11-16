Использование tor для обхода блокировок
=======================================

Решение ниже позволяет для роутера на прошивке Padavan использовать tor исключтельно для списка блокируемых ресурсов. При обращении к любым другим ресурсам будет использоваться привычное провайдерское соединение.

Описание
--------

При работе получается следующее. Если требуемый ресурс отсутствует в списке блокируемых, то используется провайдерское соединение и провайдерские dns. Если же требуемый ресурс есть в списке, тогда используется dns от google и tor для открытия ресурса. Все работает прозрачно, без настроек на клиентской стороне.
Так же прозрачно открывается домен onion

Ограничения
-----------

* Список заблокированных ресурсов обновляется только при рестарте роутера. Если заблокированный ресурс поменяет IP, он перестанет открываться до рестарта роутера.

* Вопрос возможности использования tor с процессором роутера остается открытым.

* Схема не работает при подключении к роутеру по VPN при заворачивании всего траффика через него. Т.к. пакеты от VPN видимо не проходят PREROUTING.

* Firefox почему-то не открывает onion, хотя IE и chrome открывают без проблем

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

На самом деле перестройка файла rublock.ips бессмысленна без рестарта /opt/etc/init.d/S10iptables. Который на основе него делает ipset. А стартует он только по перезагрузке. Так что при изменении ip ресурса нужно будет перезагружать роутер. Так же dnsmasq не обновляет список без рестарта. Или я не так понял (что скорее всего)

Установка tor
-------------

    opkg install tor
	
Для настройки конфигурации правим конфиг /opt/etc/tor/torrc

Исправляем в конце `user tor` на `user admin`

Для настройки прозрачного прокси в Tor в конфигурационном файле /opt/etc/tor/torrc добавляем в конец:

	ExcludeExitNodes {RU} # Заблокировать выходные ноды из России
	AutomapHostsOnResolve 1
	TransPort 9040
	DNSPort 9053
	VirtualAddrNetwork 10.254.0.0/16  # виртуальные адреса для .onion ресурсов

Тем самым прозрачный прокси будет слушать порт 9040, dns тора будет висеть на порту 9053 (Но только для 127.0.0.1).

Если нужен прокси tor, раскомментируйте и исправьте IP в строке (необязательно, схема будет работать и без этого)
    
	#SOCKSPort 192.168.0.1:9100 # Bind to this address:port too.

На

    SOCKSPort 192.168.1.1:9100 # Bind to this address:port too.
	
Теперь можете указывать socks proxy в браузере и программах 192.168.1.1:9100. Соединение пойдет через tor

**Внимание! Для анонимности этого не достаточно!**


Настройка прошивки
------------------

* Отредактируйте /opt/etc/init.d/S10iptables:


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
			# rublock redirect to tor
            iptables -t nat -A PREROUTING -p tcp --dport 80 -m set --match-set rublock dst,src -j REDIRECT --to-ports 9040
            # .onion redirect to tor
            iptables -t nat -A PREROUTING -p tcp -d 10.254.0.0/16 -j REDIRECT --to-ports 9040
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

Файл может содержать другие команды которые трогать не нужно. Нужно добавить блок

	# Create new rublock ipset and fill it with IPs from list
	if [ ! -z "$(ipset --swap rublock rublock 2>&1 | grep 'given name does not exist')" ] ; then
			ipset -N rublock iphash
			for IP in $(cat /opt/etc/rublock.ips) ; do
					ipset -A rublock $IP
			done
	fi
	# rublock redirect to tor
	iptables -t nat -A PREROUTING -p tcp --dport 80 -m set --match-set rublock dst,src -j REDIRECT --to-ports 9040
	# .onion redirect to tor
	iptables -t nat -A PREROUTING -p tcp -d 10.254.0.0/16 -j REDIRECT --to-ports 9040

Вариант перенаправления всех портов заблокированных ресурсов в тор прокси, поставил, нужно пробовать
	
	iptables -t nat -A PREROUTING -p tcp -m set --match-set rublock dst -j REDIRECT --to-ports 9040
	
OUTPUT не работает
	
В веб-интерфейсе роутера на странице Customization > Scripts отредактируйте поле Run After Router Started, раскоментировав две строчки:

    modprobe ip_set_hash_ip
    modprobe xt_set

* На странице LAN > DHCP Server допишите в поле Пользовательский файл конфигурации "dnsmasq.conf" строчку:


	conf-file=/opt/etc/rublock.dnsmasq

* На этой же странице, допишите в поле Пользовательский файл конфигурации "dnsmasq.servers" строчку:
	

	server=/onion/127.0.0.1#9053
	
Добавляет сервер dns для разрешения имен домена .onion. Будут возвращаться виртуальные адреса из указанной в настройке tor подсети.

* На странице Дополнительно - Администрирование - Сервисы - Сервис Cron (планировщик)? включить. Добавить строчку:

	1 3 * * 0 blupdate.lua
	
Будет происходить обновление списка ресурсов каждое воскресенье в 3 часа ночи. Но изменения подействуют только после перезагрузки.

Лучше конечно при старте бы обновлять, но не знаю как.

Перегрузите роутер для того, чтобы настройки вступили в силу.

Полезные ссылки
---------------

По мотивам https://github.com/DontBeAPadavan/rublock-via-vpn/

http://rover-seti.blogspot.ru/2015/11/tor-openwrt.html

https://habrahabr.ru/post/270657/

https://geektimes.ru/post/129603/

https://trac.torproject.org/projects/tor/wiki/doc/TransparentProxy#TransparentlyDoingDNSandRoutingfor.onionTraffic

[Описание iptables] (http://www.k-max.name/linux/netfilter-iptables-v-linux/)








