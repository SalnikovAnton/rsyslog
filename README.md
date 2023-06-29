# Docker
### 1) В Vagrant разворачиваем 2 виртуальные машины web и log
### 2) на web настраиваем nginx
### 3) на log настраиваем центральный лог сервер на rsyslog
### 4) настраиваем аудит, следящий за изменением конфигов nginx 
   
#### 1)Создаём виртуальные машины используя  [Vagrantfile](https://github.com/SalnikovAnton/rsyslog/blob/main/Vagrantfile "Vagrantfile") разворачиваются 2 виртуальные машины web и log  
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
Также работу nginx проверяем на хосте. В браузере введем в адерсную строку http://192.168.56.10 
СКРИН ТУТ

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
" с настройками Rsyslog и превести его в следующий вид с открытыми портами 514 (TCP и UDP)










