Раздел 1. Установка дистрибутива (5 баллов)
В программе установки дистрибутива выполните следующие настройки и затем установите систему.
Если программа установки не позволяет выполнить данные настройки, то выполните их после установки системы в командной строке.
1.1. Настройка сети
Настроена
1.2. Настройка имени хоста
Установлено имя хоста mephi-2026.domain.local. + прописан в hosts
1.3. Проверка сетевой связности
ping до 8.8.8.8. есть, DNS тоже работает

Раздел 2. Управление программным обеспечением (5 баллов)
2.1. Обновление дистрибутива
sudo dnf update 
2.2. Установка пакетов из репозиториев
Установите следующие пакеты с помощью команды dnf:
sudo dnf install nginx libcap-ng-utils
2.3. Установка локального RPM-пакета
sudo dnf download --destdir=/tmp tcpdump
ls -l /tmp/tcpdump*.rpm
sudo rpm -ivh /tmp/tcpdump-4.99.5-1.red80.x86_64.rpm 

Раздел 3. Управление файловыми системами (5 баллов)
3.1. Создание файловой системы
Создал партицию с помощью parted, отформатировал под ext4, забыл создать метку, отмонтировал и присвоил метку.
3.2. Монтирование файловой системы
Создал точку монтирования /mephi-web.
Настройте автоматическое монтирование созданного раздела при начальной загрузке, используя метку тома в качестве идентификатора файловой системы.
UUID=547583a3-3a49-4bff-8b4f-b774ac7a5664 /mephi-web              ext4    defaults        0 2
Выполните ручное монтирование раздела.
sudo mount /dev/vdb1 /mephi-web/

Раздел 4. Управление сервисами (5 баллов)
4.1. Управление веб-сервером
sudo systemctl start nginx
sudo systemctl enable nginx
4.2. Журналирование
sudo journalctl -u nginx.service

Раздел 5. Управление доступом (15 баллов)
5.1. Дискреционное управление доступом (DAC)
 5.1.1 Создана директория /data/mephi-2026
 5.1.2 созданы группы  devteam и curators
 5.1.3 Созданы 3 пользователя именами user1, user2 и user3 в группе devteam, срок действия пароля(90дней) и длина(12) уже заранее отконфигурирована в конфигах(приложены в гите).
 5.1.4 Созданы 2 куратора в группе curators
 5.1.5 Назначим директории оунера и группу sudo chown root:devteam /data/mephi-2026
 5.1.6 Назначим базовые права sudo chmod 2770 /data/mephi-2026 (SGID-бит — новые файлы наследуют группу директории (devteam))
 5.1.7 Настоим ACL:# Разработчики: полный доступ
      sudo setfacl -m g:devteam:rwx /data/mephi-2026
      # Кураторы: чтение и вход в директорию (без записи)
      sudo setfacl -m g:curators:r-x /data/mephi-2026
      # Остальные: нет доступа
      sudo setfacl -m o::--- /data/mephi-2026
 5.1.8 Не забудем про "по умолчанию":
      # Разработчики будут иметь rwx на новые объекты
      sudo setfacl -d -m g:devteam:rwx /data/mephi-2026
      # Кураторы будут иметь r-x на новые объекты
      sudo setfacl -d -m g:curators:r-x /data/mephi-2026
      # Остальные не получат никакого доступа
      sudo setfacl -d -m o::--- /data/mephi-2026
  Проверял на файловой системе - все работает по условию.
5.2. Привилегии (уменьшение количества set-UID-программ)
  при установке в RedОС 8 suid бит не стоял на tcpdump.
  для работы под усером нужно дать соотв. привилегию 
  sudo setcap cap_net_raw,cap_net_admin=eip $(which tcpdump)
  $ sudo -u user1 tcpdump 
    tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
    listening on enp7s0, link-type EN10MB (Ethernet), snapshot length 262144 bytes
    15:33:41.189822 IP mephi-2026.domain.local.ssh > 62.117.102.76.knetd: Flags [P.], seq 3978132895:3978133091, ack 3777447540, win 492, options [nop,nop,TS val 103471462 ecr 2456336413], length 196
    15:33:41.197856 IP 62.117.102.76.knetd > mephi-2026.domain.local.ssh: Flags [.], ack 196, win 501, options [nop,nop,TS val 2456336685 ecr 103471462], length 0
    15:33:41.277664 IP mephi-2026.domain.local.60456 > 10.129.0.2.domain: 28588+ [1au] PTR? 76.102.117.62.in-addr.arpa. (55)

5.3. Мандатное управление доступом (MAC)
Убедитесь, что SELinux работает в режиме "Enforcing" (включен).
Веб-сервер nginx должен отдавать файлы из нестандартной директории /mephi-web.
Настройки сайта nginx:
server {
    listen       80 default_server;
    listen       [::]:80 default_server;
    server_name  _;
    root         /mephi-web;
    location / {
    }
}
  5.3.1 Проверяем контекст директории ls -Z /mephi-web
  5.3.2 Ставим утилиту semanage
   sudo dnf install -y policycoreutils-python-utils
   sudo semanage fcontext -a -t httpd_sys_content_t "/mephi-web(/.*)?"
   sudo restorecon -Rv /mephi-web # применяем контекст
   Результат:
    [root@mephi-2026 result]# curl http://localhost
   <!DOCTYPE html>
   <html>
    <head>
     <title>MEPhI Web Server</title>
    </head>
    <body>
      <h1>Добро пожаловать на нестандартный веб-сервер под защитой SELinux!</h1>
      <p>Hello from Student: 372192</p>
    </body>
  </html>

Раздел 6. Аутентификация (5 баллов)
6.1. Ограничение входа для пользователей
   В /etc/pam.d/login включаем account required pam_access.so. Теперь логин с консоли будет согласовываться с файлом /etc/security/access.conf.
   Чтобы не было возможности зайти через su, тоже самое сделано в   /etc/pam.d/su
Кураторы проекта не должны иметь возможности входить в систему локально (через консоль). Все остальные пользователи должны иметь возможность входить в систему.
6.2. Управление паролями
Пользователь должен менять свой пароль каждые 90 дней.
 /etc/login.defs поле PASS_MAX_DAYS = 90
Минимальная длина пароля пользователя должна быть 12 символов.
  /etc/security/pwquality.conf  поле minlen = 12


