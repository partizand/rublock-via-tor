Использование tor для обхода блокировок
=======================================

Решение для роутера, позволяет прозрачно использовать tor исключтельно для списка блокируемых ресурсов. При обращении к любым другим ресурсам будет использоваться привычное провайдерское соединение.

Описание
--------

При работе получается следующее. Если требуемый ресурс отсутствует в списке блокируемых, то используется провайдерское соединение и провайдерские dns. Если же требуемый ресурс есть в списке, тогда используется dns от google и tor для открытия ресурса. Все работает прозрачно, без настроек на клиентской стороне.
Так же прозрачно открывается домен onion

Ограничения
-----------

* Список заблокированных ресурсов обновляется только при рестарте роутера. Если заблокированный ресурс поменяет IP, он перестанет открываться до рестарта роутера.

* Вопрос возможности использования tor с процессором роутера остается открытым. Вроде как грузит проц около 25%. Работать можно.

* Firefox почему-то не открывает onion, хотя IE и Chrome открывают без проблем

Требования
----------

* Прошивка [Padavan] (https://bitbucket.org/padavan/rt-n56u/overview) с ipset (любые сборки, кроме nano),
* [Развёрнутый] (https://bitbucket.org/padavan/rt-n56u/wiki/RU/%D0%98%D1%81%D0%BF%D0%BE%D0%BB%D1%8C%D0%B7%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5%20Entware) репозиторий Entware

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

Обработка dns запросов имен заблокированных сайтов через провайдера/google/tordns включается отключается в начале скрипта /opt/bin/blupdate.lua. По умолчанию поставил через провайдера, т.к. dnsmasq Будет проще
	
Запустите скрипт на исполнение:

    blupdate.lua

В результате работы скрипта будут сформированы файлы /opt/etc/rublock.dnsmasq и /opt/etc/rublock.ips, которые будут использованы далее. Скрипт добавляется в cron (см. ниже).

На самом деле перестройка файла rublock.ips бессмысленна без рестарта /opt/etc/init.d/S10iptables. Который на основе него делает ipset. А стартует он только по перезагрузке. Так что при изменении ip ресурса нужно будет перезагружать роутер. Так же dnsmasq не обновляет список без рестарта. Или я не так понял (что скорее всего)

Установка tor
-------------

    opkg install tor
	opkg install tor-geoip
	
Для настройки конфигурации правим конфиг /opt/etc/tor/torrc

Исправляем в конце `user tor` на `user admin`

Для настройки прозрачного прокси в Tor в конфигурационном файле /opt/etc/tor/torrc добавляем в конец:

	AutomapHostsOnResolve 1
	TransPort 127.0.0.1:9040 # Прозрачный прокси
	#TransPort 10.8.0.1:9040 # Нужно только для работы через VPN
	DNSPort 9053
	ExcludeExitNodes {RU}
	VirtualAddrNetwork 10.254.0.0/16  # виртуальные адреса для .onion ресурсов

Тем самым прозрачный прокси будет слушать порт 9040, dns тора будет висеть на порту 9053 (Но только для 127.0.0.1).

Готовый файл конфигурации можно скачать из репозитория

	wget --no-check-certificate -O /opt/etc/tor/torrc https://raw.githubusercontent.com/partizand/rublock-via-tor/master/opt/etc/tor/torrc


Если нужна работа для клиентов VPN сервера, расскоментируйте строку, указав в ней адрес VPN сервера роутера (можно найти [здесь] (http://my.router/vpnsrv.asp#cfg) "Локальный IP-адрес VPN-сервера")

	#TransPort 10.8.0.1:9040 # Нужно только для работы через VPN

Если нужен прокси tor, раскомментируйте и исправьте IP в строке (необязательно, схема будет работать и без этого)
    
	#SOCKSPort 192.168.0.1:9100 # Bind to this address:port too.

На

    SOCKSPort 192.168.1.1:9100 # Bind to this address:port too.
	
Теперь можете указывать socks proxy в браузере и программах 192.168.1.1:9100. Соединение пойдет через tor

**Внимание! Для анонимности этого не достаточно!**


Настройка прошивки
------------------

В веб-интерфейсе роутера на странице [Персонализация > Скрипты] (http://my.router/Advanced_Scripts_Content.asp)
добавит в "Выполнить после перезапуска правил брандмауэра":

	```
	# rublock redirect to tor
	iptables -t nat -I PREROUTING -p tcp -m set --match-set rublock dst -j REDIRECT --to-ports 9040
	# .onion redirect to tor
	iptables -t nat -I PREROUTING -p tcp -m set --match-set onion dst -j REDIRECT --to-ports 9040
	```


* Там же отредактируйте поле "Выполнить после полного запуска маршрутизатора:", раскоментировав две строчки и добавив:

    ```
	modprobe ip_set_hash_ip
    modprobe xt_set
	
	ipset -N onion iphash
	
	# Create new rublock ipset and fill it with IPs from list
	if [ ! -z "$(ipset --swap rublock rublock 2>&1 | grep 'given name does not exist')" ] ; then
			ipset -N rublock iphash
			for IP in $(cat /opt/etc/rublock.ips) ; do
					ipset -A rublock $IP
			done
	fi
	```

* На странице [LAN > DHCP-сервер] (http://my.router/Advanced_DHCP_Content.asp) допишите в поле "Пользовательский файл конфигурации dnsmasq.conf" строчку:

	```
	server=/onion/127.0.0.1#9053
	ipset=/onion/onion

	conf-file=/opt/etc/rublock.dnsmasq
	```

* На странице [Администрирование - Сервисы] (http://my.router/Advanced_Services_Content.asp) включите "Сервис Cron (планировщик)?".  Добавьте строчку:

	`1 3 * * 0 blupdate.lua`
	
	Будет происходить обновление списка ресурсов каждое воскресенье в 3 часа ночи. Но изменения подействуют только после перезагрузки.

	Лучше конечно при старте бы обновлять, но не знаю как.

* Перегрузите роутер для того, чтобы настройки вступили в силу.

Полезные ссылки
---------------

По мотивам https://github.com/DontBeAPadavan/rublock-via-vpn/

Тоже самое, поздно увидел http://forum.ixbt.com/topic.cgi?id=14:63015:3387#3387

http://rover-seti.blogspot.ru/2015/11/tor-openwrt.html

https://habrahabr.ru/post/270657/

https://geektimes.ru/post/129603/

https://trac.torproject.org/projects/tor/wiki/doc/TransparentProxy#TransparentlyDoingDNSandRoutingfor.onionTraffic

[Описание iptables] (http://www.k-max.name/linux/netfilter-iptables-v-linux/)








