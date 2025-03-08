## Part 1. Удаленное конфигурирование узла через Ansible

Составляем **Vagrantfile** для запуска 3 машин, в нем также указываем прокинутые порты на **node01** для доступа к пока еще не развернутому микросервисному приложению.

![vagrantfile](screenshots/image1.png)
 
Заходим на **manager** командой `vagrant ssh manager`

Попробуем подключиться к **node01** по ssh с паролем. Для этого на node01 нужно в файле **/etc/ssh/sshd_config** перевести поле **PasswordAuthentication** в **yes**. Применить изенения командой `sudo systemctl restart sshd`. Пытаемся подключиться:

![ssh_password](screenshots/image2.png)

Генерируем **ssh** ключ командой `ssh-keygen`. Командой `ssh-copy-id` прокидываем публичный ключ на **node01**. Теперь можем подключаться без пароля.

Копируем в домашний каталог **manager** исходники сервисов и **docker-compose.yaml** командой `cp -r /vagrant/src ~/`

Установим **Ansible** на **manager**: `sudo apt update`; `sudo apt install ansible`. Создадим директорию **ansible** и в ней inventory-файл **hosts**.

![inventory](screenshots/image3.png)

Проверим доступность указанных серверов командой `ansible -i hosts all -m ping`

![ping](screenshots/image4.png)

Настроим на **node02** подключение по **SSH** ключу как с первой машиной. После настройки:

![ping_all](screenshots/image5.png)


Напишем **playbook.yml**.
--- 
    hosts: servers // inventory файл
    remote_user: vagrant // user под которым будет подключаться ansible
    become: true // root права
    tasks:
        - name: install dependencies // устанавливаем зависимости для докера
        apt:
            update_cache: yes
            pkg:
            - apt-transport-https
            - ca-certificates
            - curl
            - gnupg-agent
            - software-properties-common

        - name: add GPG key // добавим gpg ключ для последующего скачивания из репозитория
        apt_key:
            url: https://download.docker.com/linux/ubuntu/gpg
            state: present

        - name: add docker repository to apt
        apt_repository:
            repo: deb https://download.docker.com/linux/ubuntu bionic stable
            state: present

        - name: install docker // пакеты для работы с docker, docker compose
        apt:
            pkg:
            - containerd.io
            - docker-ce
            - docker-ce-cli

        - name: check docker is active // служба докера должна быть активна
        service:
            name: docker
            state: started
            enabled: yes

        - name: check for group existence // должна присутствовать группа docker
        group:
            name: docker

        - name: adding vagrant user to docker group // vagrant должен быть в docker
        user:
            name: vagrant
            groups: docker
            append: yes

        - name: copy docker-compose.yaml // копируем docker-compose.yaml с менеджера
        copy:
            src: /home/vagrant/src/docker-compose.yaml
            dest: /home/vagrant/docker-compose.yaml

        - name: run docker compose // запускаем docker compose
        community.docker.docker_compose_v2:
            project_src: /home/vagrant
            state: present
            pull: always


Для запуска docker compose необходимо установить модуль который за это отвечает. Его нет в **ansible-core**, его надо скачивать отдельно. Для установки нужна более новая версия **ansible**, которую можно обновить только через pip. Выполняем эти команды:

`sudo apt install python3-pip`

`pip install --upgrade ansible`

`ansible-galaxy collection install community.docker`

После этого можно запустить плейбук командой `ansible-playbook -i hosts playbook.yml`

![playbook_run](screenshots/image7.png)

Запускаем тесты в **Postman**:

![postman](screenshots/image6.png)

Создадим 3 роли:

![roles_init](screenshots/image8.png)

Для работы с **postgres** понадобится также установить community модуль командой и прочие зависимости: 

`ansible-galaxy collection install community.postgresql`

Роль **application** фактически мы уже написали.

tasks/main.yml роли **apache**
---
    # tasks file for application
    - name: install apache
    apt:
        name: apache2
        update_cache: yes // устанавливаем apache и все работает из коробки

vars/main.yml роли postgresql
---
    # Variables for the role 
    db_user: myuser
    db_password: mypassword // устанавливаем параметры доступа к бд
    db_name: mydatabase

tasks/main.yml роли postgresql 
---
    - name: add postgresql apt key
    apt_key:
        url: https://www.postgresql.org/media/keys/ACCC4CF8.asc
        state: present

    - name: add postgresql repository
    apt_repository:
        repo: deb http://apt.postgresql.org/pub/repos/apt/ {{ ansible_distribution_release }}-pgdg main
        state: present

    - name: install postgresql and dependencies
    apt:
        name:
        - acl
        - libpq-dev
        - python3-psycopg2
        - postgresql-12
        update_cache: yes
        state: present

    - name: enable and start postgresql service
    service:
        name: postgresql
        enabled: yes
        state: started

    - name: configure pg_hba.conf
    lineinfile:
        path: /etc/postgresql/12/main/pg_hba.conf
        line: 'host    all             all             10.0.2.2/32             trust'
        state: present

    - name: configure postgresql.conf
    lineinfile:
        path: /etc/postgresql/12/main/postgresql.conf
        line: 'listen_addresses = *'
        state: present     // меняем конфиги чтобы затем подключится с хоста на ноду

    - name: create database user
    become: yes
    become_user: postgres
    become_method: sudo
    vars:
        ansible_become_allow_world_readable_tmpfiles: True
    postgresql_user:
        name: "{{ db_user }}"
        password: "{{ db_password }}"
        role_attr_flags: SUPERUSER

    - name: create database
    become: yes
    become_user: postgres
    become_method: sudo
    vars:
        ansible_become_allow_world_readable_tmpfiles: True
    postgresql_db:
        name: "{{ db_name }}"
        encoding: UTF-8
        owner: "{{ db_user }}"

    - name: create table
    become: yes
    become_user: postgres
    become_method: sudo
    vars:
        ansible_become_allow_world_readable_tmpfiles: True
    postgresql_table:
        db: "{{ db_name }}"
        name: table
        columns:
        - key SERIAL PRIMARY KEY
        - value text

    - name: insert values into table
    become: yes
    become_user: postgres
    become_method: sudo
    vars:
        ansible_become_allow_world_readable_tmpfiles: True
    postgresql_query:
        db: "{{ db_name }}"
        query:
        INSERT INTO "table" (value) VALUES ('sometext1'), ('sometext2'), ('sometext3')

## Part 2. Service Discovery
