# Инициализация системы. Systemd.

## 1. Сервис мониторинга лога.

Создаем файл `watchlog` с переменными окружения для нашего сервиса в директории `/etc/sysconfig`

```
# Configuration file for my watchlog service
# Place it to /etc/sysconfig

# File and word in that file that we will be monit
WORD="OTUS"
LOG=/var/log/watchlog.log
```
Создаем файл `/var/log/watchlog.log`,содеожащий ключевое слово, в котором будут осуществлятся поиск.

Например:

```
Test HW8 string 1 OTUS
Test HW8 string 2
Test HW8 string 3
Test HW8 string 4
Test HW8 string 5 OTUS
Test HW8 string 6
Test HW8 string 7
Test HW8 string 8
Test HW8 string 9
Test HW8 string 10 OTUS
Test HW8 string 11
Test HW8 string 12
Test HW8 string 13
Test HW8 string 14
Test HW8 string 15 OTUS
```

Создаем скрипт `/opt/watchlog.sh`, который собственно и будет осуществлять поиск и отправлять лог в системный журнал

```
#!/bin/bash

WORD=$1
LOG=$2
DATE=`date`

if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word - OTUS!" #logger отправляет лог в системный журнал
else
exit 0
fi
```

Даем права на запуск файла

```
chmod +x /opt/watchlog.sh
```

Создаем Unit Сервис `/etc/systemd/system/watchlog.service`

```
[Unit]
Description=Watchlog service from HW8 OTUS
Wants=watchlog.timer

[Service]
EnvironmentFile=/etc/sysconfig/watchlog
Type=oneshot
ExecStart=/opt/watchlog.sh $WORD $LOG

[Install]
WantedBy=multi-user.target
```

Создадим Unit запуска сервиса каждые 30 секунд  `/etc/systemd/system/watchlog.timer`

```
[Unit]
Description=Run watchlog script every 30 second.WH8-OTUS
Requires=watchlog.service

[Timer]
# Run every 30 second
Unit=watchlog.service
OnUnitActiveSec=30

[Install]
WantedBy=multi-user.target
```
После запускаем timer и проверяем его работоспособнось

```
[root@localhost ~]# systemctl status watchlog.timer
● watchlog.timer - Run watchlog script every 30 second.WH8-OTUS
   Loaded: loaded (/etc/systemd/system/watchlog.timer; disabled; vendor preset: disabled)
   Active: active (running) since Вс 2023-02-05 08:37:42 MSK; 33s ago
```

Проверяем сатус service

```
[root@localhost ~]# systemctl status watchlog.service
● watchlog.service - Watchlog service from HW8 OTUS
   Loaded: loaded (/etc/systemd/system/watchlog.service; disabled; vendor preset: disabled)
   Active: inactive (dead) since Вс 2023-02-05 08:37:42 MSK; 6s ago
  Process: 5711 ExecStart=/opt/watchlog.sh $WORD $LOG (code=exited, status=0/SUCCESS)
 Main PID: 5711 (code=exited, status=0/SUCCESS)

фев 05 08:37:42 localhost.localdomain systemd[1]: Starting Watchlog service from HW8 OTUS...
фев 05 08:37:42 localhost.localdomain systemd[1]: Started Watchlog service from HW8 OTUS.
```

Смотрим результат

```
[root@localhost ~]# tail -f /var/log/messages
Feb  5 09:14:10 localhost systemd: Started Watchlog service from HW8 OTUS.
Feb  5 09:15:20 localhost systemd: Starting Watchlog service from HW8 OTUS...
Feb  5 09:15:20 localhost root: Вс фев  5 09:15:20 MSK 2023: I found word - OTUS!
Feb  5 09:15:20 localhost systemd: Started Watchlog service from HW8 OTUS.
Feb  5 09:16:19 localhost systemd: Starting Watchlog service from HW8 OTUS...
Feb  5 09:16:19 localhost systemd-logind: Removed session 4.
Feb  5 09:16:19 localhost root: Вс фев  5 09:16:19 MSK 2023: I found word - OTUS!
Feb  5 09:16:19 localhost systemd: Started Watchlog service from HW8 OTUS.
Feb  5 09:16:28 localhost systemd: Started Session 10 of user root.
Feb  5 09:16:28 localhost systemd-logind: New session 10 of user root.
Feb  5 09:17:10 localhost systemd: Starting Watchlog service from HW8 OTUS...
Feb  5 09:17:10 localhost root: Вс фев  5 09:17:10 MSK 2023: I found word - OTUS!
Feb  5 09:17:10 localhost systemd: Started Watchlog service from HW8 OTUS.
```

