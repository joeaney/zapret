# ﻿zapret v.20

As a rule, DPI tricks do not help to bypass https blocking. What is it for?

-----------------
Bypass the blocking of web sites http.
How it works
----------------

DPI providers have gaps. They happen from what the DPI rules write for ordinary user programs, omitting all possible cases that are permissible by standards.
This is done for simplicity and speed. It makes no sense to catch hackers, of which 0.01%, because all the same, these locks are quite simple, even by ordinary users.

Some DPIs cannot recognize the http request if it is divided into TCP segments.  
For example, a query like `GET / HTTP/1.1\r\nHost: kinozal.tv......`  
we send in 2 parts: first comes `GET ", затем "/ HTTP/1.1\r\nHost: kinozal.tv.....`.  
Other DPI stumble when heading `Host:" spelled in another case: eg, "host:`.  
In some places, adding additional space after the method works: `GET /" => "GET  /` or adding a dot at the end of the host name: `Host: kinozal.tv.`

## How to put this into practice in the linux system

How to make the system break the request into parts? You can run the entire TCP session through transparent proxy, or you can replace the tcp window size field on the first incoming TCP packet with a SYN, ACK.

Then the client will think that the server has set a small window size for it and the first data segment will send no more than the specified length. In subsequent packages, we will not change anything.

The further behavior of the system at the choice of the size of the sent packets depends on the algorithm implemented in it. Experience shows that linux first package always sends no more than the length specified in the window size, the rest of the packets up to some time sends no more than max (36, specified_size).

After a certain number of packets, the window scaling mechanism is triggered and the scaling factor starts to be taken into account, the packet size becomes no more than max (36, specified_ramer << scale_factor).

Not very elegant behavior, but since we do not affect the size of the incoming packet, and the amount of data received via http is usually much higher than the amount sent, then only small delays will appear visually.

Windows behaves in a similar case much more predictably. The first segment goes the specified length, then the window size changes depending on the value sent in the new tcp packets. That is, the speed is almost immediately restored to a possible maximum.

To intercept a packet from SYN, ACK does not represent any complexity by means of iptables.  
However, the options for editing packages in iptables are severely limited.  
It’s just not possible to change window size with standard modules.  
For this, we will use the NFQUEUE tool. This tool allows packets to be processed by processes running in user mode.  
The process, accepting a package, can change it, which is what we need.

`iptables -t raw -I PREROUTING -p tcp --sport 80 --tcp-flags SYN,ACK SYN,ACK -j NFQUEUE --queue-num 200 --queue-bypass`

It will give the packages we need to the process that listens on the queue with the number 200.

It will replace the window size. PREROUTING will catch both packets addressed to the host itself and routed packets. That is, the solution works the same way on the client and on the router. On a PC-based or OpenWRT router.

In principle, this is enough.  
However, with such an impact on TCP there will be a slight delay.  
In order not to touch the hosts that are not blocked by the provider, you can make such a move.  
Create a list of blocked domains or download it from rublacklist.  
Secure all domains in ipv4 addresses. To drive them into the ipset with the name "zapret".  

Add to rule:

`iptables -t raw -I PREROUTING -p tcp --sport 80 --tcp-flags SYN,ACK SYN,ACK -m set --match-set zapret src -j NFQUEUE --queue-num 200 --queue-bypass`

Thus, the impact will be made only on ip addresses related to blocked sites.  
The list can be updated via cron every few days.  
If you update via rublacklist, then it will take quite a long time. More than an hour. But this process does not take away resources, so it will not cause any problems, especially if the system is constantly running.

If DPI doesn’t get by with splitting a request into segments, then sometimes a change occurs.
"Host:" on "host:". In this case, we may not need a window size replacement, so the chain PREROUTING we do not need. Instead, we hang on outgoing packets in the POSTROUTING chain:

`iptables -t mangle -I POSTROUTING -p tcp --dport 80 -m set --match-set zapret dst -j NFQUEUE --queue-num 200 --queue-bypass`

In this case, additional points are also possible. DPI can catch only the first http request, ignoring subsequent requests in the keep-alive session. Then we can reduce the load on the percent, abandoning the processing of unnecessary packages.

`iptables -t mangle -I POSTROUTING -p tcp --dport 80 -m connbytes --connbytes-dir=original --connbytes-mode=packets --connbytes 1:5 -m set --match-set zapret dst -j NFQUEUE --queue-num 200 --queue-bypass`

