# DZ_SElinux


selinux включен:

[root@localhost ~]# getenforce
Enforcing



стандартные разрешенные порты:

[root@localhost ~]# semanage port -l | grep http

http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000



на установленной системе изменил порт nginx на 4881:

    server {
        listen       4881;
        listen       [::]:4881;
        server_name  _;
        root         /usr/share/nginx/html;


после этого nginx перестал работать на нестандартном порту:

sealert -a /var/log/audit/audit.log | grep nginx

type=AVC msg=audit(1761221944.746:248): avc:  denied  { name_bind } for  pid=5809 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0



firewall выключен:

[root@localhost ~]# systemctl status firewalld
○ firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; enabled; preset: enabled)
     Active: inactive (dead) since Thu 2025-10-23 08:40:40 EDT; 17s ago


конфигурация nginx корректная:

[root@localhost ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful



из лога выше берем время и смотрим причину блокировки:

grep 1761221944.746:248 /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1761221944.746:248): avc:  denied  { name_bind } for  pid=5809 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0

        Was caused by:
        The boolean nis_enabled was set incorrectly.
        Description:
        Allow nis to enabled

        Allow access by executing:
        # setsebool -P nis_enabled 1


1 СПОСОБ. включение параметра nis_enabled и перезапуск nginx.

[root@localhost nginx]# setsebool -P nis_enabled on
[root@localhost nginx]# systemctl reload nginx



проверка работоспособности nginx:

[root@localhost nginx]# ss -tnlp | grep nginx
LISTEN 0      511          0.0.0.0:4881      0.0.0.0:*    users:(("nginx",pid=5959,fd=6),("nginx",pid=5958,fd=6),("nginx",pid=5943,fd=6))
LISTEN 0      511             [::]:4881         [::]:*    users:(("nginx",pid=5959,fd=7),("nginx",pid=5958,fd=7),("nginx",pid=5943,fd=7))


[root@localhost nginx]# curl -I http://localhost:4881
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Thu, 23 Oct 2025 13:11:12 GMT
Content-Type: text/html
Content-Length: 5760
Last-Modified: Mon, 24 Mar 2025 16:15:24 GMT
Connection: keep-alive
ETag: "67e1851c-1680"
Accept-Ranges: bytes



отключение nis_enabled. как результат nginx перестал работать:

[root@localhost nginx]#  setsebool -P nis_enabled off
[root@localhost nginx]# curl -I http://localhost:4882
curl: (7) Failed to connect to localhost port 4882: Connection refused



2 СПОСОБ. добавления порта 4881 в уже имеющийся тип.


[root@localhost nginx]# semanage port -a -t http_port_t -p tcp 4881
[root@localhost nginx]# semanage port -l | grep http
http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
http_cache_port_t              udp      3130
http_port_t                    tcp      4881, 80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988
pegasus_https_port_t           tcp      5989



проверка работоспособности nginx:

[root@localhost nginx]# curl -I http://localhost:4881
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Thu, 23 Oct 2025 13:21:00 GMT
Content-Type: text/html
Content-Length: 5760
Last-Modified: Mon, 24 Mar 2025 16:15:24 GMT
Connection: keep-alive
ETag: "67e1851c-1680"
Accept-Ranges: bytes



3 СПОСОБ. формирование и установки модуля SELinux.

убрать порт из разрешенных:

[root@localhost nginx]# semanage port -d -t http_port_t -p tcp 4881
[root@localhost nginx]# semanage port -l | grep  http_port_t
http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000
pegasus_http_port_t            tcp      5988



nginx не работает:

[root@localhost nginx]# curl -I http://localhost:4881
curl: (7) Failed to connect to localhost port 4881: Connection refused
[root@localhost nginx]# systemctl start nginx
Job for nginx.service failed because the control process exited with error code.
See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.



формирования модуля который позволит запустить nginx на порту 4881:

[root@localhost nginx]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx
******************** IMPORTANT ***********************
To make this policy package active, execute:

semodule -i nginx.pp

[root@localhost nginx]#  semodule -i nginx.pp



запуск nginx:

[root@localhost nginx]#  systemctl start nginx



проверка работоспособности nginx:

[root@localhost nginx]#  systemctl status nginx
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; preset: disabled)
     Active: active (running) since Thu 2025-10-23 09:32:45 EDT; 10s ago
    Process: 6065 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
    Process: 6066 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
    Process: 6067 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
   Main PID: 6068 (nginx)
      Tasks: 3 (limit: 11072)
     Memory: 2.9M
        CPU: 45ms
     CGroup: /system.slice/nginx.service
             ├─6068 "nginx: master process /usr/sbin/nginx"
             ├─6069 "nginx: worker process"
             └─6070 "nginx: worker process"


[root@localhost nginx]# curl -I http://localhost:4881
HTTP/1.1 200 OK
Server: nginx/1.20.1
Date: Thu, 23 Oct 2025 13:33:04 GMT
Content-Type: text/html
Content-Length: 5760
Last-Modified: Mon, 24 Mar 2025 16:15:24 GMT
Connection: keep-alive
ETag: "67e1851c-1680"
Accept-Ranges: bytes
