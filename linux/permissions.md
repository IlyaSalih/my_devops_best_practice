# Разрешения и контроль доступа
---
## Как читать права
```
-rw-r--r--  1  www-data  www-data  4096  Jan 1 12:00  config.yaml
│└─┬──┘└─┬──┘└─┬──┘
│  │     │     └── группа
│  │     └──────── владелец
│  └────────────── права (owner | group | others)
└───────────────── тип: - файл, d директория, l symlink
```
Права в числах: `4` = read · `2` = write · `1` = execute
`755` = `rwxr-xr-x` · `644` = `rw-r--r--` · `600` = `rw-------`
Для директорий execute означает другое:
`r` — можно делать `ls` (видеть содержимое)
`x` — можно делать `cd` и обращаться к файлам внутри
без `x` — даже зная имя файла, открыть его нельзя
---
## Сценарий 1: Permission Denied — диагностика
### Симптом: приложение падает с `Permission denied`, непонятно где именно.
```bash
# Шаг 1 — под каким пользователем работает процесс
ps aux | grep nginx
ps aux | grep php-fpm
# Запомни USER из вывода — это и есть реальный владелец процесса

# Шаг 2 — проверить права на конкретный файл/директорию
ls -la /var/www/html/
stat /var/www/html/config.php

# Шаг 3 — проверить всю цепочку пути
# Если процесс не может добраться до /var/www/html/file.txt —
# проверяй права на КАЖДУЮ директорию в пути
namei -l /var/www/html/config.php
# namei покажет права на каждый компонент пути — сразу видно где обрыв
```
### Пример вывода `namei -l`:
```
f: /var/www/html/config.php
drwxr-xr-x root     root     /
drwxr-xr-x root     root     var
drwxr-xr-x root     root     www
drwx------ root     root     html       ← www-data не может зайти сюда
-rw-r--r-- www-data www-data config.php
```
```bash
# Шаг 4 — проверить ACL (могут перекрывать обычные права)
getfacl /var/www/html/config.php

# Шаг 5 — проверить SELinux/AppArmor если включён
getenforce                             # SELinux: Enforcing / Permissive / Disabled
aa-status                             # AppArmor статус
dmesg | grep -i "avc\|apparmor" | tail -20   # отказы в логах
```
### Решение:
```bash
# Дать права правильно — не 777
chown www-data:www-data /var/www/html/config.php
chmod 640 /var/www/html/config.php    # владелец rw, группа r, остальные ничего

# Если нужно дать доступ группе без смены владельца
chmod g+r /var/www/html/config.php
usermod -aG www-data deployuser       # добавить пользователя в группу
# После usermod нужен новый логин — текущая сессия не обновится
```
> `chmod 777` — не решение. Это дыра в безопасности. Если соблазняет — значит не разобрался с реальной причиной.
---
## Сценарий 2: Права в production — nginx, cron, docker
### nginx / веб-сервер
```bash
# nginx работает как www-data, но master-процесс — root
ps aux | grep nginx
# root      1234  nginx: master process
# www-data  1235  nginx: worker process

# Стандартная схема прав для веб-директории
chown -R www-data:www-data /var/www/html/
find /var/www/html -type d -exec chmod 755 {} \;   # директории: rwxr-xr-x
find /var/www/html -type f -exec chmod 644 {} \;   # файлы: rw-r--r--

# Конфиги с паролями (DB, API keys)
chmod 640 /var/www/html/.env          # только владелец и группа
chown root:www-data /var/www/html/.env  # root владеет, www-data только читает

# Директория для загрузок (нужна запись)
chown www-data:www-data /var/www/html/uploads/
chmod 750 /var/www/html/uploads/      # www-data пишет, группа читает, остальные — нет
```
### cron
```bash
# Симптом: cron job не работает — Permission denied в /var/log/cron
grep CRON /var/log/syslog | tail -20
journalctl -u cron | tail -20

# Cron запускает скрипт от имени владельца crontab
crontab -l                            # чья crontab — того пользователя и права
crontab -u www-data -l               # crontab другого пользователя (root)

# Типичная ошибка — скрипт не executable
chmod +x /opt/scripts/backup.sh

# Cron не наследует PATH из .bashrc — всегда используй абсолютные пути
# Плохо:
# * * * * * python script.py
# Хорошо:
# * * * * * /usr/bin/python3 /opt/scripts/script.py

# Проверить что скрипт реально работает от нужного пользователя
su -s /bin/bash www-data -c "/opt/scripts/backup.sh"
```
### docker
```bash
# Симптом: контейнер не может писать в примонтированную директорию
docker run -v /host/data:/app/data myimage
# Error: Permission denied на /app/data

# Шаг 1 — узнать под каким UID работает процесс в контейнере
docker inspect myimage | grep -i user
docker run --rm myimage id

# Шаг 2 — сопоставить с правами на хосте
ls -la /host/data
# Если UID в контейнере 1000, а директория принадлежит root — отказ

# Решение А — сменить владельца на хосте
chown -R 1000:1000 /host/data

# Решение Б — запустить контейнер с UID хоста
docker run -u $(id -u):$(id -g) -v /host/data:/app/data myimage

# Docker socket — частая проблема
# Симптом: Got permission denied while trying to connect to Docker daemon
ls -la /var/run/docker.sock           # обычно группа docker
usermod -aG docker username           # добавить пользователя в группу docker
# Группа docker = фактически root. Добавляй только доверенных пользователей.
```
---
## Сценарий 3: sudo и sudoers — тонкости и ошибки
```bash
# Проверить что разрешено текущему пользователю
sudo -l

# Типичный вывод:
# (ALL : ALL) ALL              — полный sudo
# (ALL) NOPASSWD: /bin/systemctl restart nginx   — конкретная команда без пароля
```
### Настройка sudoers
```bash
visudo                        # ВСЕГДА редактируй через visudo — проверяет синтаксис
visudo -f /etc/sudoers.d/devops  # отдельный файл для команды (рекомендуется)
```
### Примеры правил:
```bash
# Полный sudo для пользователя
username ALL=(ALL:ALL) ALL

# Конкретные команды без пароля (для деплой-скриптов)
deploy ALL=(ALL) NOPASSWD: /bin/systemctl restart nginx, /bin/systemctl restart php-fpm

# Группа может перезапускать сервисы
%devops ALL=(ALL) NOPASSWD: /bin/systemctl restart *, /bin/systemctl status *

# Запретить опасные команды даже при широких правах
deploy ALL=(ALL) NOPASSWD: ALL, !/bin/bash, !/bin/su, !/usr/bin/vim
```
### Типичные ошибки sudoers
```bash
# Ошибка 1: пробел перед NOPASSWD
# Плохо:  username ALL=(ALL) NOPASSWD : /bin/systemctl
# Хорошо: username ALL=(ALL) NOPASSWD: /bin/systemctl

# Ошибка 2: относительный путь вместо абсолютного
# Плохо:  username ALL=(ALL) NOPASSWD: systemctl
# Хорошо: username ALL=(ALL) NOPASSWD: /bin/systemctl

# Ошибка 3: забыли про sudo -E (переменные окружения)
# Скрипт деплоя работает локально но не через sudo — нет нужных ENV
sudo -E ./deploy.sh           # передать окружение в sudo

# Ошибка 4: пользователь добавлен в sudoers но не может
sudo -l 2>&1 | grep -i error
# Проверить что файл в /etc/sudoers.d/ имеет права 440
chmod 440 /etc/sudoers.d/devops
```
```bash
# Аудит — кто и что делал через sudo
grep sudo /var/log/auth.log | tail -30
journalctl | grep sudo | tail -30
```
---
SUID / SGID / Sticky Bit
SUID (Set User ID) — выполнять от имени владельца файла
```bash
# Найти все SUID файлы в системе
find / -perm /u+s -type f 2>/dev/null
find / -perm -4000 -type f 2>/dev/null   # то же, числом

# Классический пример — passwd
ls -la /usr/bin/passwd
# -rwsr-xr-x root root /usr/bin/passwd
# 's' вместо 'x' у владельца = SUID установлен
# passwd запускается от root даже когда вызывает обычный пользователь

# Установить SUID
chmod u+s /path/to/binary
chmod 4755 /path/to/binary
```
> SUID на скриптах (bash, python) не работает в Linux — ядро игнорирует его для interpreted файлов. Работает только для бинарников.
> Лишние SUID файлы = потенциальный вектор privilege escalation. Аудируй регулярно.
SGID (Set Group ID) — выполнять от имени группы / наследовать группу
```bash
# На файле — запускать от имени группы владельца
chmod g+s /usr/bin/somebinary
chmod 2755 /usr/bin/somebinary

# На директории — все новые файлы наследуют группу директории
chmod g+s /var/www/html/
# Без SGID: новый файл получает основную группу создателя
# С SGID:   новый файл получает группу директории (www-data)
# Полезно когда несколько пользователей работают с одной директорией

ls -la /var/www/
# drwxrwsr-x www-data www-data html
# 's' вместо 'x' у группы = SGID на директории
```
### Sticky Bit — удалять только свои файлы
```bash
# Классический пример — /tmp
ls -la / | grep tmp
# drwxrwxrwt root root tmp
# 't' вместо 'x' у others = sticky bit

# Без sticky: любой может удалить чужой файл в общей директории
# С sticky:   удалить файл может только его владелец или root

chmod +t /shared/uploads/
chmod 1777 /shared/uploads/   # числом

# Проверить
ls -la | grep "^d.*t "
find / -perm /+t -type d 2>/dev/null
```
---
## ACL — права тонкой настройки
### Стандартные Unix-права позволяют задать права только для одного владельца и одной группы. ACL позволяет задать права для любого пользователя или группы на конкретный файл.
```bash
# Проверить есть ли ACL на файле
getfacl /var/www/html/config.php
ls -la /var/www/html/config.php   # '+' в конце прав = есть ACL
# -rw-r--r--+ www-data www-data config.php

# Посмотреть ACL
getfacl /var/www/html/

# Добавить права конкретному пользователю
setfacl -m u:deployuser:r /var/www/html/config.php    # read
setfacl -m u:deployuser:rw /var/www/html/config.php   # read+write

# Добавить права группе
setfacl -m g:devops:rx /var/www/html/                 # read+execute на директорию

# Рекурсивно
setfacl -R -m u:deployuser:rx /var/www/html/

# Установить default ACL — новые файлы наследуют эти права
setfacl -d -m u:deployuser:rx /var/www/html/
setfacl -d -m g:devops:rx /var/www/html/

# Удалить ACL для пользователя
setfacl -x u:deployuser /var/www/html/config.php

# Удалить все ACL
setfacl -b /var/www/html/config.php
```
### Cценарий: CI/CD пользователь должен читать конфиги но не быть в группе www-data:
```bash
# Вместо того чтобы добавлять deploy в группу www-data (небезопасно)
setfacl -m u:deploy:r /var/www/html/.env
setfacl -m u:deploy:rx /var/www/html/
getfacl /var/www/html/.env   # проверить результат
```
> ACL работают только если файловая система смонтирована с опцией `acl`. На современных системах (ext4, xfs) включено по умолчанию. Проверить: `tune2fs -l /dev/sda1 | grep "Default mount"` или `mount | grep acl`.
---
### Шпаргалка: диагностика за 2 минуты
```bash
# 1. Под кем работает процесс?
ps aux | grep <service>

# 2. Кто владеет файлом и какие права?
ls -la /path/to/file
stat /path/to/file

# 3. Где в цепочке пути обрыв?
namei -l /full/path/to/file

# 4. Есть ли ACL?
getfacl /path/to/file

# 5. SELinux/AppArmor мешает?
dmesg | grep -i "avc\|apparmor" | tail -10

# 6. Быстрая проверка — может ли пользователь добраться до файла
sudo -u www-data cat /path/to/file
sudo -u www-data ls /path/to/dir
```