## 2. Переписать init-скрипт на unit-файл.

Ставим `spawn-fcgi` и необходимые пакеты

```
yum install -y epel-release

yum install -y spawn-fcgi

 yum -y install php
```

В файле `/etc/sysconfig/spawn-fcgi`  раскоментируем последние две строки

```
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u apache -g apache -s $SOCKET -S -M 0600 -C 32 -F 1 -P /var/run/spawn-fcgi.pid -- /usr/bin/php-cgi"
```
Создаем сам unit-файл `/etc/systemd/system/spawn-fcgi.service` со следующим содержимым

```
[Unit]
Description=Spawn-fcgi startup service by OTUS HW8  
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/sysconfig/spawn-fcgi
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```

Производим запуск и убеждаемся в том, что все работает

```
[root@localhost ~]# systemctl start spawn-fcgi
[root@localhost ~]# systemctl status spawn-fcgi
● spawn-fcgi.service - Spawn-fcgi startup service by OTUS HW8
   Loaded: loaded (/etc/systemd/system/spawn-fcgi.service; disabled; vendor preset: disabled)
   Active: active (running) since Вс 2023-02-05 13:20:15 MSK; 2h 53min ago
 Main PID: 1952 (php-cgi)
   CGroup: /system.slice/spawn-fcgi.service
           ├─1952 /usr/bin/php-cgi
           ├─1955 /usr/bin/php-cgi
           ├─1956 /usr/bin/php-cgi
           ├─1957 /usr/bin/php-cgi
           ├─1958 /usr/bin/php-cgi
           ├─1959 /usr/bin/php-cgi
           ├─1960 /usr/bin/php-cgi
           ├─1961 /usr/bin/php-cgi
           ├─1962 /usr/bin/php-cgi
           ├─1963 /usr/bin/php-cgi
           ├─1964 /usr/bin/php-cgi
           ├─1965 /usr/bin/php-cgi
           ├─1966 /usr/bin/php-cgi
           ├─1967 /usr/bin/php-cgi
           ├─1968 /usr/bin/php-cgi
           ├─1969 /usr/bin/php-cgi
           ├─1970 /usr/bin/php-cgi
           ├─1971 /usr/bin/php-cgi
           ├─1972 /usr/bin/php-cgi
           ├─1973 /usr/bin/php-cgi
           ├─1974 /usr/bin/php-cgi
           ├─1975 /usr/bin/php-cgi
           ├─1976 /usr/bin/php-cgi
           ├─1977 /usr/bin/php-cgi
           ├─1978 /usr/bin/php-cgi
           ├─1979 /usr/bin/php-cgi
           ├─1980 /usr/bin/php-cgi
           ├─1981 /usr/bin/php-cgi
           ├─1982 /usr/bin/php-cgi
           ├─1983 /usr/bin/php-cgi
           ├─1984 /usr/bin/php-cgi
           ├─1985 /usr/bin/php-cgi
           └─1986 /usr/bin/php-cgi

фев 05 13:20:15 localhost.localdomain systemd[1]: Started Spawn-fcgi startup service by OTUS HW8.
```

## 3. Дополнить unit-файл httpd (apache) возможностью запустить несколько инстансов сервера с разными конфигурационными файлами.

Ставим APACHE

```
yum -y install httpd
```

Текст исходного файла `/usr/lib/systemd/system/httpd.service` копируем в  Unit-файл сервиса для нескольких экземляров httpd `/etc/systemd/system/httpd@.service` с небольшими коррективами

