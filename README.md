# rpm
Чтобы получать пакеты и обновления, вам необходимо выполнить следующие команды:
```
cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
yum update -y
```
Устанавливаем пакеты:

```
[root@rpms vagrant]# yum install -y \
redhat-lsb-core \ 
wget \
rpmdevtools \
> rpm-build \
> createrepo \
> yum-utils \
> cmake \
> gcc \
> git
Failed to set locale, defaulting to C.UTF-8
Complete!
```
Последняя надпись указывает, что не хватает языкового пакета, не критично, но доставим пакеты:
```
dnf install langpacks-en glibc-all-langpacks -y
```

Возьмем для примера NGINX и соберем его с поддержкой openssl

Загрузим SRPM пакет NGINX для дальнейшей работы над ним:

```
[root@rpms vagrant]# wget https://nginx.org/packages/centos/8/SRPMS/nginx-1.20.2-1.el8.ngx.src.rpm
--2024-04-08 16:50:34--  https://nginx.org/packages/centos/8/SRPMS/nginx-1.20.2-1.el8.ngx.src.rpm
Resolving nginx.org (nginx.org)... 3.125.197.172, 52.58.199.22, 2a05:d014:5c0:2600::6, ...
Connecting to nginx.org (nginx.org)|3.125.197.172|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1086865 (1.0M) [application/x-redhat-package-manager]
Saving to: 'nginx-1.20.2-1.el8.ngx.src.rpm'

nginx-1.20.2-1.el8.ngx.src.rpm                              100%[========================================================================================================================================>]   1.04M  3.20MB/s    in 0.3s

2024-04-08 16:50:34 (3.20 MB/s) - 'nginx-1.20.2-1.el8.ngx.src.rpm' saved [1086865/1086865]
```
При установке такого пакета в домашней директории создается древо каталогов для
сборки:
[root@rpms ~]# rpm -i  nginx-1.20.2-1.el8.ngx.src.rpm

```
 
Также нужно скачать и разархивировать последний исходник для openssl - он
потребуется при сборке:
```
[root@rpms vagrant]# wget https://www.openssl.org/source/openssl-1.1.1w.tar.gz
```
[root@rpms vagrant]# wget https://www.openssl.org/source/openssl-1.1.1w.tar.gz
--2024-04-08 22:27:49--  https://www.openssl.org/source/openssl-1.1.1w.tar.gz
Распознаётся www.openssl.org (www.openssl.org)… 34.36.58.177, 2600:1901:0:1812::
Подключение к www.openssl.org (www.openssl.org)|34.36.58.177|:443... соединение установлено.
HTTP-запрос отправлен. Ожидание ответа… 200 OK
Длина: 9893384 (9,4M) [application/x-tar]
Сохранение в: «openssl-1.1.1w.tar.gz»

openssl-1.1.1w.tar.gz                                       100%[========================================================================================================================================>]   9,43M  8,72MB/s    за 1,1s

2024-04-08 22:27:51 (8,72 MB/s) - «openssl-1.1.1w.tar.gz» сохранён [9893384/9893384]

[root@rpms vagrant]# tar -xvf openssl-1.1.1w.tar.gz
openssl-1.1.1w/
openssl-1.1.1w/ACKNOWLEDGEMENTS
openssl-1.1.1w/AUTHORS
openssl-1.1.1w/CHANGES
openssl-1.1.1w/CONTRIBUTING
openssl-1.1.1w/Configurations/
```
поставим все зависимости, чтобы в процессе сборки не было ошибок:
```
[root@rpms ngx_brotli]# yum-builddep /root/rpmbuild/SPECS/nginx.spec

Последняя проверка окончания срока действия метаданных: 0:00:11 назад, Вт 09 апр 2024 18:26:15.
Пакет openssl-devel-1:1.1.1k-12.el8.x86_64 уже установлен.
Пакет pcre-devel-8.42-6.el8.x86_64 уже установлен.
Пакет systemd-239-82.el8.x86_64 уже установлен.
Пакет zlib-devel-1.2.11-25.el8.x86_64 уже установлен.
Зависимости разрешены.
Отсутствуют действия для выполнения
Выполнено!
```
поправить сам spec файл, чтобы NGINX:
```
vi /root/rpmbuild/SPECS/nginx.spec
путь до openssl указываем ДО каталога:

--with-openssl=/root/openssl-1.1.1w    

```
Также убираем лишние зависимости в данном дистрибутиве:
Должно быть так в измененном файле:

