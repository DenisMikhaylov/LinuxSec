# Лабораторная работа: Управление журналами Linux (хранение, ротация, архивирование, защита от изменений)

**Дисциплина:** Защита информации в операционных системах  
**Тема:** Управление журналами Linux  
**Среда:** Debian 12 (Bookworm)  
**Продолжительность:** 2 академических часа (90 минут)  
**Количество виртуальных машин:** 1 (Server)  
**Учётные записи:** `debian` / `root`, пароль: `debian`

---

## Цель работы

Освоить механизмы управления журналами в Debian Linux: настройку ротации через `logrotate`, ограничение хранения журналов `journald`, архивирование и резервное копирование, ручной поиск информации в журналах, а также защиту журналов от несанкционированного изменения и удаления.

**Формат работы:** каждое действие — одна команда. Вводите команды по порядку, фиксируйте результат в отчёт.

---

## ЭТАП 1. Подготовка и обзор журналов

### Шаг 1.1. Вход в систему

Откройте терминал и войдите под пользователем `debian`. Затем перейдите в режим суперпользователя:

```bash
su - root
```

**Введите пароль:** `debian`

**Ожидаемый результат:** приглашение командной строки изменилось на `root@server:~#`.

---

### Шаг 1.2. Просмотр списка журналов

```bash
ls -la /var/log/
```

**Ожидаемый результат:** список файлов и каталогов, включая `syslog`, `auth.log`, `kern.log`, `dpkg.log`, `journal/`, `remote/`.

**Зафиксируйте вывод в отчёт.**

---

### Шаг 1.3. Просмотр содержимого ключевых журналов

```bash
tail -20 /var/log/syslog
```

**Ожидаемый результат:** последние 20 строк системного журнала.

```bash
tail -20 /var/log/auth.log
```

**Ожидаемый результат:** последние 20 строк журнала аутентификации (входы, sudo, SSH).

```bash
tail -20 /var/log/kern.log
```

**Ожидаемый результат:** последние 20 строк журнала ядра.

---

### Шаг 1.4. Просмотр прав доступа к журналам

```bash
ls -l /var/log/syslog /var/log/auth.log /var/log/kern.log
```

**Ожидаемый результат:**
```
-rw-r----- 1 root adm 123456 Oct 16 10:00 /var/log/auth.log
-rw-r----- 1 root adm 234567 Oct 16 10:00 /var/log/kern.log
-rw-r----- 1 root adm 345678 Oct 16 10:00 /var/log/syslog
```

**Зафиксируйте вывод в отчёт.** Обратите внимание: права `640` означают, что читать журналы могут только `root` и члены группы `adm`.

---

## ЭТАП 2. Ручной поиск информации в журналах

### Шаг 2.1. Поиск событий по ключевому слову

```bash
grep "Failed password" /var/log/auth.log | tail -10
```

**Ожидаемый результат:** строки с сообщениями о неудачных попытках входа по SSH (если они были).

Если строк нет, сгенерируйте событие:

```bash
ssh -o ConnectTimeout=3 debian@localhost false
```

Введите неверный пароль. Затем повторите поиск.

---

### Шаг 2.2. Подсчёт количества событий

```bash
grep -c "Failed password" /var/log/auth.log
```

**Ожидаемый результат:** число — количество неудачных попыток входа.

```bash
grep -c "sudo" /var/log/auth.log
```

**Ожидаемый результат:** число — количество событий, связанных с `sudo`.

**Зафиксируйте вывод в отчёт.**

---

### Шаг 2.3. Поиск событий за конкретную дату

```bash
grep "$(date +%Y-%m-%d)" /var/log/syslog | tail -20
```

**Ожидаемый результат:** строки журнала за текущую дату.

---

### Шаг 2.4. Извлечение IP-адресов из журнала

```bash
grep "Failed password" /var/log/auth.log | awk '{print $(NF-3)}' | sort | uniq -c | sort -rn
```

**Ожидаемый результат:** список IP-адресов с количеством неудачных попыток входа, отсортированный по убыванию.

**Зафиксируйте вывод в отчёт.**

---

### Шаг 2.5. Поиск событий с временными метками

```bash
grep -E "Oct 16 (09|10):" /var/log/syslog | head -30
```

**Ожидаемый результат:** строки журнала за 09:00–10:59 текущего дня.

---

### Шаг 2.6. Статистика по уровням важности

```bash
awk '{print $6}' /var/log/syslog | sort | uniq -c | sort -rn
```

**Ожидаемый результат:** распределение сообщений по уровню важности (например, `error`, `warning`, `info`).

**Зафиксируйте вывод в отчёт.**

---

### Шаг 2.7. Поиск в journald с фильтрацией

```bash
journalctl -p err --since "1 hour ago" --no-pager | tail -30
```

**Ожидаемый результат:** сообщения об ошибках за последний час.

