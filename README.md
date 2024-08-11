### ДЗ ##

* Написать service, который будет раз в 30 секунд мониторить лог на предмет наличия ключевого слова (файл лога и ключевое слово должны задаваться в /etc/default).
* Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта (https://gist.github.com/cea2k/1318020).
* Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.

### **Выполнение** ###
Создаём файл с конфигурацией для сервиса в директории /etc/default из неё сервис будет брать необходимые переменные
команды выполняются из под root:
cd /etc/default
touch watchlog
nano watchlog

```
# Configuration file for my watchlog service`
# Place it to /etc/default`
# File and word in that file that we will be monit`
WORD="ALERT"`
LOG=/var/log/watchlog.log`
```

`Затем создаем /var/log/watchlog.log и пишем туда строки на своё усмотрение,
плюс ключевое слово ‘ALERT’`

Создадим скрипт
```
cd
cd /opt
nano watchlog.sh
#!/bin/bash
WORD=$1
LOG=$2
DATE=`date`
if grep $WORD $LOG &> /dev/null
then
logger "$DATE: I found word, Master!"
else
exit 0
fi 
```
`Команда logger отправляет лог в системный журнал.`

Добавим права на запуск файла:
```
chmod +x /opt/watchlog.sh
```

Создадим юнит для сервиса:
```
cd /etc/systemd
cd system/
nano watchlog.service
[Unit]
Description=My watchlog service

[Service]
Type=oneshot
EnvironmentFile=/etc/default/watchlog
ExecStart=/opt/watchlog.sh $WORD $LOG
```
Создадим юнит для таймера:
```
[Unit]
Description=Run watchlog script every 30 second

[Timer]
# Run every 30 second
OnUnitActiveSec=30
Unit=watchlog.service
[Install]
WantedBy=multi-user.target
```
Затем достаточно только запустить timer:
```
systemctl start watchlog.timer
```
результат проверяем: 
```
tail -n 1000 /var/log/syslog  | grep word
```
`Установить spawn-fcgi и создать unit-файл (spawn-fcgi.sevice) с помощью переделки init-скрипта
Устанавливаем spawn-fcgi и необходимые для него пакеты:`
```
apt install spawn-fcgi php php-cgi php-cli  apache2 libapache2-mod-fcgid -y
```
Необходимо создать файл с настройками для будущего сервиса в файле /etc/spawn-fcgi/fcgi.conf.
```
cd
cd /etc
mkdir spawn-fcgi
cd spawn-fcgi
nano fcgi.conf
Он должен получится следующего вида:
# You must set some working options before the "spawn-fcgi" service will work.
# If SOCKET points to a file, then this file is cleaned up by the init script.
#
# See spawn-fcgi(1) for all possible options.
#
# Example :
SOCKET=/var/run/php-fcgi.sock
OPTIONS="-u www-data -g www-data -s $SOCKET -S -M 0600 -C 32 -F 1 -- /usr/bin/php-cgi"

Сам юнит-файл будет примерно следующего вида:
[Unit]
Description=Spawn-fcgi startup service by Otus
After=network.target

[Service]
Type=simple
PIDFile=/var/run/spawn-fcgi.pid
EnvironmentFile=/etc/spawn-fcgi/fcgi.conf
ExecStart=/usr/bin/spawn-fcgi -n $OPTIONS
KillMode=process

[Install]
WantedBy=multi-user.target
```
Убеждаемся, что все успешно работает:
```
systemctl start spawn-fcgi
systemctl status spawn-fcgi
```
Доработать unit-файл Nginx (nginx.service) для запуска нескольких инстансов сервера с разными конфигурационными файлами одновременно.
Установим Nginx из стандартного репозитория:
```
apt install nginx -y
```
Для запуска нескольких экземпляров сервиса модифицируем исходный service для использования различной конфигурации, а также PID-файлов. Для этого создадим новый Unit для работы с шаблонами (/etc/systemd/system/nginx@.service)
```
cat >> /etc/systemd/system/nginx@.service
# Stop dance for nginx
# =======================
#
# ExecStop sends SIGSTOP (graceful stop) to the nginx process.
# If, after 5s (--retry QUIT/5) nginx is still running, systemd takes control
# and sends SIGTERM (fast shutdown) to the main process.
# After another 5s (TimeoutStopSec=5), and if nginx is alive, systemd sends
# SIGKILL to all the remaining processes in the process group (KillMode=mixed).
#
# nginx signals reference doc:
# http://nginx.org/en/docs/control.html
#
[Unit]
Description=A high performance web server and a reverse proxy server
Documentation=man:nginx(8)
After=network.target nss-lookup.target

[Service]
Type=forking
PIDFile=/run/nginx-%I.pid
ExecStartPre=/usr/sbin/nginx -t -c /etc/nginx/nginx-%I.conf -q -g 'daemon on; master_process on;'
ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;'
ExecReload=/usr/sbin/nginx -c /etc/nginx/nginx-%I.conf -g 'daemon on; master_process on;' -s reload
ExecStop=-/sbin/start-stop-daemon --quiet --stop --retry QUIT/5 --pidfile /run/nginx-%I.pid
TimeoutStopSec=5
KillMode=mixed

[Install]
WantedBy=multi-user.target
```
Необходимо создать два файла конфигурации (/etc/nginx/nginx-first.conf, /etc/nginx/nginx-second.conf). Их можно сформировать из стандартного конфига /etc/nginx/nginx.conf, 
с модификацией путей до PID-файлов и разделением по портам:
```
pid /run/nginx-first.pid;

http {
…
	server {
		listen 9001;
	}
#include /etc/nginx/sites-enabled/*;
….
}
```
Проверим работу:
```
systemctl start nginx@first
systemctl start nginx@second
systemctl status nginx@first
systemctl status nginx@second
```
Проверить можно несколькими способами, например, посмотреть, какие порты слушаются:
```
ss -tnulp | grep nginx
```
Или просмотреть список процессов:
```
ps afx | grep nginx
```
`Результаты выполнения ДЗ видны на приложенных скриншотах.`







