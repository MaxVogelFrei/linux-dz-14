# Домашнее задание 14
Logs
## Задание
в вагранте поднимаем 2 машины web и log  
на web поднимаем nginx  
на log настраиваем центральный лог сервер на любой системе на выбор  
- journald  
- rsyslog  
- elk  
настраиваем аудит следящий за изменением конфигов нжинкса  

все критичные логи с web должны собираться и локально и удаленно  
все логи с nginx должны уходить на удаленный сервер (локально только критичные)  
логи аудита должны также уходить на удаленную систему  

## Решение
Настрою сбор логов на базе rsyslog и auditd  
На машине web дополнительно установлю epel, nginx и audispd-plugins для auditd  
### Конфигурация машины LOG
rsyslog.conf  
включу модули для получение логов с удаленных хостов, и добавлю темплейт и правило  
логи с других хостов попадут в папки с именами хостов и внутри буду собираться в файлы по названию приложения  
```bash
~
module(load="imudp")
module(load="imtcp" MaxSessions="500")
~
template(name="RemoteHost" type="string" string="/var/log/%HOSTNAME%/%PROGRAMNAME%.log")
~
ruleset(name="remote") {
    action(type="omfile" DynaFile="RemoteHost")
}

input(type="imudp" port="514" ruleset="remote")
input(type="imtcp" port="514" ruleset="remote")
```
auditd.conf  
раскоментирую строку с номером порта, и увеличу очередь на получение  
```bash
tcp_listen_port = 60
tcp_listen_queue = 50
```

### Конфигурация WEB
rsyslog.conf  
раскоментирую строки в forwarding rules  
на ip лог-сервера будут отправлены все логи с уровнем crit и выше
```bash
$ActionQueueFileName fwdRule1 # unique name prefix for spool files
$ActionQueueMaxDiskSpace 1g   # 1gb space limit (use as much as possible)
$ActionQueueSaveOnShutdown on # save messages to disk on shutdown
$ActionQueueType LinkedList   # run asynchronously
$ActionResumeRetryCount -1    # infinite retries if host is down
*.crit @@192.168.14.4:514
```
nginx.conf  
в локальный файл логов будут собраны только crit и выше, все остальные логи отправляются на хост 14.4  
```bash
error_log /var/log/nginx/error.log crit;
error_log syslog:server=192.168.14.4:514,facility=local7,tag=nginx,severity=info;
access_log syslog:server=192.168.14.4:514,facility=local7,tag=nginx,severity=info main;
```
audit.rules
добавляю правило для слежения за папкой /etc/nginx
```bash
-w /etc/nginx/ -p rwa -k nginx_config_access
```
audisp-remote.conf  
указываю адрес куда отправлять лог аудита  
```bash
remote_server = 192.168.14.4
```
au-remote.conf  
включаю плагин отправки на другой хост  
```bash
active = yes
```

### Результат
в логе auditd видно что приходит информация о доступе к файлу nginx.conf  
```bash
cat /var/log/audit/audit.log | grep nginx
node=web type=CONFIG_CHANGE msg=audit(1583335469.573:7): auid=4294967295 ses=4294967295 subj=system_u:system_r:unconfined_service_t:s0 op=add_rule key="nginx_config_access" list=4 res=1
node=web type=SYSCALL msg=audit(1583335569.481:64): arch=c000003e syscall=2 success=yes exit=3 a0=7ffc3d9e379d a1=0 a2=1fffffffffff0000 a3=7ffc3d9e2320 items=1 ppid=2762 pid=2780 auid=1000 uid=0 gid=0 euid=0 suid=0 fsuid=0 egid=0 sgid=0 fsgid=0 tty=pts0 ses=1 comm="cat" exe="/usr/bin/cat" subj=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023 key="nginx_config_access"
node=web type=PATH msg=audit(1583335569.481:64): item=0 name="/etc/nginx/nginx.conf" inode=33779033 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 obj=system_u:object_r:httpd_config_t:s0 objtype=NORMAL cap_fp=0000000000000000 cap_fi=0000000000000000 cap_fe=0 cap_fver=0
```
в файл /var/log/web/nginx.log попали логи с nginx  
```bash
cat nginx.log
Mar  4 15:28:14 web nginx: 192.168.14.4 - - [04/Mar/2020:15:28:14 +0000] "GET / HTTP/1.1" 200 4833 "-" "curl/7.29.0" "-"
Mar  4 15:28:17 web nginx: 2020/03/04 15:28:17 [error] 2817#0: *2 open() "/usr/share/nginx/html/111" failed (2: No such file or directory), client: 192.168.14.4, server: _, request: "GET /111 HTTP/1.1", host: "192.168.14.3"
Mar  4 15:28:17 web nginx: 192.168.14.4 - - [04/Mar/2020:15:28:17 +0000] "GET /111 HTTP/1.1" 404 3650 "-" "curl/7.29.0" "-"
```
При этом в логе на хосте web пусто, т.к. событий crit не случалось 
```bash
[root@web nginx]# ll
total 0
-rw-r--r--. 1 root root 0 Mar  4 15:28 error.log
```



