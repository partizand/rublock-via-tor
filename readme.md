Использование tor для обхода блокировок
=======================================

Пока в процессе настройки. Работать не будет!!!

Решение ниже позволяет использовать tor исключтельно для списка блокируемых ресурсов. При обращении к любым другим ресурсам будет использоваться привычное провайдерское соединение.

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

Установка tor
-------------

    opkg install tor
	
Для настройки конфигурации правим конфиг /opt/etc/tor/torrc

Исправляем в конце `user tor` на `user admin`