It happens that the provider monitors the entire HTTP session with keep-alive requests. In this case, it is not enough to restrict the TCP window when establishing a connection. You must send each new request with separate TCP segments. This task is solved through the full proxying of traffic through a transparent proxy (TPROXY or DNAT). TPROXY does not work with connections originating from the local system, so this solution is applicable only on the router. DNAT works with local connections, but there is a danger of entering into infinite recursion, so the daemon is started as a separate user, and DNAT is disabled for this user via "-m owner". Full proxying requires more processor resources than manipulating outgoing packets without reconstructing a TCP connection.

`
iptables -t nat -I PREROUTING -p tcp --dport 80 -j DNAT --to 127.0.0.1:1188
iptables -t nat -I OUTPUT -p tcp --dport 80 -m owner ! --uid-owner tpws -j DNAT --to 127.0.0.1:1188
`

## nfqws

This program is a packet modifier and a NFQUEUE queue handler.
It takes the following parameters:

`
 --daemon		: demonize prog
 --qnum=200		: queue number
 --wsize=4		: change tcp window size to specified size
 --hostcase		: change the case of the "Host:" header to "host:" by default.
 --hostnospace		: remove the space after the "Host:" and move it to the end of the value "User-Agent:" to save the packet length
 --hostspell=HoST	: the exact spelling of the Host header (you can use "HOST" or "HoSt"). automatically includes `--hostcase` The manipulation parameters can be combined in any combination.
`

## tpws

tpws - This is a transparent proxy.
`
 --bind-addr		; what address to listen to. maybe ipv4 or ipv6 address. if not specified, then listens on all ipv4 and ipv6 addresses
 --port=<port>		; which port to listen to
 --daemon               ; demonize prog
 --user=<username>	; change the uid of the process
 --split-http-req=method|host	; way to split http requests into segments: around the method (GET, POST) or around the Host header
 --split-pos=<offset>	; divide all messages into segments in the specified position. If sending is longer than 8Kb (receive buffer size), each block will be divided by 8Kb.
 --hostcase             ; change the case of the "Host:" header. default to "host:".
 --hostspell=HoST	; the exact spelling of the Host header (you can use "HOST" or "HoSt"). automatically includes `--hostcase`
 --hostdot		; Add a dot after the host name: `Host: kinozal.tv.`
 --hosttab		; add tab after hostname: `Host: kinozal.tv \ t`
 --hostnospace		; remove space after "Host:"
 --methodspace		; add space after method: "GET /" => "GET  /"
 --methodeol		; add line feed before method: "GET /" => "\r\nGET  /"
 --unixeol		; convert 0D0A to 0A and use everywhere 0A
 --hostlist=<filename>  ; act only on domains included in the list of filename. subdomains are automatically counted. The file must have a host on each line.
			; The list is read once at the start and is stored in the memory in the form of a hierarchical structure for a quick search.
			; for a list of RKN, a system with 128 Mb of memory may be required! calculate the RAM requirement for the process as 3-5 times the size of the list file.
			; on the HUP signal, the list will be re-read with the next accepted connection. The manipulation parameters can be combined in any combination.
There are exceptions: `split-pos` replaces `split-http-req`. hostdot and hosttab are mutually exclusive.
`

## Providers

Поскольку ситуация с блокировками на отдельных провайдерах может меняться, информация может устаревать. Она дана больше для примера, чем как прямое руководство.
Автор не занимается мониторингом и оперативным обновлением этой информации.

mns.ru : нужна замена window size на 3. mns.ru убирает заблокированные домены из выдачи своих DNS серверов. меняем на сторонние. аплинк westcall банит по IP адреса из списка РКН, где присутствует https

at-home.ru : при дефолтном подключении все блокировалось по IP. после заказа внешнего IP (static NAT) банятся по IP https адреса
 Для обхода DPI работает замена windows size на 3, но была замечена нестабильность и подвисания. Лучше всего работает сплит запроса около метода в течение всей http сессии.
 В https подменяется сертификат. Если у вас все блокируется по IP, то нет никакого способа, кроме как проксирование порта 80 по аналогии с 443.

beeline (corbina) : нужна замена регистра "Host:" на протяжении всей http сессии. С некоторых пор "host" не работает, но работают другие регистры букв.

