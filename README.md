## DNS

Задачи:

- [X] взять стенд https://github.com/erlong15/vagrant-bind
- [X] добавить еще один сервер client2
- [X] завести в зоне dns.lab имена: 
  - web1 - смотрит на клиент1
  - web2 смотрит на клиент2
- [X] завести еще одну зону newdns.lab
  - завести в ней запись www - смотрит на обоих клиентов
- [X] настроить [split-dns](#настройка-split-dns):  
  - клиент1 видит обе зоны, но в зоне dns.lab только web1
  - клиент2 видит только dns.lab
- [X] настроить все без выключения [selinux](#selinux-устранение-проблем)
- [X] Формат сдачи ДЗ - vagrant + ansible

### Запуск проекта

1. Клонируем ропозиторий: `git clone https://github.com/mmmex/vagrant-bind.git`
2. Переходим в папку с проектом: `cd vagrant-bind`
3. Выполняем запуск: `vagrant up`

* В процессе доставки конфигурации ansible на ns01 [у меня возникла ошибка](#ошибка-при-запуске-стенда). Решение: `vagrant provision ns01; vagrant up`

* Будет поднято 3 ВМ:

| Имя хоста | IP-адрес      | Роль              |
|:----------|:-------------:|:------------------|
| ns01      | 192.168.50.10 | master, recursive |
| ns02      | 192.168.50.11 | slave, recursive  |
| client    | 192.168.50.15 | client            |
| client2   | 192.168.15.16 | client            |

* Имеются 3 зоны на master:

| Зона       | Ресурсные записи (hostname TYPE ip)                                                                    |
|:-----------|--------------------------------------------------------------------------------------------------------|
| ddns.lab   | ns01 **A** 192.168.50.10; ns02 **A** 192.168.50.11                                                     |
| dns.lab    | ns01 **A** 192.168.50.10; ns02 **A** 192.168.50.11; web1 **A** 192.168.50.15; web2 **A** 192.168.50.16 |
| newdns.lab | ns01 **A** 192.168.50.10; ns02 **A** 192.168.50.11; www **A** 192.168.50.15; www **A** 192.168.50.16   |

### Настройка split-dns

* client доступны зоны dns.lab и newdns.lab, но в зоне dns.lab представлены записи согласно настроенной для него [ресурсной таблицы](https://github.com/mmmex/vagrant-bind/blob/master/provisioning/named.dns.lab.client):

**/etc/named.conf:**
```bash
key "client-key" {
        algorithm hmac-sha256;
        secret "BcXAlSSl+HJTWTy0wC24hrFOKdadVy70oyKwiEqYG68=";
};

acl client { !key client2-key; key client-key; 192.168.50.15; };

view "client" {
    match-clients { client; };

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "/etc/named/named.dns.lab.client";
        also-notify { 192.168.50.11 key client-key; };
    };

    // newdns.lab zone
    zone "newdns.lab" {
        type master;
        file "/etc/named/named.newdns.lab";
        also-notify { 192.168.50.11 key client-key; };
    };
};
```
* client2 доступна только одна [зона dns.lab](https://github.com/mmmex/vagrant-bind/blob/master/provisioning/named.dns.lab):

**/etc/named.conf:**
```bash
key "client2-key" {
        algorithm hmac-sha256;
        secret "tCNiBFlaheUuHW6FOa3DJC42BWhdszDhVW8eWgwLlYg=";
};

acl client2 { !key client-key; key client2-key; 192.168.50.16; };

view "client2" {
    match-clients { client2; };

    // dns.lab zone
    zone "dns.lab" {
        type master;
        file "/etc/named/named.dns.lab";
        also-notify { 192.168.50.11 key client2-key; };
    };

    // dns.lab zone reverse
    zone "50.168.192.in-addr.arpa" {
        type master;
        file "/etc/named/named.dns.lab.rev";
        also-notify { 192.168.50.11 key client2-key; };
    };
};
```

#### Проверка

Проверяем client:
```bash
[vagrant@client ~]$ ping www.newdns.lab
PING www.newdns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.114 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.035 ms
^C
--- www.newdns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1001ms
rtt min/avg/max/mdev = 0.035/0.074/0.114/0.040 ms
[vagrant@client ~]$ ping web2.dns.lab
ping: web2.dns.lab: Name or service not known
[vagrant@client ~]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from client (192.168.50.15): icmp_seq=1 ttl=64 time=0.013 ms
64 bytes from client (192.168.50.15): icmp_seq=2 ttl=64 time=0.032 ms
^C
--- web1.dns.lab ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1002ms
rtt min/avg/max/mdev = 0.013/0.022/0.032/0.010 ms
```

Проверяем client2:
```bash
[vagrant@client2 ~]$ ping web1.dns.lab
PING web1.dns.lab (192.168.50.15) 56(84) bytes of data.
64 bytes from 192.168.50.15 (192.168.50.15): icmp_seq=1 ttl=64 time=1.54 ms
^C
--- web1.dns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 1.542/1.542/1.542/0.000 ms
[vagrant@client2 ~]$ ping web2.dns.lab
PING web2.dns.lab (192.168.50.16) 56(84) bytes of data.
64 bytes from client2 (192.168.50.16): icmp_seq=1 ttl=64 time=0.060 ms
^C
--- web2.dns.lab ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 0.060/0.060/0.060/0.000 ms
[vagrant@client2 ~]$ ping www.newdns.lab
ping: www.newdns.lab: Name or service not known
```

### SELinux устранение проблем

В логах сервиса named на ВМ ns02 такие ошибки:
```log
Jan 28 02:49:24 ns02 named[616]: dumping master file: /etc/named/tmp-oLRkyApU1s: open: permission denied
Jan 28 02:52:15 ns02 named[616]: dumping master file: /etc/named/tmp-HVOwFAlo37: open: permission denied
Jan 28 02:53:32 ns02 named[616]: dumping master file: /etc/named/tmp-DFhO6OH9QM: open: permission denied
Jan 28 02:54:16 ns02 named[616]: dumping master file: /etc/named/tmp-e37cg54tOV: open: permission denied
Jan 28 03:00:54 ns02 named[616]: dumping master file: /etc/named/tmp-ntupCSfOgY: open: permission denied
Jan 28 03:06:12 ns02 named[616]: dumping master file: /etc/named/tmp-bemFwpTzfo: open: permission denied
Jan 28 03:07:02 ns02 named[616]: dumping master file: /etc/named/tmp-KxpQ8THmqO: open: permission denied
Jan 28 03:08:31 ns02 named[616]: dumping master file: /etc/named/tmp-RKl4x8Qq6o: open: permission denied
```

Смотрим логи audit:
```bash
[root@ns02 ~]# cat /var/log/audit/audit.log | audit2why
type=AVC msg=audit(1674875346.953:1548): avc:  denied  { create } for  pid=4391 comm="isc-worker0000" name="tmp-irKLEjHg3j" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1674875346.982:1549): avc:  denied  { create } for  pid=4391 comm="isc-worker0000" name="tmp-odzDs5dMSs" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1674875346.993:1550): avc:  denied  { create } for  pid=4391 comm="isc-worker0000" name="tmp-sgIBANOJgS" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1674875347.001:1551): avc:  denied  { create } for  pid=4391 comm="isc-worker0000" name="tmp-Gw0flFBPwe" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

type=AVC msg=audit(1674875971.631:1619): avc:  denied  { create } for  pid=4391 comm="isc-worker0000" name="tmp-Wz70QZndHT" scontext=system_u:system_r:named_t:s0 tcontext=system_u:object_r:etc_t:s0 tclass=file permissive=0

        Was caused by:
                Missing type enforcement (TE) allow rule.

                You can use audit2allow to generate a loadable module to allow this access.

[root@ns02 ~]# cat /var/log/audit/audit.log | audit2allow


#============= named_t ==============

#!!!! WARNING: 'etc_t' is a base type.
allow named_t etc_t:file create;
```

Проверяю файлы зон, которые должны приехать с master:

```bash
[root@ns02 ~]# ls -alZ /etc/named
drw-rwx---. root named system_u:object_r:etc_t:s0       .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
```

Посмотреть в каком каталоге должны лежать файлы зон с типом `named_zone_t`:
```bash
[root@ns02 ~]# semanage fcontext -l | grep named_zone_t
/var/named(/.*)?                                   all files          system_u:object_r:named_zone_t:s0 
/var/named/chroot/var/named(/.*)?                  all files          system_u:object_r:named_zone_t:s0 
```

Файлы зон должны находится в каталоге `/var/named`, и ранее я уже [разбирал данную проблему](https://github.com/mmmex/selinux#%D0%BE%D0%B1%D0%B5%D1%81%D0%BF%D0%B5%D1%87%D0%B8%D1%82%D1%8C-%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BE%D1%81%D0%BF%D0%BE%D1%81%D0%BE%D0%B1%D0%BD%D0%BE%D1%81%D1%82%D1%8C-%D0%BF%D1%80%D0%B8%D0%BB%D0%BE%D0%B6%D0%B5%D0%BD%D0%B8%D1%8F-%D0%BF%D1%80%D0%B8-%D0%B2%D0%BA%D0%BB%D1%8E%D1%87%D0%B5%D0%BD%D0%BD%D0%BE%D0%BC-selinux). Есть два пути, в прошлый раз я перемещал каталог с зонами в /var/named, в данном примере изменю тип контекста безопаности для каталога /etc/named.

В [playbook ansible](https://github.com/mmmex/vagrant-bind/blob/master/provisioning/playbook.yml) добавил 3 задачи для ns01 и ns02:

```yml
  - name: Setsebool for named_zone_t
    seboolean:
      name: named_write_master_zones
      state: yes
      persistent: yes
  - name: SElinux the crutch for /etc/named
    sefcontext:
      target: "/etc/named(/.*)?"
      setype: named_zone_t
      state: present
  - name: Apply context
    command: restorecon -R -v /etc/named
```

Те-же действия с командной строки:
```bash
[root@ns01 ~]# setsebool -P named_write_master_zones 1
[root@ns01 ~]# semanage fcontext -a -t named_zone_t "/etc/named(/.*)?"
[root@ns01 ~]# restorecon -R -v /etc/named
restorecon reset /etc/named context system_u:object_r:etc_t:s0->system_u:object_r:named_zone_t:s0
restorecon reset /etc/named/named.ddns.lab context system_u:object_r:etc_t:s0->system_u:object_r:named_zone_t:s0
restorecon reset /etc/named/named.dns.lab context system_u:object_r:etc_t:s0->system_u:object_r:named_zone_t:s0
restorecon reset /etc/named/named.dns.lab.client context system_u:object_r:etc_t:s0->system_u:object_r:named_zone_t:s0
restorecon reset /etc/named/named.dns.lab.rev context system_u:object_r:etc_t:s0->system_u:object_r:named_zone_t:s0
restorecon reset /etc/named/named.newdns.lab context system_u:object_r:etc_t:s0->system_u:object_r:named_zone_t:s0
```

В каталоге на ns02 должы приехать зоны с мастера:
```bash
[root@ns02 ~]# ls -alZ /etc/named
drw-rwx---. root  named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root  root  system_u:object_r:etc_t:s0       ..
-rw-r--r--. named named system_u:object_r:named_zone_t:s0 named.ddns.lab
-rw-r--r--. named named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-r--r--. named named system_u:object_r:named_zone_t:s0 named.dns.lab.rev
-rw-r--r--. named named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

Также посмотрим в каталог /etc/named на ns01:
```bash
[root@ns01 ~]# ls -alZ /etc/named
drw-rwx---. root named system_u:object_r:named_zone_t:s0 .
drwxr-xr-x. root root  system_u:object_r:etc_t:s0       ..
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.ddns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.client
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.dns.lab.rev
-rw-rw----. root named system_u:object_r:named_zone_t:s0 named.newdns.lab
```

#### Ошибка при запуске стенда

После запуска стенда `vagrant up`, на сервере ns01 в процессе работы скрипта ansible возникает ошибка:
```bash
TASK [Setsebool for named_zone_t] **********************************************
fatal: [ns01]: FAILED! => {"changed": false, "module_stderr": "Shared connection to 127.0.0.1 closed.\r\n", "module_stdout": "/bin/sh: line 1:  5017 Убито              /usr/bin/python /home/vagrant/.ansible/tmp/ansible-tmp-1674916369.157325-230713544677949/AnsiballZ_seboolean.py\r\n", "msg": "MODULE FAILURE\nSee stdout/stderr for the exact error", "rc": 137}                                                                                                                  

PLAY RECAP *********************************************************************
ns01                       : ok=13   changed=11   unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

После повторного применения скрипта командой `vagrant provision ns01` все проходит удачно. В чем причина понять сразу не получилось.

Мое окружение:
```bash
test@test-virtual-machine:~/Otus/dns/vagrant-bind$ vagrant -v
Vagrant 2.3.0
test@test-virtual-machine:~/Otus/dns/vagrant-bind$ ansible --version
ansible 2.9.6
  config file = /etc/ansible/ansible.cfg
  configured module search path = ['/home/test/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python3/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 3.8.10 (default, Nov 14 2022, 12:59:47) [GCC 9.4.0]
test@test-virtual-machine:~/Otus/dns/vagrant-bind$ uname -a
Linux test-virtual-machine 5.15.0-58-generic #64~20.04.1-Ubuntu SMP Fri Jan 6 16:42:31 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
```

---

[Полезная ссылка по SELinux](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-security-enhanced_linux-working_with_selinux-selinux_contexts_labeling_files)

[Bind 9 Official Docs](https://bind9.readthedocs.io/en/latest/index.html)

[Полезная ссылка по git submodules](https://git-scm.com/book/ru/v2/%D0%98%D0%BD%D1%81%D1%82%D1%80%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D1%8B-Git-%D0%9F%D0%BE%D0%B4%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D0%B8)