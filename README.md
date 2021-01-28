**SELinux**

1. Установить nginx

`yum install epel-release`

`yum install nginx`

`systemctl status nginx.service`

2. Установить утилиты для работы с selinux

`yum install setools-console policycoreutils-python-utils setroubleshoot-server -y`

3. Проверить статус selinux

`sestatus`

```
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Memory protection checking:     actual (secure)
Max kernel policy version:      31
```

4. Изменить стандартный порт nginx

`vi /etc/nginx/nginx.conf`

```
http {
    server {
        listen       8088 default_server;
        listen       [::]:8088 default_server;
	}
}
```

5. Запустить сервис nginx

`systemctl start nginx.service`

6. Проверить статус сервиса nginx

`systemctl status nginx.service`

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Sun 2021-01-24 08:49:01 UTC; 12s ago
  Process: 256874 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=1/FAILURE)
  Process: 256872 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)

Jan 24 08:49:01 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 24 08:49:01 selinux nginx[256874]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 24 08:49:01 selinux nginx[256874]: nginx: [emerg] bind() to 0.0.0.0:8088 failed (13: Permission denied)
Jan 24 08:49:01 selinux nginx[256874]: nginx: configuration file /etc/nginx/nginx.conf test failed
Jan 24 08:49:01 selinux systemd[1]: nginx.service: Control process exited, code=exited status=1
Jan 24 08:49:01 selinux systemd[1]: nginx.service: Failed with result 'exit-code'.
Jan 24 08:49:01 selinux systemd[1]: Failed to start The nginx HTTP and reverse proxy server.
```

7. Запустить nginx на нестандартном порту 3-мя разными способами:

Запустить утилиту для траблшутинга (парсит audit.log и даёт рекомендации)

`sealert -a /var/log/audit/audit.log`

7.1 Способ 1. Использовать boolean

`setsebool -P nis_enabled 1`

`systemctl start nginx.service`

`systemctl status nginx.service`

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-01-24 12:04:31 UTC; 3s ago
  Process: 27742 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27741 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27739 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27744 (nginx)
    Tasks: 3 (limit: 17868)
   Memory: 5.2M
   CGroup: /system.slice/nginx.service
           ├─27744 nginx: master process /usr/sbin/nginx
           ├─27745 nginx: worker process
           └─27746 nginx: worker process

Jan 24 12:04:31 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 24 12:04:31 selinux nginx[27741]: nginx: the configuration file /etc/nginx/nginx.conf syntax>
Jan 24 12:04:31 selinux nginx[27741]: nginx: configuration file /etc/nginx/nginx.conf test is su>
Jan 24 12:04:31 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

`ss -tulpn`

`tcp       LISTEN     0          128                     [::]:8088                  [::]:*         users:(("nginx",pid=27746,fd=9),("nginx",pid=27745,fd=9),("nginx",pid=27744,fd=9))`

```
# Отключить boolean
# setsebool -P nis_enabled 0
```

7.2 Способ 2. Добавить порт в список разрешенных

`semanage port -l | grep http_port`

`http_port_t                    tcp      80, 81, 443, 488, 8008, 8009, 8443, 9000`

`semanage port -a -t http_port_t -p tcp 8088`

`systemctl start nginx.service`

`systemctl status nginx.service`

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-01-24 11:32:11 UTC; 2min 30s ago
  Process: 15573 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 15571 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 15569 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 15574 (nginx)
    Tasks: 3 (limit: 17868)
   Memory: 5.3M
   CGroup: /system.slice/nginx.service
           ├─15574 nginx: master process /usr/sbin/nginx
           ├─15575 nginx: worker process
           └─15576 nginx: worker process

Jan 24 11:32:11 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 24 11:32:11 selinux nginx[15571]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 24 11:32:11 selinux nginx[15571]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 24 11:32:11 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

`ss -tulpn`

`tcp       LISTEN     0          128                     [::]:8088                  [::]:*         users:(("nginx",pid=15576,fd=9),("nginx",pid=15575,fd=9),("nginx",pid=15574,fd=9))`

```
# Удалить порт
# semanage port -d -t http_port_t -p tcp 8088
```

7.3 Способ 3. Сформировать и установить модуль

`ausearch -c 'nginx' --raw | audit2allow -M my-nginx`

`semodule -X 300 -i my-nginx.pp`

`semodule -l | grep my-nginx`

`systemctl start nginx.service`

`systemctl status nginx.service`

```
● nginx.service - The nginx HTTP and reverse proxy server
   Loaded: loaded (/usr/lib/systemd/system/nginx.service; disabled; vendor preset: disabled)
   Active: active (running) since Sun 2021-01-24 12:04:31 UTC; 26min ago
  Process: 27742 ExecStart=/usr/sbin/nginx (code=exited, status=0/SUCCESS)
  Process: 27741 ExecStartPre=/usr/sbin/nginx -t (code=exited, status=0/SUCCESS)
  Process: 27739 ExecStartPre=/usr/bin/rm -f /run/nginx.pid (code=exited, status=0/SUCCESS)
 Main PID: 27744 (nginx)
    Tasks: 3 (limit: 17868)
   Memory: 5.2M
   CGroup: /system.slice/nginx.service
           ├─27744 nginx: master process /usr/sbin/nginx
           ├─27745 nginx: worker process
           └─27746 nginx: worker process