```bash
journalctl -u ssh --since today --no-pager | tail -20
```

**Ожидаемый результат:** последние 20 сообщений от сервиса SSH за сегодня.

---

## ЭТАП 3. Ротация журналов через logrotate

### Шаг 3.1. Просмотр глобальной конфигурации logrotate

```bash
cat /etc/logrotate.conf
```

**Ожидаемый результат:** файл с настройками по умолчанию:
```
weekly
rotate 4
create
dateext
compress
include /etc/logrotate.d
```

---

### Шаг 3.2. Просмотр конфигурации для rsyslog

```bash
cat /etc/logrotate.d/rsyslog
```

**Ожидаемый результат:** блоки для `/var/log/syslog`, `/var/log/auth.log`, `/var/log/kern.log` и других журналов.

---

### Шаг 3.3. Создание пользовательской конфигурации ротации

```bash
nano /etc/logrotate.d/custom-app
```

Вставьте текст:

```
/var/log/custom-app.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 640 root adm
    dateext
}
```

Сохраните (`Ctrl+O`, `Enter`, `Ctrl+X`).

**Пояснение параметров:**
- `daily` — ротация каждый день;
- `rotate 7` — хранить 7 архивных копий;
- `compress` — сжимать старые копии (gzip);
- `delaycompress` — отложить сжатие на один цикл (безопасно для активных сервисов);
- `missingok` — не выдавать ошибку, если файла нет;
- `notifempty` — не ротировать пустой файл;
- `create 640 root adm` — создать новый файл с указанными правами;
- `dateext` — добавлять дату к имени архива (YYYYMMDD).

---

### Шаг 3.4. Проверка конфигурации logrotate

```bash
logrotate -d /etc/logrotate.d/custom-app
```

**Ожидаемый результат:** отладочный вывод с описанием действий, которые были бы выполнены. Ошибок быть не должно.

---

### Шаг 3.5. Создание тестового журнала и принудительная ротация

```bash
echo "test log entry $(date)" > /var/log/custom-app.log
```

```bash
logrotate -f /etc/logrotate.d/custom-app
```

**Ожидаемый результат:** команда выполнена без вывода.

```bash
ls -la /var/log/custom-app*
```

**Ожидаемый результат:** файлы `custom-app.log` (новый) и `custom-app.log-YYYYMMDD` (архив).

**Зафиксируйте вывод в отчёт.**

---

### Шаг 3.6. Проверка сжатия

Подождите одну ротацию (или выполните ещё раз):

```bash
echo "second entry" >> /var/log/custom-app.log
```

```bash
logrotate -f /etc/logrotate.d/custom-app
```

```bash
ls -la /var/log/custom-app*
```

**Ожидаемый результат:** предыдущий архив сжат в `.gz`.

---

### Шаг 3.7. Просмотр состояния logrotate

```bash
cat /var/lib/logrotate/status
```

**Ожидаемый результат:** записи о последней ротации для каждого журнала.

---

## ЭТАП 4. Настройка хранения журналов journald

### Шаг 4.1. Включение постоянного хранения

```bash
mkdir -p /var/log/journal
```

```bash
systemctl restart systemd-journald
```

**Ожидаемый результат:** команды выполнены без ошибок.

---

### Шаг 4.2. Настройка ограничений хранения

```bash
nano /etc/systemd/journald.conf
```

Найдите и измените (или добавьте в раздел `[Journal]`):

```
[Journal]
Storage=persistent
SystemMaxUse=512M
SystemKeepFree=1G
SystemMaxFileSize=50M
MaxRetentionSec=7day
Compress=yes
```

Сохраните (`Ctrl+O`, `Enter`, `Ctrl+X`).

**Пояснение параметров:**
- `Storage=persistent` — хранить журналы на диске в `/var/log/journal/`;
- `SystemMaxUse=512M` — максимальный общий размер журналов;
- `SystemKeepFree=1G` — оставлять не менее 1 ГБ свободного места;
- `SystemMaxFileSize=50M` — максимальный размер одного файла;
- `MaxRetentionSec=7day` — хранить журналы не более 7 дней;
- `Compress=yes` — сжимать данные.

---

### Шаг 4.3. Применение изменений

```bash
systemctl restart systemd-journald
```

```bash
journalctl --disk-usage
```

**Ожидаемый результат:** `Archived and active journals take up ... in the file system.`

---

### Шаг 4.4. Проверка размера и сжатия

```bash
ls -la /var/log/journal/
```

**Ожидаемый результат:** каталог с подкаталогами, содержащими файлы `.journal` и `.journal~` (архивные).

```bash
journalctl --verify
```

**Ожидаемый результат:** `PASS` для всех файлов журнала.

**Зафиксируйте вывод в отчёт.**

---

## ЭТАП 5. Архивирование и резервное копирование журналов

### Шаг 5.1. Создание каталога для архивов