dom.ru : нужно проксирование HTTP сессий через tpws с заменой регистра "Host:" и разделение TCP сегментов на хедере "Host:".
  Ахтунг ! Домру блокирует все поддомены заблоченого домена. IP адреса всевозможных поддоменов узнать невозможно из реестра
  блокировок, поэтому если вдруг на каком-то сайте вылезает блокировочный баннер, то идите в консоль firefox, вкладка network.
  Загружайте сайт и смотрите куда идет редирект. Потом вносите домен в zapret-hosts-user.txt. Например, на kinozal.tv имеются
  2 запрашиваемых поддомена : s.kinozal.tv и st.kinozal.tv с разными IP адресами.
  Домру перехватывает DNS запросы и всовывает свой лже-ответ. Это обходится через дроп лже-ответа посредством iptables по наличию IP адреса заглушки или через dnscrypt.

sknt.ru : проверена работа с tpws с параметром "--split-http-req=method". провайдер мониторит каждый пакет, поэтому при использовании nfq 2-й запрос в той же сессии зависает

Ростелеком/tkt : помогает разделение http запроса на сегменты, настройки mns.ru подходят
  ТКТ был куплен ростелекомом, используется фильтрация ростелекома.
  Поскольку DPI не отбрасывает входящую сессию, а только всовывает свой пакет, который приходит раньше ответа от настоящего сервера,
  блокировки так же обходятся без применения "тяжелой артиллерии" следующим правилом :
  iptables -t raw -I PREROUTING -p tcp --sport 80 -m string --hex-string "|0D0A|Location: http://warning.rt.ru" --algo bm -j DROP --from 40 --to 200

tiera : Требуется сплит http запросов в течение всей сессии.

Другие провайдеры
-----------------

Первым делом необходимо выяснить не подменят ли ваш провайдер DNS.
Посмотрите во что ресолвятся заблокированные хосты у вашего провайдера и через какой-нибудь web net tools, которых можно нагуглить множество. Сравните.
Если ответы разные, то попробуйте заресолвить те же хосты с DNS сервера 8.8.8.8 через вашего провайдера.
Если ответ от 8.8.8.8 нормальный - поменяйте DNS. Если ответ ненормальный, значит провайдер перехватывает запросы на сторонние DNS.
Используйте dnscrypt.

Далее необходимо выяснить какой метод обхода DPI работает на вашем провайдере.
В этом вам поможет скрипт https://github.com/ValdikSS/blockcheck.
Выберите какой демон вы будете использовать : nfqws или tpws.
Подготовьте вручную правила iptables для вашего случая, выполните их.
Запустите демон с нужными параметрами вручную.
Проверьте работает ли.
Когда вы найдете рабочий вариант, отредактируйте init скрипт для вашей системы.
Раскомментируйте ISP=custom. Добавьте ваш код в места "# PLACEHOLDER" по аналогии с секциями для других провайдеров для найденной рабочей комбинации.
Для openwrt поместите в /etc/firewall.user свой код по аналогии с готовыми скриптами.

Способы получения списка заблокированных IP
-------------------------------------------

1) Внесите заблокирванные домены в ipset/zapret-hosts-user.txt и запустите ipset/get_user.sh
На выходе получите ipset/zapret-ip-user.txt с IP адресами.

2) ipset/get_reestr.sh получает список доменов от rublacklist и дальше их ресолвит в ip адреса
в файл ipset/zapret-ip.txt. В этом списке есть готовые IP адреса, но судя во всему они там в точности в том виде,
что вносит в реестр РосКомПозор. Адреса могут меняться, позор не успевает их обновлять, а провайдеры редко
банят по IP : вместо этого они банят http запросы с "нехорошим" заголовком "Host:" вне зависимости
от IP адреса. Поэтому скрипт ресолвит все сам, хотя это и занимает много времени.
Дополнительное требование - объем памяти в /tmp для сохранения туда скачанного файла, размер которого
несколько Мб и продолжает расти. На роутерах openwrt /tmp представляет собой tmpfs , то есть ramdisk.
В случае роутера с 32 мб памяти ее не хватит, и будут проблемы. В этом случае используйте
следующий скрипт.

3) ipset/get_anizapret.sh. быстро и без нагрузки на роутер получает лист с antizapret.prostovpn.org.

4) ipset/get_combined.sh. для провайдеров, которые блокируют по IP https, а остальное по DPI. IP https заносится в ipset ipban, остальные в ipset zapret.
Поскольку скачивается большой список РКН, требования к месту в /tmp аналогичны 2)

