### OTUS-Linux-2023-04-L17 | Selinux

####Запуск nginx на нестандартном порту разными способами

1. Первый способ запустить nginx на нестандартном порту - разрежить параметр nis_enabled:
	
	*[root@SeLinux ~]# grep 1692533212.734:893 /var/log/audit/audit.log | audit2why*

	type=AVC msg=audit(1692533212.734:893): avc:  denied  { name_bind } for  pid=3166 comm="nginx" src=4881 scontext=system_u:system_r:httpd_t:s0 tcontext=system_u:object_r:unreserved_port_t:s0 tclass=tcp_socket permissive=0
	Was caused by:
 	The boolean nis_enabled was set incorrectly. 
 	Description:
 	Allow nis to enabled

 	*[root@SeLinux ~]# **setsebool -P nis_enabled on***

 	Проверяем:

 	*[root@SeLinux ~]# systemctl status nginx*

	b''' nginx.service - The nginx HTTP and reverse proxy server
   	Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   	Active: **active (running)** since Sun 2023-08-20 12:17:10 UTC; 6s ago


2. Второй способ - добавление нужного порта в разрешённые для соответствующего типа:

	*[root@SeLinux ~]# **semanage port -a -t http_port_t -p tcp 4881***
	
	*[root@SeLinux ~]# semanage port -l | grep http*

	http_cache_port_t              tcp      8080, 8118, 8123, 10001-10010
	http_cache_port_t              udp      3130
	**http_port_t**                    **tcp      4881**, 80, 81, 443, 488, 8008, 8009, 8443, 9000
	pegasus_http_port_t            tcp      5988
	pegasus_https_port_t           tcp      5989

	Проеряем:

	*[root@SeLinux ~]# systemctl status nginx*

	b nginx.service - The nginx HTTP and reverse proxy server
   	Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   	Active: **active (running)** since Sun 2023-08-20 12:43:27 UTC; 6s ago`

3. Третий способ - формирование и установка модуля для Selinux:
	
	*[root@SeLinux ~]# grep nginx /var/log/audit/audit.log | audit2allow -M nginx*

	******************** IMPORTANT ***********************
	To make this policy package active, execute:

	semodule -i nginx.pp
	
	*[root@SeLinux ~]# **semodule -i nginx.pp***
	*[root@SeLinux ~]# systemctl status nginx*
	
	b nginx.service - The nginx HTTP and reverse proxy server
   	Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   	**Active: active (running)** since Sun 2023-08-20 13:07:17 UTC; 1s ago

####Обеспечение работоспособности приложения при включенном SELinux

Стэнд из двух хостов: client и ns01 (сервер)

При попытке внести изменения в зону наблюдается проблема на серверной части стэнда, ошибка в контексте безопасности. Вместо типа named_t используется тип etc_t:
	*[root@ns01 ~]# cat /var/log/audit/audit.log | audit2why*

	type=AVC msg=audit(1692541771.016:1924): avc:  denied  { create } for  pid=5144 comm="isc-worker0000" name="named.ddns.lab.view1.jnl" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
        Missing type enforcement (TE) allow rule.
        You can use audit2allow to generate a loadable module to allow this access.
    
    *[root@ns01 ~]# ls -laZ /etc/named*

	drw-rwx---. root named system_u:object_r:**etc_t**:s0       .
	drwxr-xr-x. root root  system_u:object_r:**etc_t**:s0       ..
	drw-rwx---. root named unconfined_u:object_r:**etc_t**:s0   dynamic
	-rw-rw----. root named system_u:object_r:**etc_t**:s0       named.50.168.192.rev
	-rw-rw----. root named system_u:object_r:**etc_t**:s0       named.dns.lab
	-rw-rw----. root named system_u:object_r:**etc_t**:s0       named.dns.lab.view1
	-rw-rw----. root named system_u:object_r:**etc_t**:s0       named.newdns.lab

	Контекст безопасности неправильный, конфигурационные файлы лежат в другом каталоге.	

	Смотрим где должны лежать конфигурационные файлы:
	*[root@ns01 ~]# semanage fcontext -l | grep named*

	/etc/rndc.*                                        regular file       system_u:object_r:named_conf_t:s0 
	**/var/named**(/.*)?                                   all files          system_u:object_r:**named_zone_t**:s0

	Изменяем тип контекста безопасности каталога и проверяем:
	
	*[root@ns01 ~]# chcon -R -t named_zone_t /etc/named*
	*[root@ns01 ~]# ls -laZ /etc/named*
	
	drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
	drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
	drw-rwx---. root named unconfined_u:object_r:named_zone_t:s0 dynamic
	-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.50.168.192.rev
	-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
	-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.view1
	rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab

	Пробуем внести изменения в зону на клиенте ещё раз:
	*[root@client ~]# nsupdate -k /etc/named.zonetransfer.key*

	server 192.168.50.10
	zone ddns.lab
	update add www.ddns.lab. 60 A 192.168.50.15
	send
	quit
	
	Изменения внеслись успешно.

	Проверяем после перезагрузки клиента и сервера:
	
	*[root@client ~]# dig www.ddns.lab*
	
	; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.14 <<>> www.ddns.lab
	;; global options: +cmd
	;; Got answer:
	;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 43462
	;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2

	;; OPT PSEUDOSECTION:
	; EDNS: version: 0, flags:; udp: 4096
	;; QUESTION SECTION:
	;www.ddns.lab.                  IN      A

	;; ANSWER SECTION:
	www.ddns.lab.           60      IN      A       192.168.50.15

	;; AUTHORITY SECTION:
	ddns.lab.               3600    IN      NS      ns01.dns.lab.
	
	;; ADDITIONAL SECTION:
	ns01.dns.lab.           3600    IN      A       192.168.50.10
	
	;; Query time: 1 msec
	;; SERVER: 192.168.50.10#53(192.168.50.10)
	;; WHEN: Sun Aug 20 15:19:48 UTC 2023
	;; MSG SIZE  rcvd: 96


