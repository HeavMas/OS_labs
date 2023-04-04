# Лабораторная работа в Visual Studio code Ansible 2 #
===========================

## inventory файл ##
---------------------------
Файл содержит множество параметров ip-хоста, порт хоста, путь до приватного ключа. Для сверки данных в inventory файле  можно использовать команду "vagrant ssh-config" 

## ansible.cfg ###
-------------------
Чтобы дальнейшем каждый раз явно указывать наш инвентори файл в
командной строке. 
1.  Inventory - указывает на файл, в котором перечислены ваши хосты
2. remote_user - Устанавливает пользователя для входа в систему для целевых машин. Если пусто, используется значение по умолчанию подключаемого модуля, обычно это пользователь, выполняющий Ansible в данный момент
3. host_key_checking - Проверка ключа перед подключением 
4. transport - Используемый подключаемый модуль по умолчанию, опция «умный» будет переключаться между «ssh» и «paramiko» в зависимости от ОС контроллера и версии ssh.

## epel.yml ##
--------------
epel.yml - простой Playbook который будет выполнять  установку пакета epel-release

## nginx.yml ##
---------------
nginx.yml - Playbook для
установки NGINX
tags - можно вывести в консоль список тегов и
выполнить, например, только часть из задач описанных в плейбуке, а именно установку
NGINX.
template - копирует подготовленный шаблон на хост

## nginx.conf.j2 ##
-------------------
nginx.conf.j2 - шаблона для конфига NGINX

### Что делать ###

1. Пишем  чтобы убедится, что Vagrantfile не содержит ошибок - vagrant status
2. Поднимаем виртуалку - vagrant up
3. Теперь требуется изменить файлы в inventory для этого пишем команду - vagrant ssh-config
4. Заходим в inventory при помощи команды - nano inventory
5. Пишем - "cat inventory" чтобы проверить что файлы из конфига совпадают с информацией из ssh-config-а
6. Убедимся, что Ansible может управлять нашим хостом - "ansible nginx -i inventory -m ping"
7. Чтобы не пришлось в дальнейшем каждый раз явно указывать наш инвентори файл в
   командной строке, создадим файл конфигурации ansible.cfg
   Для этого в текущем каталоге создадим файл ansible.cfg со следующим содержанием - "cat ansible.cfg" 
"   [defaults]
    inventory = inventory
    remote_user= vagrant
    host_key_checking = False
    transport=smart"
8. Теперь из инвентори можно убрать информацию о пользователе 
   " nginx ansible_host=127.0.0.1 ansible_port=2203 ansible_ssh_private_key_file=.vagrant/ma
chines/nginx/virtualbox/private_key "
9. Посмотрим какое ядро установлено на хосте - "ansible nginx -m command -a "uname -r""
10.  Проверим статус сервиса firewalld - "ansible nginx -m systemd -a name=firewalld"
11. Установим пакет epel-release на наш хост - "ansible nginx -m yum -a "name=epel-release state=present" -b"
12. Напишем простой Playbook который будет выполнять одну из задач, которые мы делали на
    прошлом слайде - а именно: установку пакета epel-release. Создайте файл epel.yml со
    следующим содержимым. Внимательно соблюдайте отступы, т. к. YaML очень чувствителен
    к синтаксису: 
   "- name: Install EPEL Repo
    hosts: webservers
    become: true
    tasks:
        - name: Install EPEL Repo package from standard repo
        yum:
            name: epel-release
            state: present "
13. После чего запустите выполнение Playbook - "ansible-playbook epel.yml"
14. Теперь собственно приступим к выполнению домашнего задания и написания Playbook-а для
    установки NGINX. Будем писать его постепенно, шаг за шагом.За основу возьмем уже созданный нами плейбук epel.yml. Скопируйте этот файл с именем
  " nginx.yml (командой cp):
    - name: Install nginx package from epel repo
    hosts: webservers
    become: true
    tasks:
        - name: Install EPEL Repo package from standard repo
        yum:
            name: epel-release
            state: present
        - name: install nginx from repo
        yum:
            name: nginx
            state: latest
        tags:
            nginx-package
            packages "
15. Теперь можно вывести в консоль список тегов и
    выполнить, например, только часть из задач описанных в плейбуке, а именно установку
    NGINX. В нашем случае так, например, можно осуществлять его обновление.
    Выведем в консоль все теги - "ansible-playbook nginx.yml —list-tags"
16. Далее создадим файл шаблона для конфига NGINX, имя файла nginx.conf.j2 - "cat nginx.conf.j2". С следующим содержанием:
  " events {
    worker_connections 1024;
    }
    http {
    server {
    listen {{ nginx_listen_port }} default_server;
    server_name default_server;
    root /usr/share/nginx/html;
    location / {
    }
   }
  }"
17. И добавим в плейбук задачу, которая копирует подготовленный шаблон на хост. Для этой
    задачи используется модуль template - "cat nginx.yml"
 "- name: Install nginx package from epel repo
  hosts: webservers
  become: true
  vars:
    nginx_listen_port: 8080
  tasks:
    - name: Install EPEL Repo package from standard repo
      yum:
        name: epel-release
        state: present
    - name: install nginx from repo
      yum:
        name: nginx
        state: latest
      tags:
        - nginx-package
        - packages
    - name: Create config file from template
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify:
        - restart nginx
      tags:
        - nginx-configuration
  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes "
18. Пишем  - "ansible-playbook nginx.yml"
19. Теперь, чтобы проверить работу NGINX на нестандартном порту, нам надо выяснить IP
    адрес, который получила виртуальная машина при создании vagrant-ом
    Это можно сделать командой - "vagrant ssh -c "ip addr show""
20. В полученном выводе найдите описание сетевых интерфейсов. Первый интерфейс —
    локальный loopback, второй — автоматически добавленный вагрантом, предназначеный для
    интерконнекта ядра вагранта к виртуальной машине и не доступен снаружи виртуалки. А
    третий как раз публичный и к нему можно обращаться.
    Попингуйте этот адрес, убедитесь, что он отвечает.
    Затем в браузере откройте страницу
   " http://ip_address:8080 " 