Jan 24 12:04:31 selinux systemd[1]: Starting The nginx HTTP and reverse proxy server...
Jan 24 12:04:31 selinux nginx[27741]: nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
Jan 24 12:04:31 selinux nginx[27741]: nginx: configuration file /etc/nginx/nginx.conf test is successful
Jan 24 12:04:31 selinux systemd[1]: Started The nginx HTTP and reverse proxy server.
```

`ss -tulpn`

`tcp       LISTEN     0          128                     [::]:8088                  [::]:*         users:(("nginx",pid=27746,fd=9),("nginx",pid=27745,fd=9),("nginx",pid=27744,fd=9))`

##################################################################################################

Развёрнуты три виртуалки: ns01, client, ansible. Доступ у ним по паролю (SSH). Так как запускаем vagrant из под windows 10, провижининг хостов (ns01, client) будем делать с помощью виртуалки ansible.

На виртуалке ansible:

1. Установить Ansible (ansible 2.9.16)

```
yum install epel-release -y
yum update -y
yum install ansible -y
```

2. Отредактировать ansible.cfg

`vi /etc/ansible/ansible.cfg`

```
[defaults]
forks = 25
transport = ssh
host_key_checking = False
roles_path = /home/vagrant/selinux_dns_problems/provisioning/
retry_files_enabled = False
ansible_user = vagrant
inventory = /home/vagrant/selinux_dns_problems/provisioning/hosts

[diff]
always = True
context = 3
```

3. Создать директорию с ролью и скопировать в неё файлы из репозитория

`mkdir -p /home/vagrant/selinux_dns_problems/provisioning/`

`tree /home/`

```
/home/
└── vagrant
    └── selinux_dns_problems
        └── provisioning
            ├── ansible.cfg
            ├── files
            │   ├── client
            │   │   ├── motd
            │   │   ├── resolv.conf
            │   │   └── rndc.conf
            │   ├── named.zonetransfer.key.special
            │   └── ns01
            │       ├── named.50.168.192.rev
            │       ├── named.conf
            │       ├── named.ddns.lab
            │       ├── named.ddns.lab.view1
            │       ├── named.dns.lab
            │       ├── named.dns.lab.view1
            │       ├── named.newdns.lab
            │       └── resolv.conf
            ├── hosts
            └── playbook.yml

6 directories, 15 files
```

4. Отредактировать инвентарь hosts и проверить доступность хостов

`vi /home/vagrant/selinux_dns_problems/provisioning/hosts`

```
[selinux_lab]
#192.168.50.10
#192.168.50.15
ns01
client

[selinux_lab:vars]
ansible_connection=ssh
ansible_ssh_user=vagrant
ansible_ssh_pass=vagrant
```

`ansible-inventory --graph --vars`

```
@all:
  |--@selinux_lab:
  |  |--client
  |  |  |--{ansible_connection = ssh}
  |  |  |--{ansible_ssh_pass = vagrant}
  |  |  |--{ansible_ssh_user = vagrant}
  |  |--ns01
  |  |  |--{ansible_connection = ssh}
  |  |  |--{ansible_ssh_pass = vagrant}
  |  |  |--{ansible_ssh_user = vagrant}
  |  |--{ansible_connection = ssh}
  |  |--{ansible_ssh_pass = vagrant}
  |  |--{ansible_ssh_user = vagrant}
  |--@ungrouped:
```

5. Отредактировать /etc/hosts

`vi /etc/hosts`

```
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
127.0.1.1 ansible ansible
192.168.50.10 ns01 ns01
192.168.50.15 client client
```

`ansible -m ping all`

```
client | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
ns01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python"
    },
    "changed": false,
    "ping": "pong"
}
```

6. Запустить ansible playbook

`ansible-playbook /home/vagrant/selinux_dns_problems/provisioning/playbook.yml`

7. При попытке удаленно (с хоста client) внести изменения в зону ddns.lab происходит следующее:

```
nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
update failed: SERVFAIL
> quit
```

8. Предположительно ошибка связана с SELinux

`ansible -m shell -a '/usr/sbin/sestatus' all`

```
client | CHANGED | rc=0 >>
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
ns01 | CHANGED | rc=0 >>
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux root directory:         /etc/selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
```

9. Запустить утилиту на хосте ns01 для траблшутинга (парсит audit.log и даёт рекомендации)

`sealert -a /var/log/audit/audit.log`

10. Проверить метки на хосте ns01

`ls -lZ /etc/named/`

```
drw-rwx---. root named unconfined_u:object_r:etc_t:s0   dynamic
-rw-rw----. root named system_u:object_r:etc_t:s0       named.50.168.192.rev
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab
-rw-rw----. root named system_u:object_r:etc_t:s0       named.dns.lab.view1
-rw-rw----. root named system_u:object_r:etc_t:s0       named.newdns.lab
```

11. Добавить метки рекурсивно к директории на хосте ns01

`chcon -R -t named_zone_t /etc/named/`

12. Повторить удаленное (с хоста client) внесение изменений в зону ddns.lab:

```
nsupdate -k /etc/named.zonetransfer.key
> server 192.168.50.10
> zone ddns.lab
> update add www.ddns.lab. 60 A 192.168.50.15
> send
> quit
```

Изменения внесены без возникновения ошибки