Все варианты рассмотренных скриптов автоматически создают и заполняют ipset.
Варианты 2-4 дополнительно вызывают вариант 1.

На роутерах не рекомендуется вызывать эти скрипты чаще раза за 2 суток, поскольку сохранение идет
либо во внутреннюю флэш память роутера, либо в случае extroot - на флэшку.
В обоих случаях слишком частая запись может убить флэшку, но если это произойдет с внутренней
флэш памятью, то вы просто убьете роутер.

Принудительное обновление ipset выполняет скрипт ipset/create_ipset.sh.
Список РКН уже достиг внушительных размеров в сотни тысяч IP адресов. Поэтому для оптимизации ipset
применяется утилита ip2net. Она берет список отдельных IP адресов и пытается в нем найти подсети максимального размера (от /22 до /30),
в которых заблокировано более 3/4 адресов. ip2net написан на языке C, поскольку операция ресурсоемкая. Иные способы роутер может не потянуть.
Если ip2net скомпилирован или в каталог ip2net скопирован бинарик, то скрипт create_ipset.sh использует ipset типа hash:net, прогоняя список через ip2net.
В противном случае используется ipset типа hash:ip, список загружается как есть.
Соответственно, если вам не нравится ip2net, просто уберите из каталога ip2net бинарик.

Можно внести список доменов в ipset/zapret-hosts-user-ipban.txt. Их ip адреса будут помещены
в отдельный ipset "ipban". Он может использоваться для принудительного завертывания всех
соединений на прозрачный proxy "redsocks" или на VPN.

Фильтрация по именам доменов
----------------------------

Альтернативой ipset является использование tpws со списком доменов.
Список доменов РКН может быть получен скриптом ipset/get_hostlist.sh - кладется в ipset/zapret-hosts.txt.
Этот скрипт автоматически добавляет к списку РКН домены из zapret-hosts-user.txt.
tpws должен запускаться без фильтрации по ipset. Весь трафик http идет через tpws, и он решает нужно ли
применять дурение в зависимости от поля Host: в http запросе.
Это создает повышенную нагрузку на систему.
Сам поиск по доменам работает очень быстро, нагрузка связана с прокачиванием объема данных через процесс.
Вариант хорошо подходит для тех, у кого быстрая система с 128+ Мб памяти и провайдер применяет DPI.

Пример установки на debian 7
----------------------------
Debian 7 изначально содержит ядро 3.2. Оно не умеет делать DNAT на localhost.
Конечно, можно не привязывать tpws к 127.0.0.1 и заменить в правилах iptables "DNAT 127.0.0.1" на "REDIRECT",
но лучше установить более свежее ядро. Оно есть в стабильном репозитории :
 apt-get update
 apt-get install linux-image-3.16
Установить пакеты :
 apt-get update
 apt-get install libnetfilter-queue-dev ipset curl
Скопировать директорию "zapret" в /opt.
Собрать nfqws :
 cd /opt/zapret/nfq
 make
Собрать tpws :
 cd /opt/zapret/tpws
 make
Собрать ip2net :
 cd /opt/zapret/ip2net
 make
Скопировать /opt/zapret/init.d/debian7/zapret в /etc/init.d.
В /etc/init.d/zapret выбрать пераметр "ISP". В зависимости от него будут применены нужные правила.
Там же выбрать параметр SLAVE_ETH, соответствующий названию внутреннего сетевого интерфейса.
Включить автостарт : chkconfig zapret on
(опционально) Вручную первый раз получить новый список ip адресов : /opt/zapret/ipset/get_antizapret.sh
Зашедулить задание обновления листа :
 crontab -e
 Создать строчку  "0 12 * * */2 /opt/zapret/ipset/get_antizapret.sh". Это значит в 12:00 каждые 2 дня обновлять список.
Запустить службу : service zapret start
Попробовать зайти куда-нибудь : http://ej.ru, http://kinozal.tv, http://grani.ru.
Если не работает, то остановить службу zapret, добавить правило в iptables вручную,
запустить nfqws в терминале под рутом с нужными параметрами.
Пытаться подключаться к заблоченым сайтам, смотреть вывод программы.
Если нет никакой реакции, значит скорее всего указан неверный номер очереди или ip назначения нет в ipset.
Если реакция есть, но блокировка не обходится, значит параметры обхода подобраные неверно, или это средство
не работает в вашем случае на вашем провайдере.
Никто и не говорил, что это будет работать везде.
Попробуйте снять дамп в wireshark или "tcpdump -vvv -X host <ip>", посмотрите действительно ли первый
сегмент TCP уходит коротким и меняется ли регистр "Host:".