Source0: http://nginx.org/download/%{name}-%{version}.tar.gz
Source1: logrotate
Source2: nginx.conf
Source3: nginx.default.conf
Source4: nginx.service
Source5: nginx.upgrade.sh
Source6: nginx.suse.logrotate
Source7: nginx-debug.service
Source8: nginx.copyright
Source9: nginx.check-reload.sh
```

```
приступаем к сборке RPM пакета:
```
rpmbuild -bb /root/rpmbuild/SPECS/nginx.spec

Применилось после второго раза:
Проверяем
```
[root@rpms ~]# ll rpmbuild/RPMS/x86_64/
итого 3148
-rw-r--r--. 1 root root  839064 апр  9 22:54 nginx-1.20.2-1.el8.ngx.x86_64.rpm
-rw-r--r--. 1 root root 2381132 апр  9 22:54 nginx-debuginfo-1.20.2-1.el8.ngx.x86_64.rpm
```
Теперь можно установить наш пакет и убедиться, что nginx работает:
```
[root@rpms ~]# yum localinstall -y rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm
Последняя проверка окончания срока действия метаданных: 0:29:18 назад, Вт 09 апр 2024 22:35:57.
Зависимости разрешены.
=============================================================================================================================================================================================================================================
 Пакет                                               Архитектура                                          Версия                                                            Репозиторий                                                Размер
=============================================================================================================================================================================================================================================
Установка:
 nginx                                               x86_64                                               1:1.20.2-1.el8.ngx                                                @commandline                                               819 k

Результат транзакции
=============================================================================================================================================================================================================================================
Установка  1 Пакет

Общий размер: 819 k
Объем изменений: 2.8 M
Загрузка пакетов:
Проверка транзакции
Проверка транзакции успешно завершена.
Идет проверка транзакции
Тест транзакции проведен успешно
Выполнение транзакции
  Подготовка       :                                                                                                                                                                                                                     1/1
  Запуск скриптлета: nginx-1:1.20.2-1.el8.ngx.x86_64                                                                                                                                                                                     1/1
  Установка        : nginx-1:1.20.2-1.el8.ngx.x86_64                                                                                                                                                                                     1/1
  Запуск скриптлета: nginx-1:1.20.2-1.el8.ngx.x86_64                                                                                                                                                                                     1/1
----------------------------------------------------------------------

Thanks for using nginx!

Please find the official documentation for nginx here:
* https://nginx.org/en/docs/

Please subscribe to nginx-announce mailing list to get
the most important news about nginx:
* https://nginx.org/en/support.html

Commercial subscriptions for nginx are available on:
* https://nginx.com/products/

----------------------------------------------------------------------

  Проверка         : nginx-1:1.20.2-1.el8.ngx.x86_64                                                                                                                                                                                     1/1

Установлен:
  nginx-1:1.20.2-1.el8.ngx.x86_64

Выполнено!
```
[root@rpms ~]# systemctl start nginx

```
```
[root@rpms ~]# systemctl status nginx
● nginx.service - nginx - high performance web server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Tue 2024-04-09 23:06:23 UTC; 44s ago
     Docs: http://nginx.org/en/docs/
  Process: 64959 ExecStart=/usr/sbin/nginx -c /etc/nginx/nginx.conf (code=exited, status=0/SUCCESS)
 Main PID: 64960 (nginx)
    Tasks: 2 (limit: 49502)
   Memory: 2.0M
   CGroup: /system.slice/nginx.service
           ├─64960 nginx: master process /usr/sbin/nginx -c /etc/nginx/nginx.conf
           └─64961 nginx: worker process

апр 09 23:06:23 rpms systemd[1]: Starting nginx - high performance web server...
апр 09 23:06:23 rpms systemd[1]: nginx.service: Can't open PID file /var/run/nginx.pid (yet?) after start: No such file or directory
апр 09 23:06:23 rpms systemd[1]: Started nginx - high performance web server.
```
```
Создать свой репозиторий и разместить там ранее собранный RPM

```
Создадим каталог:
```
```
[root@rpms ~]# mkdir /usr/share/nginx/html/repo
[root@rpms ~]# ll /usr/share/nginx/html
итого 8
-rw-r--r--. 1 root root 494 апр  9 22:54 50x.html
-rw-r--r--. 1 root root 612 апр  9 22:54 index.html
drwxr-xr-x. 2 root root   6 апр  9 23:09 repo
```

Копируем туда наш собранный RPM

```
[root@rpms ~]# cp rpmbuild/RPMS/x86_64/nginx-1.20.2-1.el8.ngx.x86_64.rpm /usr/share/nginx/html/repo/
```
```
[root@rpms ~]# wget https://downloads.percona.com/downloads/percona-distribution-mysql-ps/percona-distribution-mysql-ps-8.0.28/binary/redhat/8/x86_64/percona-orchestrator-3.2.6-2.el8.x86_64.rpm -O /usr/share/nginx/html/repo/percona-orchestrator-3.2.6-2.el8.x86_64.rpm
--2024-04-09 23:16:11--  https://downloads.percona.com/downloads/percona-distribution-mysql-ps/percona-distribution-mysql-ps-8.0.28/binary/redhat/8/x86_64/percona-orchestrator-3.2.6-2.el8.x86_64.rpm
Распознаётся downloads.percona.com (downloads.percona.com)… 49.12.125.205, 2a01:4f8:242:5792::2
Подключение к downloads.percona.com (downloads.percona.com)|49.12.125.205|:443... соединение установлено.
HTTP-запрос отправлен. Ожидание ответа… 200 OK
Длина: 5222976 (5,0M) [application/x-redhat-package-manager]
Сохранение в: «/usr/share/nginx/html/repo/percona-orchestrator-3.2.6-2.el8.x86_64.rpm»

