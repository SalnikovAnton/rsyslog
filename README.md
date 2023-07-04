# Стенд с Vagrant c Rsyslog
### 1) В Vagrant разворачиваем 2 виртуальные машины web и log
### 2) на web настраиваем nginx
### 3) на log настраиваем центральный лог сервер на rsyslog
### 4) настраиваем аудит, следящий за изменением конфигов nginx 
   
#### 1)Создаём виртуальные машины используя  [Vagrantfile](https://github.com/SalnikovAnton/rsyslog/blob/main/Vagrantfile "Vagrantfile")  разворачиваются 2 виртуальные машины web и log  
#### !!! в Vagrantfile добавлен пункт для автоматизации с помощью Ansible. Дополнительно потребуется скопировать файлы playbook.yml и nginx.conf в директорию с Vagrantfile.
```
~/rsyslog$ vagrant status
Current machine states:

web                       running (virtualbox)
log                       running (virtualbox)
```
Для правильной работы c логами, нужно, чтобы на всех хостах было настроено одинаковое время.
```
[root@web ~]# sudo timedatectl set-timezone Europe/Moscow
[root@log ~]# systemctl restart chronyd
[root@web ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2023-06-29 20:11:48 MSK; 1min 9s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 3265 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 3260 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 3263 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─3263 /usr/sbin/chronyd
***
[root@web ~]# date
Thu Jun 29 20:13:38 MSK 2023
```
и на второй машине
```
[root@log ~]# sudo timedatectl set-timezone Europe/Moscow
[root@log ~]# systemctl restart chronyd
[root@log ~]# systemctl status chronyd
● chronyd.service - NTP client/server
   Loaded: loaded (/usr/lib/systemd/system/chronyd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2023-06-29 20:14:22 MSK; 2s ago
     Docs: man:chronyd(8)
           man:chrony.conf(5)
  Process: 3255 ExecStartPost=/usr/libexec/chrony-helper update-daemon (code=exited, status=0/SUCCESS)
  Process: 3251 ExecStart=/usr/sbin/chronyd $OPTIONS (code=exited, status=0/SUCCESS)
 Main PID: 3253 (chronyd)
   CGroup: /system.slice/chronyd.service
           └─3253 /usr/sbin/chronyd
***
[root@log ~]# date
Thu Jun 29 20:14:44 MSK 2023
```
#### 2) Устанавливаем nginx
```
[root@web ~]# yum install epel-release
***
Installed:
  epel-release.noarch 0:7-11                                                                         

Complete!

[root@web ~]# yum install -y nginx
***
Installed:
  nginx.x86_64 1:1.20.1-10.el7                                                                       

Dependency Installed:
  centos-indexhtml.noarch 0:7-9.el7.centos         centos-logos.noarch 0:70.0.6-3.el7.centos        
  gperftools-libs.x86_64 0:2.6.1-1.el7             nginx-filesystem.noarch 1:1.20.1-10.el7          
  openssl11-libs.x86_64 1:1.1.1k-5.el7            

Complete!

[root@web ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-06-29 20:36:23 MSK; 2s ago
  Process: 22149 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22146 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22145 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22151 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22151 nginx: master process /usr/sbin/nginx
           └─22153 nginx: worker process
***
```
Так же работу nginx проверяем на хосте. В браузере введем в адерсную строку http://192.168.56.10   
![Image alt](https://github.com/SalnikovAnton/rsyslog/blob/main/nginx.png)   

#### 3) Настройка центрального сервера сбора логов   
Для сбора логов будем использовать rsyslog, он должен быть установлен по умолчанию в нашей ОС
```
[root@log ~]# yum list rsyslog
Failed to set locale, defaulting to C
Loaded plugins: fastestmirror
Determining fastest mirrors
 * base: mirror.sale-dedic.com
 * extras: mirrors.datahouse.ru
 * updates: centosmirror.netcup.net
base                                                                          | 3.6 kB  00:00:00     
extras                                                                        | 2.9 kB  00:00:00     
updates                                                                       | 2.9 kB  00:00:00     
(1/4): extras/7/x86_64/primary_db                                             | 249 kB  00:00:00     
(2/4): base/7/x86_64/group_gz                                                 | 153 kB  00:00:00     
(3/4): base/7/x86_64/primary_db                                               | 6.1 MB  00:00:02     
(4/4): updates/7/x86_64/primary_db                                            |  22 MB  00:00:16     
Installed Packages
rsyslog.x86_64                              8.24.0-52.el7                                   @anaconda
Available Packages
rsyslog.x86_64                              8.24.0-57.el7_9.3                               updates  
``` 
Для того, чтобы наш сервер мог принимать логи, нам необходимо изминить следующий файл "/etc/rsyslog.conf
" с настройками Rsyslog и превести его в следующий вид с открытыми портами 514 (TCP и UDP) [rsyslog.conf](https://github.com/SalnikovAnton/rsyslog/blob/main/rsyslog.conf "rsyslog.conf")   
Если ошибок не допущено, то у нас будут видны открытые порты TCP,UDP 514:
```
[root@log ~]# ss -tuln
Netid State      Recv-Q Send-Q   Local Address:Port                  Peer Address:Port              
udp   UNCONN     0      0                    *:111                              *:*                  
udp   UNCONN     0      0                    *:928                              *:*                  
udp   UNCONN     0      0                    *:514                              *:*                  
udp   UNCONN     0      0            127.0.0.1:323                              *:*                  
udp   UNCONN     0      0                    *:68                               *:*                  
udp   UNCONN     0      0                 [::]:111                           [::]:*                  
udp   UNCONN     0      0                 [::]:928                           [::]:*                  
udp   UNCONN     0      0                 [::]:514                           [::]:*                  
udp   UNCONN     0      0                [::1]:323                           [::]:*                  
tcp   LISTEN     0      128                  *:22                               *:*                  
tcp   LISTEN     0      100          127.0.0.1:25                               *:*                  
tcp   LISTEN     0      25                   *:514                              *:*                  
tcp   LISTEN     0      128                  *:111                              *:*                  
tcp   LISTEN     0      128               [::]:22                            [::]:*                  
tcp   LISTEN     0      100              [::1]:25                            [::]:*                  
tcp   LISTEN     0      25                [::]:514                           [::]:*                  
tcp   LISTEN     0      128               [::]:111                           [::]:*   
``` 
Теперь настроим отправку логов с web-сервера. Находим в файле /etc/nginx/nginx.conf раздел с логами и приводим их к следующему виду [nginx.conf](https://github.com/SalnikovAnton/rsyslog/blob/main/nginx.conf "nginx.conf")       
Проверяем файл командой и перезапускаем nginx
```
[root@web ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
[root@web ~]# vi /etc/nginx/nginx.conf
[root@web ~]# systemctl restart nginx
[root@web ~]# systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Thu 2023-06-29 21:30:17 MSK; 3min 31s ago
  Process: 22208 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 22205 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 22204 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 22210 (nginx)
   CGroup: /system.slice/nginx.service
           ├─22210 nginx: master process /usr/sbin/nginx
           └─22212 nginx: worker process
***
``` 
Для проверки отправки логов удалим картинку веб-сраницы: rm /usr/share/nginx/html/img/header-background.png и несколько раз зайдем по адресу "http://192.168.50.10". На log-сервере смотрим логи
```
[root@log web]# cat /var/log/rsyslog/web/nginx_access.log
Jun 29 22:57:12 web nginx_access: 192.168.56.1 - - [29/Jun/2023:22:57:12 +0300] "GET / HTTP/1.1" 200 4833 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/111.0"
Jun 29 22:57:12 web nginx_access: 192.168.56.1 - - [29/Jun/2023:22:57:12 +0300] "GET /favicon.ico HTTP/1.1" 404 3650 "http://192.168.56.10/" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/111.0"
Jun 29 22:57:13 web nginx_access: 192.168.56.1 - - [29/Jun/2023:22:57:13 +0300] "GET / HTTP/1.1" 200 4833 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/111.0"
Jun 29 22:57:14 web nginx_access: 192.168.56.1 - - [29/Jun/2023:22:57:14 +0300] "GET / HTTP/1.1" 304 0 "-" "Mozilla/5.0 (X11; Ubuntu; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/111.0"

[root@log web]# cat /var/log/rsyslog/web/nginx_error.log 
Jun 29 21:39:48 web nginx_error: 2023/06/29 21:39:48 [error] 22212#22212: *1 "/usr/share/nginx/html/repo/index.html" is not found (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /repo/ HTTP/1.1", host: "192.168.56.10"
Jun 29 21:39:48 web nginx_error: 2023/06/29 21:39:48 [error] 22212#22212: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /favicon.ico HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/repo/"
Jun 29 22:57:12 web nginx_error: 2023/06/29 22:57:12 [error] 22335#22335: *1 open() "/usr/share/nginx/html/favicon.ico" failed (2: No such file or directory), client: 192.168.56.1, server: _, request: "GET /favicon.ico HTTP/1.1", host: "192.168.56.10", referrer: "http://192.168.56.10/"

``` 
логи отправляются корректно

#### 4) Настройка аудита, контролирующего изменения конфигурации nginx
За аудит отвечает утилита audit, проверяем её наличие
```
[root@web ~]# rpm -qa | grep audit
audit-2.8.5-4.el7.x86_64
audit-libs-2.8.5-4.el7.x86_64
``` 
Добавим правило, которое будет отслеживать изменения в конфигруации nginx. Для этого отредактируем файл /etc/audit/rules.d/[audit.rules](https://github.com/SalnikovAnton/rsyslog/blob/main/audit.rules "audit.rules")   
Перезапускаем службу auditd
```
[root@web ~]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
``` 
После данных изменений у нас начнут локально записываться логи аудита. Чтобы проверить, что логи аудита начали записываться локально, нужно внести изменения в файл "/etc/nginx/nginx.conf" или поменять его атрибут, потом посмотреть информацию об изменениях: 
```
[root@web nginx]# ausearch -f /etc/nginx/nginx.conf
----
time->Thu Jun 29 22:56:20 2023
type=CONFIG_CHANGE msg=audit(1688068580.164:1017): auid=1000 ses=4 op=updated_rules path="/etc/nginx/nginx.conf" key="nginx_conf" list=4 res=1
----
time->Thu Jun 29 22:56:20 2023
type=CONFIG_CHANGE msg=audit(1688068580.164:1019): auid=1000 ses=4 op=updated_rules path="/etc/nginx/nginx.conf" key="nginx_conf" list=4 res=1
``` 
Также можно воспользоваться поиском по файлу "/var/log/audit/audit.log", указав наш тэг:
```
[root@web nginx]# grep nginx_conf /var/log/audit/audit.log
type=CONFIG_CHANGE msg=audit(1688066946.894:1014): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1688066946.894:1015): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_conf" list=4 res=1
type=CONFIG_CHANGE msg=audit(1688068580.164:1017): auid=1000 ses=4 op=updated_rules path="/etc/nginx/nginx.conf" key="nginx_conf" list=4 res=1
type=SYSCALL msg=audit(1688068580.164:1018): arch=c000003e syscall=82 success=yes exit=0 a0=14b09d0 a1=14be2a0 a2=fffffffffffffe80 a3=7ffc628ff660 items=4 ppid=3161 pid=22315 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=4 comm="vi" exe="/usr/bin/vi" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_conf"
***
``` 
Далее настроим пересылку логов на удаленный сервер. Auditd по умолчанию не умеет пересылать логи, для пересылки на web-сервере потребуется установить пакет audispd-plugins:
```
[root@web nginx]# yum -y install audispd-plugins
***
Installed:
  audispd-plugins.x86_64 0:2.8.5-4.el7                                                               

Complete!
``` 
Правим файл /etc/audit/[auditd.conf](https://github.com/SalnikovAnton/rsyslog/blob/main/auditd.conf "auditd.conf") и /etc/audisp/plugins.d/[au-remote.conf](https://github.com/SalnikovAnton/rsyslog/blob/main/au-remote.conf "au-remote.conf") и /etc/audisp/audisp-remote.conf [audisp-remote.conf](https://github.com/SalnikovAnton/rsyslog/blob/main/audisp-remote.conf "audisp-remote.conf")   
Перезапускаем службу auditd
```
[root@web nginx]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
``` 
На этом настройка web-сервера завершена. Далее настроим Log-сервер.   
Отроем порт TCP 60, для этого уберем значки комментария в файле /etc/audit/auditd.conf:
```
tcp_listen_port = 60
``` 
Перезапустим службу auditd
```
[root@log web]# service auditd restart
Stopping logging:                                          [  OK  ]
Redirecting start to /bin/systemctl start auditd.service
``` 
На этом настройка пересылки логов аудита закончена   
Выполним проверку   

![Image alt](https://github.com/SalnikovAnton/rsyslog/blob/main/test_1.png)   
![Image alt](https://github.com/SalnikovAnton/rsyslog/blob/main/test_2.png)   










