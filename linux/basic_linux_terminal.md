# Базовые команды для управления и администрирования ОС через терминал

---
## Навигация и файлы
### pwd - текущая директория
```bash
pwd          # текущая директория
pwd -L       # с symlink (по умолчанию)
pwd -P       # реальный физический путь
```
### cd - перейти
```bash
cd /etc          # абсолютный путь
cd nginx         # относительный путь
cd ..            # уровень выше
cd ../..         # два уровня выше
cd ~             # домашняя директория
cd -             # предыдущая директория
cd /             # корень
```
### ls - просмотр содержимого
```bash
ls -l            # длинный формат
ls -a            # показать скрытые файлы
ls -la           # длинный + скрытые
ls -lh           # размеры в KB/MB/GB
ls -lt           # сортировка по времени (новые сверху)
ls -ltr          # сортировка по времени (старые сверху)
ls -lS           # сортировка по размеру
ls -R            # рекурсивно
ls -d */         # только директории
ls -i            # показать inode
ls -1            # по одному файлу на строку
ls --color=auto  # цветной вывод
```
### mkdir - создать каталог
```bash
mkdir mydir          # создать директорию
mkdir -p a/b/c       # создать вложенные (не ругаться если существуют)
mkdir -m 755 mydir   # создать с заданными правами
mkdir -v mydir       # verbose
```
### touch - создать файл
```bash
touch file.txt           # создать файл / обновить timestamp
touch -a file.txt        # обновить только atime
touch -m file.txt        # обновить только mtime
touch -t 202401011200 f  # установить конкретное время (YYYYMMDDhhmm)
touch -r ref.txt f.txt   # скопировать timestamp с ref.txt
```
### cp - копирование
```bash
cp file.txt /tmp/            # скопировать файл
cp -r dir/ /tmp/             # рекурсивно (для директорий)
cp -p file.txt /tmp/         # сохранить права, владельца, timestamp
cp -a dir/ /tmp/             # архивный режим (= -rp + symlinks)
cp -i file.txt /tmp/         # спросить перед перезаписью
cp -n file.txt /tmp/         # не перезаписывать существующие
cp -u file.txt /tmp/         # только если источник новее
cp -v file.txt /tmp/         # verbose
cp -l file.txt /tmp/         # жёсткая ссылка вместо копии
cp -s file.txt /tmp/         # символическая ссылка вместо копии
cp --backup=numbered f /tmp/ # нумерованный бэкап при перезаписи
```
### mv - перемещение
```bash
mv file.txt /tmp/        # переместить
mv old.txt new.txt       # переименовать
mv -i file.txt /tmp/     # спросить перед перезаписью
mv -n file.txt /tmp/     # не перезаписывать существующие
mv -u file.txt /tmp/     # только если источник новее
mv -v file.txt /tmp/     # verbose
mv -b file.txt /tmp/     # бэкап перезаписываемого файла
```
### rm - удаление
```bash
rm file.txt          # удалить файл
rm -r dir/           # рекурсивно
rm -f file.txt       # принудительно (без подтверждения)
rm -rf dir/          # рекурсивно + принудительно
rm -i file.txt       # спросить перед каждым удалением
rm -v file.txt       # verbose
rm -d emptydir/      # удалить пустую директорию
```
> Всегда проверяй путь перед `-rf`. `rm -rf /` уничтожит систему.
### ln - ссылки
```bash
ln file.txt hard.txt         # жёсткая ссылка
ln -s /etc/nginx symlink     # символическая ссылка
ln -sf target link           # принудительно заменить существующую
ln -v file.txt link.txt      # verbose
```
## Текст и редакторы
### cat - вывод содержимого файла
```bash
cat file.txt             # вывести содержимое
cat -n file.txt          # с номерами строк
cat -A file.txt          # показать спецсимволы
cat -s file.txt          # убрать повторяющиеся пустые строки
cat file1 file2 > out    # объединить файлы
cat >> file.txt          # дописать с клавиатуры (Ctrl+D — завершить)
```
### less - вывод содержимого постранично
```bash
less file.txt            # открыть файл
less +F file.txt         # следить за файлом (Ctrl+C — выйти из режима)
less -N file.txt         # с номерами строк
less -S file.txt         # не переносить длинные строки
less -i file.txt         # поиск без учёта регистра
less -p "ERROR" file.txt # открыть и найти ERROR
```
> `j/k` строки · `d/u` полстраницы · `/pattern` поиск · `q` выход
### head / tail - вывод содержимого начало / конец
```bash
head file.txt            # первые 10 строк
head -n 20 file.txt      # первые N строк
head -c 100 file.txt     # первые N байт

tail file.txt            # последние 10 строк
tail -n 50 file.txt      # последние N строк
tail -c 200 file.txt     # последние N байт
tail -f file.txt         # следить в реальном времени
tail -F file.txt         # следить + переоткрывать при ротации
tail -n +5 file.txt      # вывести начиная с 5-й строки
```
### grep - поиск внутри файла
```bash
grep "ERROR" file.txt         # найти строки
grep -i "error" file.txt      # без учёта регистра
grep -v "INFO" file.txt       # инвертировать (всё кроме)
grep -c "ERROR" file.txt      # количество совпадений
grep -n "ERROR" file.txt      # с номерами строк
grep -l "ERROR" /var/log/*    # только имена файлов
grep -L "ERROR" /var/log/*    # файлы БЕЗ совпадений
grep -r "ERROR" /var/log/     # рекурсивно
grep -A 3 "ERROR" file.txt    # 3 строки после
grep -B 2 "ERROR" file.txt    # 2 строки до
grep -C 3 "ERROR" file.txt    # 3 строки до и после
grep -w "error" file.txt      # только целые слова
grep -e "ERROR" -e "WARN" f   # несколько паттернов
grep -E "ERROR|WARN" file.txt # расширенные regex
grep -P "\d{4}" file.txt      # Perl-совместимые regex
grep -o "ERROR.*" file.txt    # только совпавшая часть
grep -m 10 "ERROR" file.txt   # остановиться после N совпадений
grep -q "ERROR" file.txt      # тихий режим (exit 0/1 для скриптов)
grep --include="*.log" -r "E" # только в .log файлах
grep --line-buffered "E" f    # построчная буферизация (для pipe с tail -f)
```
### wc - счетчик
```bash
wc file.txt      # строки, слова, байты
wc -l file.txt   # строки
wc -w file.txt   # слова
wc -c file.txt   # байты
wc -m file.txt   # символы (UTF-8)
wc -L file.txt   # длина самой длинной строки
```
### sort - сортировка
```bash
sort file.txt            # алфавитная сортировка
sort -r file.txt         # обратный порядок
sort -n file.txt         # числовая сортировка
sort -k 2 file.txt       # по 2-му полю
sort -t: -k 3 -n f       # разделитель : поле 3
sort -u file.txt         # убрать дубликаты
sort -h file.txt         # с учётом суффиксов (1K, 2M)
sort -R file.txt         # случайный порядок
```
### uniq - фильтр дублей
```bash
uniq file.txt            # убрать соседние дубликаты (нужен sort)
uniq -c file.txt         # посчитать повторения
uniq -d file.txt         # только дублирующиеся строки
uniq -u file.txt         # только уникальные строки
uniq -i file.txt         # без учёта регистра
sort file.txt | uniq -c | sort -rn   # топ повторяющихся строк
```
### sed - поиск и замена
```bash
sed 's/old/new/' file.txt          # заменить первое вхождение
sed 's/old/new/g' file.txt         # заменить все вхождения
sed -i 's/old/new/g' file.txt      # in-place
sed -i.bak 's/old/new/g' file.txt  # in-place с бэкапом
sed -n '5,10p' file.txt            # вывести строки 5-10
sed '5,10d' file.txt               # удалить строки 5-10
sed '/ERROR/d' file.txt            # удалить строки с паттерном
sed '/ERROR/!d' file.txt           # оставить только с паттерном
sed 's/^/prefix: /' file.txt       # добавить префикс каждой строке
sed -n '/START/,/END/p' file.txt   # блок между паттернами
```
### awk - работа с таблицами
```bash
awk '{print $1}' file.txt          # первое поле
awk '{print $1, $3}' file.txt      # первое и третье
awk -F: '{print $1}' /etc/passwd   # разделитель :
awk '{print NR, $0}' file.txt      # с номерами строк
awk 'NR==5' file.txt               # 5-я строка
awk 'NR>=5 && NR<=10' file.txt     # строки 5-10
awk '/ERROR/ {print}' file.txt     # строки с паттерном
awk '{sum+=$1} END {print sum}'    # сумма первого столбца
awk '{print $NF}' file.txt         # последнее поле
```
### diff - сравнение
```bash
diff file1 file2         # построчное сравнение
diff -u file1 file2      # unified формат
diff -i file1 file2      # без учёта регистра
diff -b file1 file2      # игнорировать пробелы
diff -r dir1/ dir2/      # рекурсивно
diff -q file1 file2      # только факт отличия
```
### nano - простой редактор
```bash
nano file.txt        # открыть файл
nano -l file.txt     # с номерами строк
nano -v file.txt     # только для чтения
nano -B file.txt     # бэкап при сохранении
nano +15 file.txt    # открыть на 15-й строке
```
> `Ctrl+O` сохранить · `Ctrl+X` выйти · `Ctrl+W` поиск · `Ctrl+K` вырезать строку
### vim - сложный редактор
```bash
vim file.txt         # открыть файл
vim +15 file.txt     # открыть на 15-й строке
vim +/ERROR file.txt # открыть и найти ERROR
vim -R file.txt      # только для чтения
vim -d file1 file2   # diff режим
```
> `i` insert · `Esc` normal · `:w` сохранить · `:q` выйти · `:wq` сохранить и выйти · `:q!` без сохранения
> `gg` начало · `G` конец · `/pattern` поиск · `n/N` следующий/предыдущий
## Системное администрирование
### df / du / free - память storage /RAM
```bash
df -h                    # свободное место (human-readable)
df -T                    # с типом файловой системы
df -i                    # inodes
df -x tmpfs              # исключить тип ФС

du -sh /var/log          # размер директории
du -sh /var/log/*        # размер каждого элемента
du -h --max-depth=1 /    # один уровень вглубь
du -a /etc               # все файлы
du -c /var/log/*.log     # с итого
du -sh * | sort -rh | head -10   # топ-10 по размеру

free -h                  # human-readable
free -m                  # в мегабайтах
free -s 2                # обновлять каждые 2 сек
```
### uname / hostname / uptime - что за система / имя хоста / как долго работаешь
```bash
uname -a                 # всё (ядро, версия, архитектура)
uname -r                 # версия ядра
uname -m                 # архитектура
uname -n                 # hostname

hostname                 # имя хоста
hostname -f              # FQDN
hostname -I              # все IP адреса
hostnamectl              # расширенная информация
hostnamectl set-hostname newname

uptime                   # время работы, load average
uptime -p                # human-readable
uptime -s                # время запуска
```
## Архивы и пакеты
### tar
```bash
tar -czf archive.tar.gz dir/      # gzip архив
tar -cjf archive.tar.bz2 dir/     # bzip2
tar -cJf archive.tar.xz dir/      # xz (меньше, медленнее)
tar -cf archive.tar dir/          # без сжатия
tar -xzf archive.tar.gz           # распаковать gzip
tar -xjf archive.tar.bz2          # распаковать bzip2
tar -xf archive.tar.gz -C /tmp/   # распаковать в директорию
tar -tzf archive.tar.gz           # список файлов без распаковки
tar -czf archive.tar.gz -v dir/   # verbose
tar --exclude=".git" -czf a.tar.gz project/
```
> `-c` создать · `-x` распаковать · `-t` список · `-z` gzip · `-j` bzip2 · `-J` xz · `-f` имя файла · `-C` куда
### zip / unzip
```bash
zip archive.zip file1 file2   # создать
zip -r archive.zip dir/       # рекурсивно
zip -9 archive.zip file       # максимальное сжатие
zip -e archive.zip file       # с паролем
zip -u archive.zip newfile    # добавить/обновить файл
unzip archive.zip             # распаковать
unzip archive.zip -d /tmp/    # в директорию
unzip -l archive.zip          # список файлов
```
## Pipe, перенаправление, утилиты
### Перенаправление
```bash
command > file.txt       # stdout в файл (перезапись)
command >> file.txt      # stdout в файл (добавление)
command 2> err.txt       # stderr в файл
command 2>&1             # stderr → stdout
command > out.txt 2>&1   # stdout и stderr в файл
command &> file.txt      # то же (bash shorthand)
command < input.txt      # stdin из файла
command > /dev/null 2>&1 # выбросить всё
```
### Комбинирование
```bash
cmd1 | cmd2          # stdout cmd1 → stdin cmd2
cmd1 && cmd2         # cmd2 только если cmd1 успешен
cmd1 || cmd2         # cmd2 только если cmd1 провалился
cmd1 ; cmd2          # оба, независимо от результата
command &            # запустить в фоне
```