ubuntu 12,14
------------

Имеется готовый конфиг для upstart : zapret.conf. Его нужно скопировать в /etc/init и настроить по аналогии с debian.
Запуск службы : "start zapret"
Останов службы : "stop zapret"
Просмотр сообщений : cat /var/log/upstart/zapret.log
Ubuntu 12 так же, как и debian 7, оснащено ядром 3.2. См замечание в разделе "debian 7".

ubuntu 16,debian 8
------------------

Процесс аналогичен debian 7, однако требуется зарегистрировать init скрипты в systemd после их копирования в /etc/init.d.
По умолчанию lsb-core может быть не установлен.
apt-get update
apt-get --no-install-recommends install lsb-core

install : /usr/lib/lsb/install_initd zapret
remove : /usr/lib/lsb/remove_initd zapret
start : sytemctl start zapret
stop : systemctl stop zapret
status, output messages : systemctl status zapret

Другие linux системы
--------------------

Существует несколько основных систем запуска служб : sysvinit, upstart, systemd.
Настройка зависит от системы, используемой в вашем дистрибутиве.
Типичная стратегия - найти скрипт или конфигурацию запуска других служб и написать свой по аналогии,
при необходимости почитывая документацию по системе запуска.
Нужные команды можно взять из предложенных скриптов.


Фаерволлы
---------

Если вы используете какую-то систему управления фаерволом, то она может вступать в конфликт
с имеющимся скриптом запуска. В этом случае правила для iptables должны быть прикручены
к вашему фаерволу отдельно от скрипта запуска tpws или nfqws.
Именно так решается вопрос в случае с openwrt, поскольку там своя система управления фаерволом.
При повторном применении правил она могла бы поломать настройки iptables, сделанные скриптом из init.d.

Что делать с openwrt/LEDE
-------------------------

Установить дополнительные пакеты :
opkg update
opkg install iptables-mod-extra iptables-mod-nfqueue iptables-mod-filter iptables-mod-ipopt ipset curl bind-tools
(для новых LEDE) opkg install kmod-ipt-raw

Самая главная трудность - скомпилировать программы на C. Это можно сделать на linux x64 при помощи SDK, который
можно скачать с официального сайта openwrt или LEDE. Но процесс кросс компиляции - это всегда сложности.
Недостаточно запустить make как на традиционной linux системе.
Поэтому в binaries имеются готовые статические бинарики для всех самых распространенных архитектур.
Статическая сборка означает, что бинарик не зависит от типа libc (glibc, uclibc или musl) и наличия установленных so - его можно использовать сразу.
Лишь бы подходил тип CPU. У ARM и MIPS есть несколько версий. Найдите работающий на вашей системе вариант.
Скорее всего таковой найдется. Если нет - вам придется собирать самостоятельно.

Скопировать директорию "zapret" в /opt на роутер.
Скопировать работающий бинарик nfqws в /opt/zapret/nfq, tpws в /opt/zapret/tpws, ip2net в /opt/zapret/ip2net.
Скопировать /opt/zapret/init.d/zapret в /etc/init.d.
В /etc/init.d/zapret выбрать пераметр "ISP". В зависимости от него будут применены нужные правила.
/etc/init.d/zapret enable
/etc/init.d/zapret start
В зависимости от вашего провайдера внести нужные записи в /etc/firewall.user.
/etc/init.d/firewall restart
Посмотреть через iptables -L или через luci вкладку "firewall" появились ли нужные правила.
Зашедулить задание обновления листа :
 crontab -e
 Создать строчку  "0 12 * * */2 /opt/zapret/ipset/get_antizapret.sh". Это значит в 12:00 каждые 2 дня обновлять список.

Обход блокировки https
----------------------

Как правило трюки с DPI не помогают для обхода блокировки https.
Приходится перенаправлять трафик через сторонний хост.
Предлагается использовать прозрачный редирект через socks5 посредством iptables+redsocks, либо iptables+iproute+openvpn.
Настройка варианта с redsocks на openwrt описана в https.txt.