/usr/share/nginx/html/repo/percona-orchestrator-3.2.6-2.el8 100%[========================================================================================================================================>]   4,98M  4,92MB/s    за 1,0s

2024-04-09 23:16:12 (4,92 MB/s) - «/usr/share/nginx/html/repo/percona-orchestrator-3.2.6-2.el8.x86_64.rpm» сохранён [5222976/5222976]

```
Инициализируем репозиторий командой:
```
[root@rpms ~]# createrepo /usr/share/nginx/html/repo/
Directory walk started
Directory walk done - 2 packages
Temporary output repo path: /usr/share/nginx/html/repo/.repodata/
Preparing sqlite DBs
Pool started (with 5 workers)
Pool finished
```
```
Для прозрачности настроим в NGINX доступ к листингу каталога:
location / {
root /usr/share/nginx/html;
index index.html index.htm;
autoindex on; < Добавили эту директиву
}
```
Проверяем синтаксис и перезапускаем NGINX:
```
[root@rpms ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
```
[root@rpms ~]# nginx -s reload
```
```
[root@rpms ~]# curl -a http://localhost/repo/
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          09-Apr-2024 23:18                   -
<a href="nginx-1.20.2-1.el8.ngx.x86_64.rpm">nginx-1.20.2-1.el8.ngx.x86_64.rpm</a>                  09-Apr-2024 23:12              839064
<a href="percona-orchestrator-3.2.6-2.el8.x86_64.rpm">percona-orchestrator-3.2.6-2.el8.x86_64.rpm</a>        16-Feb-2022 15:57             5222976
</pre><hr></body>
</html>
```

Добавим его в /etc/yum.repos.d:

```
[root@rpms ~]# cat >> /etc/yum.repos.d/otus.repo << EOF
> [otus]
> name=otus-linux
> baseurl=http://localhost/repo
> gpgcheck=0
> enabled=1
> EOF
```
Убедимся, что репозиторий подключился и посмотрим, что в нем есть:
```
[root@rpms ~]# yum repolist enabled | grep otus
otus                                    otus-linux
```
```
[root@rpms ~]# yum list | grep otus
otus-linux                                      261 kB/s | 2.8 kB     00:00
percona-orchestrator.x86_64                            2:3.2.6-2.el8                                          otus
```
установим репозиторий percona-release:
```
[root@rpms ~]# yum install percona-orchestrator.x86_64 -y
Последняя проверка окончания срока действия метаданных: 0:01:27 назад, Вт 09 апр 2024 23:33:45.
Зависимости разрешены.
=============================================================================================================================================================================================================================================
 Пакет                                                            Архитектура                                        Версия                                                      Репозиторий                                           Размер
=============================================================================================================================================================================================================================================
Установка:
 percona-orchestrator                                             x86_64                                             2:3.2.6-2.el8                                               otus                                                  5.0 M
Установка зависимостей:
 jq                                                               x86_64                                             1.5-12.el8                                                  appstream                                             161 k
 oniguruma                                                        x86_64                                             6.8.2-2.el8                                                 appstream                                             187 k

Результат транзакции
=============================================================================================================================================================================================================================================
Установка  3 Пакета

Объем загрузки: 5.3 M
Объем изменений: 17 M
Загрузка пакетов:
(1/3): percona-orchestrator-3.2.6-2.el8.x86_64.rpm                                                                                                                                                            93 MB/s | 5.0 MB     00:00
(2/3): oniguruma-6.8.2-2.el8.x86_64.rpm                                                                                                                                                                      780 kB/s | 187 kB     00:00
(3/3): jq-1.5-12.el8.x86_64.rpm                                                                                                                                                                              580 kB/s | 161 kB     00:00
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
Общий размер                                                                                                                                                                                                  19 MB/s | 5.3 MB     00:00
Проверка транзакции
Проверка транзакции успешно завершена.
Идет проверка транзакции
Тест транзакции проведен успешно
Выполнение транзакции
  Подготовка       :                                                                                                                                                                                                                     1/1
  Установка        : oniguruma-6.8.2-2.el8.x86_64                                                                                                                                                                                        1/3
  Запуск скриптлета: oniguruma-6.8.2-2.el8.x86_64                                                                                                                                                                                        1/3
  Установка        : jq-1.5-12.el8.x86_64                                                                                                                                                                                                2/3
  Установка        : percona-orchestrator-2:3.2.6-2.el8.x86_64                                                                                                                                                                           3/3
  Запуск скриптлета: percona-orchestrator-2:3.2.6-2.el8.x86_64                                                                                                                                                                           3/3
  Проверка         : jq-1.5-12.el8.x86_64                                                                                                                                                                                                1/3
  Проверка         : oniguruma-6.8.2-2.el8.x86_64                                                                                                                                                                                        2/3
  Проверка         : percona-orchestrator-2:3.2.6-2.el8.x86_64                                                                                                                                                                           3/3

Установлен:
  jq-1.5-12.el8.x86_64                                                 oniguruma-6.8.2-2.el8.x86_64                                                 percona-orchestrator-2:3.2.6-2.el8.x86_64

Выполнено!
```

