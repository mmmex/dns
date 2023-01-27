## DNS

Задачи:

- [X] взять стенд https://github.com/erlong15/vagrant-bind
- [X] добавить еще один сервер client2
- [X] завести в зоне dns.lab имена: 
        - web1 - смотрит на клиент1
        - web2 смотрит на клиент2
- [ ] завести еще одну зону newdns.lab
        - завести в ней запись www - смотрит на обоих клиентов
- [ ] настроить split-dns:  
        - клиент1 - видит обе зоны, но в зоне dns.lab только web1
        - клиент2 видит только dns.lab
- [ ] настроить все без выключения selinux
- [ ] Формат сдачи ДЗ - vagrant + ansible

[Полезная ссылка по git submodules](https://git-scm.com/book/ru/v2/%D0%98%D0%BD%D1%81%D1%82%D1%80%D1%83%D0%BC%D0%B5%D0%BD%D1%82%D1%8B-Git-%D0%9F%D0%BE%D0%B4%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D0%B8)