```bash
mkdir -p /backup/logs
```

**Ожидаемый результат:** команда выполнена без вывода.

---

### Шаг 5.2. Создание tar-архива журналов

```bash
tar -czvf /backup/logs/varlog-$(date +%F).tar.gz /var/log/
```

**Ожидаемый результат:** список файлов, добавляемых в архив, и итоговый размер.

**Зафиксируйте вывод в отчёт.**

---

### Шаг 5.3. Проверка архива

```bash
tar -tzvf /backup/logs/varlog-$(date +%F).tar.gz | head -20
```

**Ожидаемый результат:** список первых 20 файлов в архиве.

---

### Шаг 5.4. Создание инкрементного архива через rsync

```bash
rsync -a --delete /var/log/ /backup/logs/current/
```

**Ожидаемый результат:** команда выполнена без вывода (или с минимальным выводом).

```bash
ls -la /backup/logs/current/
```

**Ожидаемый результат:** зеркальная копия `/var/log/`.

**Зафиксируйте вывод в отчёт.**

---

### Шаг 5.5. Проверка целостности архива

```bash
gzip -t /backup/logs/varlog-$(date +%F).tar.gz && echo "Archive OK"
```

**Ожидаемый результат:** `Archive OK`.

---

## ЭТАП 6. Защита журналов от изменений

### Шаг 6.1. Установка атрибута append-only

```bash
chattr +a /var/log/auth.log
```

**Ожидаемый результат:** команда выполнена без вывода.

```bash
lsattr /var/log/auth.log
```

**Ожидаемый результат:** в выводе присутствует флаг `a` (например, `-----a--------e--`).

---

### Шаг 6.2. Проверка защиты append-only

Попробуйте удалить журнал:

```bash
rm /var/log/auth.log
```

**Ожидаемый результат:** `rm: cannot remove '/var/log/auth.log': Operation not permitted`.

Попробуйте перезаписать журнал:

```bash
echo "test" > /var/log/auth.log
```

**Ожидаемый результат:** `bash: /var/log/auth.log: Operation not permitted`.

**Зафиксируйте вывод в отчёт.**

---

### Шаг 6.3. Установка атрибута immutable

```bash
chattr +i /var/log/syslog
```

```bash
lsattr /var/log/syslog
```

**Ожидаемый результат:** в выводе присутствует флаг `i`.

---

### Шаг 6.4. Проверка защиты immutable

Попробуйте изменить файл:

```bash
echo "test" >> /var/log/syslog
```

**Ожидаемый результат:** `bash: /var/log/syslog: Operation not permitted`.

Попробуйте переименовать:

```bash
mv /var/log/syslog /var/log/syslog.bak
```

**Ожидаемый результат:** `mv: cannot move '/var/log/syslog': Operation not permitted`.

**Зафиксируйте вывод в отчёт.**

---

### Шаг 6.5. Снятие защиты (для продолжения работы)

```bash
chattr -a /var/log/auth.log
```

```bash
chattr -i /var/log/syslog
```

**Ожидаемый результат:** команды выполнены без вывода.

Проверьте:

```bash
lsattr /var/log/auth.log /var/log/syslog
```

**Ожидаемый результат:** флаги `a` и `i` отсутствуют.

---

### Шаг 6.6. Аудит изменений журналов через auditd

```bash
apt install -y auditd
```

```bash
systemctl enable --now auditd
```

Добавьте правило аудита:

```bash
auditctl -w /var/log/auth.log -p wa -k log_auth_changes
```

**Ожидаемый результат:** команда выполнена без вывода.

Проверьте:

```bash
auditctl -l | grep log_auth_changes
```

**Ожидаемый результат:** строка с правилом.

---

### Шаг 6.7. Генерация события и поиск

```bash
logger -p auth.notice "Test audit event"
```

```bash
ausearch -k log_auth_changes -ts recent
```

**Ожидаемый результат:** блок с событием, содержащим `key="log_auth_changes"`.

**Зафиксируйте вывод в отчёт.**

---

### Шаг 6.8. Проверка целостности через хеширование

```bash
md5sum /var/log/auth.log /var/log/syslog > /backup/logs/logs.md5
```

```bash
cat /backup/logs/logs.md5
```

**Ожидаемый результат:** две строки с хешами.

**Зафиксируйте вывод в отчёт.** Позже можно проверить целостность командой:

```bash
md5sum -c /backup/logs/logs.md5
```

**Ожидаемый результат:** `OK` для каждого файла.

---



## Рекомендуемые источники

1. `man logrotate`, `man logrotate.conf`
2. `man journald.conf`, `man journalctl`
3. `man chattr`, `man lsattr`
4. `man rsync`, `man tar`
5. `man auditctl`, `man ausearch`
6. CIS Debian Linux 12 Benchmark — разделы по логированию и аудиту
