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

1. Клонируем ропозиторий: `git clone ...`
2. Переходим в папку с проектом: `cd dns/vagrant-bind`
3. Выполняем запуск: `vagrant up`

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

* client доступны зоны dns.lab и newdns.lab, но в зоне dns.lab представлены записи согласно настроенной для него [ресурсной таблицы](vagrant-bind/provisioning/named.dns.lab.client):

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
* client2 доступна только одна [зона dns.lab](vagrant-bind/provisioning/named.dns.lab):

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

В [playbook ansible](vagrant-bind/provisioning/playbook.yml) добавил 3 задачи для ns01 и ns02:

```yml
  - name: Setsebool for named_zone_t
    ansible.builtin.command:
      name: setsebool -P named_write_master_zones 1
  - name: SElinux the crutch for /etc/named
    sefcontext:
      target: "/etc/named(/.*)?"
      setype: named_zone_t
      state: present
  - name: Apply context
    command: restorecon -R -v /etc/named
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

[Полезная ссылка по SELinux](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/sect-security-enhanced_linux-working_with_selinux-selinux_contexts_labeling_files)

[Bind 9 Official Docs](https://bind9.readthedocs.io/en/latest/index.html)

[Полезная ссылка по git submodules](https://git-scm.com/book/ru/v2/%D0%98%D0%BD%D1%81%D1%82%D1%80%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D1%8B-Git-%D0%9F%D0%BE%D0%B4%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D0%B8)