# Файл менеджент & Анализ логов

---
## 1. Поиск файлов
### Найти файл по имени
```bash
find /var/log -name "*.log"
find /etc -name "nginx.conf"
```
### Найти файлы старше N дней (например, для очистки)
```bash
find /var/log -name "*.log" -mtime +30
```
### Найти и удалить одной командой
```bash
find /var/log -name "*.log" -mtime +30 -delete
```
> Сначала запускаем без `-delete` — нужно убедится, что список файлов правильный.
### Найти тяжёлые файлы (больше 100MB)
```bash
find / -size +100M -type f 2>/dev/null
```
> `2>/dev/null` — подавляет ошибки доступа к системным директориям.
---
## 2. Анализ логов
### Базовая фильтрация
```bash
grep "ERROR" /var/log/app.log
grep -i "error" /var/log/app.log        # регистронезависимо
grep -v "INFO" /var/log/app.log         # всё кроме INFO
```
### Контекст вокруг ошибки (критично при диагностике)
```bash
grep -A 5 -B 2 "ERROR" /var/log/app.log
# -A 5 = 5 строк после, -B 2 = 2 строки до
```
> В production ошибка редко стоит одна — контекст показывает что было до и после.
### Реалтайм мониторинг
```bash
tail -f /var/log/app.log
tail -f /var/log/app.log | grep --line-buffered "ERROR\|WARN"
```
> `--line-buffered` обязателен при pipe с `grep` — без него вывод буферизуется и ты не видишь новые строки сразу.
### Подсчёт ошибок (оценка масштаба инцидента)
```bash
grep -c "ERROR" /var/log/app.log
grep "ERROR" /var/log/app.log | wc -l
```
### Поиск по нескольким файлам
```bash
grep -r "connection refused" /var/log/
grep -r -l "ERROR" /var/log/           # только имена файлов, без содержимого
```
---
## 3. Работа с содержимым файлов
### Просмотр без открытия редактора
```bash
cat /etc/nginx/nginx.conf              # весь файл
head -n 20 /var/log/app.log            # первые 20 строк
tail -n 50 /var/log/app.log            # последние 50 строк
```
### Сравнение файлов конфигурации
```bash
diff /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```
> Полезно после изменений — быстро видишь что именно поменялось.
---
## 4. Дисковое пространство
Где заканчивается место
```bash
df -h                                  # свободное место по разделам
du -sh /var/log/*                      # размер каждого лога
du -sh /* 2>/dev/null | sort -rh | head -10   # топ-10 тяжёлых директорий
```
### Ротация логов вручную (если logrotate не настроен)
```bash
> /var/log/app.log                     # очистить файл без удаления (процесс продолжает писать)
# НЕ делай: rm /var/log/app.log        # процесс будет писать в удалённый файл, место не освободится
```
---
## 5. Права доступа
### Быстрая проверка
```bash
ls -la /etc/nginx/
stat /etc/passwd
```
### Изменить владельца рекурсивно
```bash
chown -R www-data:www-data /var/www/html
```
### Безопасные права для конфигов с секретами
```bash
chmod 600 /etc/app/secrets.env         # только владелец читает и пишет
chmod 640 /etc/nginx/nginx.conf        # владелец rw, группа r, остальные ничего
```
---