```
[Unit]
Description=The Apache HTTP Server
After=network.target remote-fs.target nss-lookup.target
Documentation=man:httpd(8)
Documentation=man:apachectl(8)

[Service]
Type=notify
EnvironmentFile=/etc/sysconfig/httpd-%i  <-- здесь дописали необходимую конструкцию
ExecStart=/usr/sbin/httpd $OPTIONS -DFOREGROUND
ExecReload=/usr/sbin/httpd $OPTIONS -k graceful
ExecStop=/bin/kill -WINCH ${MAINPID}
# We want systemd to give httpd some time to finish gracefully, but still want
# it to kill httpd after TimeoutStopSec if something went wrong during the
# graceful stop. Normally, Systemd sends SIGTERM signal right after the
# ExecStop, which would kill httpd. We are sending useless SIGCONT here to give
# httpd time to finish.
KillSignal=SIGCONT
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

Создаем файлы (для каждого запускаемого сервиса свой)

`/etc/sysconfig/httpd-first`
```
OPTIONS=-f conf/first.conf
```

`/etc/sysconfig/httpd-second`

```
OPTIONS=-f conf/second.conf
```

Для каждого сервиса создаем свой config файл (за основу берем `/etc/httpd/conf/httpd.conf`)

Config `/etc/httpd/conf/first.conf` будет копией `/etc/httpd/conf/httpd.conf`


А config `/etc/httpd/conf/second.conf` будет отличаться от `/etc/httpd/conf/httpd.conf`
только двумя конструкциями

```
PidFile /var/run/httpd-second.pid

...

Listen 8080

...
```

Стартуем оба сервиса и проверяем их

```
[root@localhost ~]# systemctl start httpd@first
[root@localhost ~]# systemctl status httpd@first
● httpd@first.service - The Apache HTTP Server OTUS HW8
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Вс 2023-02-05 16:43:45 MSK; 3s ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 3678 (httpd)
   Status: "Processing requests..."
   CGroup: /system.slice/system-httpd.slice/httpd@first.service
           ├─3678 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3679 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3680 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3681 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           ├─3682 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND
           └─3683 /usr/sbin/httpd -f conf/first.conf -DFOREGROUND

фев 05 16:43:45 localhost.localdomain systemd[1]: Starting The Apache HTTP Server OTUS HW8...
фев 05 16:43:45 localhost.localdomain httpd[3678]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using localhost.localdomain. Set the 'ServerName' directive globally to suppress this message
фев 05 16:43:45 localhost.localdomain systemd[1]: Started The Apache HTTP Server OTUS HW8.
```

```
[root@localhost ~]# systemctl start httpd@second
[root@localhost ~]# systemctl status httpd@second
● httpd@second.service - The Apache HTTP Server OTUS HW8
   Loaded: loaded (/etc/systemd/system/httpd@.service; disabled; vendor preset: disabled)
   Active: active (running) since Вс 2023-02-05 14:15:25 MSK; 2h 30min ago
     Docs: man:httpd(8)
           man:apachectl(8)
 Main PID: 3073 (httpd)
   Status: "Total requests: 0; Current requests/sec: 0; Current traffic:   0 B/sec"
   CGroup: /system.slice/system-httpd.slice/httpd@second.service
           ├─3073 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3075 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3076 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3079 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           ├─3081 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND
           └─3082 /usr/sbin/httpd -f conf/second.conf -DFOREGROUND

фев 05 14:15:24 localhost.localdomain systemd[1]: Starting The Apache HTTP Server...
фев 05 14:15:25 localhost.localdomain httpd[3073]: AH00558: httpd: Could not reliably determine the server's fully qualified domain name, using localhost.localdomain. Set the 'ServerName' directive globally to suppress this message
фев 05 14:15:25 localhost.localdomain systemd[1]: Started The Apache HTTP Server.
```

Проверим прослушиваемые порты

```
[root@localhost ~]# ss -tnulp | grep httpd
tcp    LISTEN     0      128    [::]:80                 [::]:*                   users:(("httpd",pid=3683,fd=4),("httpd",pid=3682,fd=4),("httpd",pid=3681,fd=4),("httpd",pid=3680,fd=4),("httpd",pid=3679,fd=4),("httpd",pid=3678,fd=4))
tcp    LISTEN     0      128    [::]:8080               [::]:*                   users:(("httpd",pid=3082,fd=4),("httpd",pid=3081,fd=4),("httpd",pid=3079,fd=4),("httpd",pid=3076,fd=4),("httpd",pid=3075,fd=4),("httpd",pid=3073,fd=4))
```