# Лабораторная работа: Мониторинг событий безопасности Linux

**Дисциплина:** Защита информации в операционных системах  
**Тема:** Мониторинг событий безопасности Linux  
**Среда:** Debian 12 (Bookworm)  
**Продолжительность:** 2 академических часа (90 минут)  
**Количество виртуальных машин:** 2 (Gate, Server)  
**Учётные записи:** `debian` / `root`, пароль: `debian`

---

## Цель работы

Освоить настройку подсистемы аудита `auditd`, централизованного сбора журналов через `rsyslog`, средств активного противодействия `fail2ban` и инструментов анализа журналов. Научиться выявлять инциденты безопасности по данным журналов.

**Формат работы:** каждое действие — одна команда. Вводите команды по порядку, фиксируйте результат в отчёт.

---

## ЭТАП 1. Подготовка и определение IP-адресов

### Шаг 1.1. Вход в систему (на обеих машинах Gate и Server)

Откройте терминал и войдите под пользователем `debian`. Затем перейдите в режим суперпользователя:

```bash
su - root
```

**Введите пароль:** `debian`

**Ожидаемый результат:** приглашение командной строки изменилось на `root@имя_машины:~#`.

---

### Шаг 1.2. Определение IP-адреса на Gate

```bash
ip -4 addr show | grep "inet " | grep -v "127.0.0.1"
```

**Ожидаемый результат:** строка вида:
```
inet 192.168.56.10/24 brd 192.168.56.255 scope global enp0s8
```

**Запишите в отчёт значение IP-адреса Gate (без маски `/24`).** Обозначим его `<GATE_IP>`.

---

### Шаг 1.3. Определение IP-адреса на Server

На второй виртуальной машине выполните ту же команду:

```bash
ip -4 addr show | grep "inet " | grep -v "127.0.0.1"
```

**Ожидаемый результат:** строка вида:
```
inet 192.168.56.20/24 brd 192.168.56.255 scope global enp0s8
```

**Запишите в отчёт значение IP-адреса Server (без маски `/24`).** Обозначим его `<SERVER_IP>`.

---

### Шаг 1.4. Сохранение IP-адресов в переменные окружения

Выполните на **Gate** (подставьте своё значение вместо примера):

```bash
export GATE_IP=192.168.56.10
export SERVER_IP=192.168.56.20
```

Выполните на **Server** (подставьте своё значение вместо примера):

```bash
export GATE_IP=192.168.56.10
export SERVER_IP=192.168.56.20
```

**Ожидаемый результат:** команды выполнены без сообщений об ошибках.

---

### Шаг 1.5. Проверка связности

На **Gate** выполните:

```bash
ping -c 2 $SERVER_IP
```

**Ожидаемый результат:**
```
2 packets transmitted, 2 received, 0% packet loss
```

На **Server** выполните:

```bash
ping -c 2 $GATE_IP
```

**Ожидаемый результат:** то же самое, `0% packet loss`.

Если пакеты не проходят — проверьте, что обе машины в одной подсети и подключены к одному виртуальному коммутатору.

---

## ЭТАП 2. Настройка auditd на Server

### Шаг 2.1. Установка auditd

Выполните на **Server**:

```bash
apt update
```

**Ожидаемый результат:** завершается без ошибок.

```bash
apt install -y auditd audispd-plugins
```

**Ожидаемый результат:** пакеты установлены, в конце — `Setting up auditd ...`.

---

### Шаг 2.2. Запуск и включение автозагрузки

```bash
systemctl enable --now auditd
```

**Ожидаемый результат:** `Created symlink ...` или отсутствие вывода.

Проверьте статус:

```bash
systemctl status auditd
```

**Ожидаемый результат:** строка `Active: active (running)`.

---

### Шаг 2.3. Проверка состояния подсистемы аудита

```bash
auditctl -s
```

**Ожидаемый результат:** вывод вида:
```
enabled 1
failure 1
pid 1234
rate_limit 0
backlog_limit 8192
lost 0
backlog 0
```

---

### Шаг 2.4. Создание правил аудита

```bash
nano /etc/audit/rules.d/10-security-baseline.rules
```

Вставьте в файл следующий текст:

```
-D
-b 8192
-f 1
-w /etc/passwd -p wa -k identity_passwd
-w /etc/shadow -p wa -k identity_shadow
-w /etc/group -p wa -k identity_group
-w /etc/gshadow -p wa -k identity_gshadow
-w /etc/sudoers -p wa -k sudoers_changes
-w /etc/ssh/sshd_config -p wa -k ssh_config
-w /usr/bin/passwd -p x -k passwd_exec
-w /usr/bin/sudo -p x -k sudo_exec
-w /bin/su -p x -k su_exec
-w /etc/audit/ -p wa -k audit_config
-a always,exit -F arch=b64 -S execve -F euid!=0 -k user_exec
```

Сохраните файл: `Ctrl+O`, `Enter`, затем `Ctrl+X`.

---

### Шаг 2.5. Загрузка правил

```bash
augenrules --load
```

**Ожидаемый результат:** вывод строк вида `No rules ...` или сообщения о загрузке.

```bash
augenrules --check
```

**Ожидаемый результат:** `No change` или `Rules loaded`.

```bash
auditctl -l
```

**Ожидаемый результат:** список всех загруженных правил (12+ строк, начинающихся с `-w` или `-a`).

---

### Шаг 2.6. Генерация события безопасности

```bash
echo "# test comment" >> /etc/passwd
```

**Ожидаемый результат:** команда выполнена без вывода.

```bash
sudo ls /root
```

**Ожидаемый результат:** ввод пароля `debian` и список файлов (пустой, если `/root` пуст).

---

### Шаг 2.7. Поиск события изменения /etc/passwd

```bash
ausearch -k identity_passwd -ts recent
```

**Ожидаемый результат:** блок вида:
```
time->... type=SYSCALL msg=audit(...): arch=c000003e syscall=257 ... comm="bash" ... key="identity_passwd"
```

---

### Шаг 2.8. Поиск события выполнения sudo

```bash
ausearch -k sudo_exec -ts recent
```

**Ожидаемый результат:** блок с `type=EXECVE ... a0="sudo" ... key="sudo_exec"`.

---

### Шаг 2.9. Сводный отчёт по ключам аудита

```bash
aureport -k --summary
```

**Ожидаемый результат:** таблица с колонками `Key` и `Count`, где для ключей `identity_passwd` и `sudo_exec` указаны ненулевые значения.

**Зафиксируйте вывод в отчёт.**

---

## ЭТАП 3. Настройка централизованного сбора журналов

### Шаг 3.1. Установка rsyslog на Gate

Выполните на **Gate**:

```bash
apt install -y rsyslog
```

```bash
systemctl enable --now rsyslog
```

**Ожидаемый результат:** пакет установлен, сервис активен.

---

### Шаг 3.2. Настройка приёма журналов на Gate

```bash
nano /etc/rsyslog.d/10-remote-input.conf
```

Вставьте текст:

```
module(load="imudp")
input(type="imudp" port="514")
module(load="imtcp")
input(type="imtcp" port="514")
$template RemoteLogs,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?RemoteLogs
& stop
```

Сохраните (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### Шаг 3.3. Создание каталога и перезапуск rsyslog на Gate

```bash
mkdir -p /var/log/remote
```

```bash
chown syslog:adm /var/log/remote
```

```bash
systemctl restart rsyslog
```

**Ожидаемый результат:** команды выполнены без ошибок.

Проверьте:

```bash
systemctl status rsyslog
```

**Ожидаемый результат:** `Active: active (running)`.

---

### Шаг 3.4. Открытие порта 514 на Gate

```bash
iptables -A INPUT -p udp --dport 514 -j ACCEPT
```

```bash
iptables -A INPUT -p tcp --dport 514 -j ACCEPT
```

**Ожидаемый результат:** команды выполнены без вывода.

---

### Шаг 3.5. Настройка отправки журналов с Server

На **Server** создайте файл:

```bash
nano /etc/rsyslog.d/10-remote-output.conf
```

Вставьте строку (замените `192.168.56.10` на ваш `<GATE_IP>`):

```
*.* @@192.168.56.10:514
```

Сохраните (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### Шаг 3.6. Перезапуск rsyslog на Server

```bash
systemctl restart rsyslog
```

**Ожидаемый результат:** команда выполнена без ошибок.

---

### Шаг 3.7. Генерация тестового события на Server

```bash
logger -p auth.notice "Test security event from Server"
```

**Ожидаемый результат:** команда выполнена без вывода.

---

### Шаг 3.8. Проверка получения журнала на Gate

На **Gate** сначала узнайте имя хоста Server:

```bash
SERVER_HOSTNAME=$(ssh debian@$SERVER_IP hostname)
echo $SERVER_HOSTNAME
```

**Ожидаемый результат:** имя хоста Server (например, `server`).

Проверьте получение журналов:

```bash
ls -la /var/log/remote/$SERVER_HOSTNAME/
```

**Ожидаемый результат:** файлы вида `rsyslog.log`, `sudo.log`, `sshd.log` и др.

```bash
cat /var/log/remote/$SERVER_HOSTNAME/*
```

**Ожидаемый результат:** среди прочих строк присутствует `Test security event from Server`.

**Зафиксируйте вывод в отчёт.**

---

## ЭТАП 4. Настройка fail2ban на Server

### Шаг 4.1. Установка fail2ban

Выполните на **Server**:

```bash
apt install -y fail2ban
```

```bash
systemctl enable --now fail2ban
```

**Ожидаемый результат:** пакет установлен, сервис активен.

---

### Шаг 4.2. Создание конфигурации jail

```bash
nano /etc/fail2ban/jail.local
```

Вставьте текст:

```
[DEFAULT]
bantime = 300
findtime = 60
maxretry = 3
backend = systemd

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 600
```

Сохраните (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### Шаг 4.3. Перезапуск fail2ban

```bash
systemctl restart fail2ban
```

```bash
fail2ban-client status
```

**Ожидаемый результат:** список jail-ов, включая `sshd`.

```bash
fail2ban-client status sshd
```

**Ожидаемый результат:**
```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed: 0
|- Actions
   |- Currently banned: 0
   |- Total banned: 0
   |- Banned IP list:
```

**Зафиксируйте вывод в отчёт.**

---

## ЭТАП 5. Имитация атаки и обнаружение

### Шаг 5.1. Имитация перебора паролей с Gate

На **Gate** выполните три команды подряд (каждая — неудачная попытка входа):

```bash
ssh -o ConnectTimeout=3 debian@$SERVER_IP false
```

```bash
ssh -o ConnectTimeout=3 debian@$SERVER_IP false
```

```bash
ssh -o ConnectTimeout=3 debian@$SERVER_IP false
```

При каждой команде введите **неверный** пароль (например, `wrongpass`).

**Ожидаемый результат:** `Permission denied` в выводе каждой команды.

---

### Шаг 5.2. Проверка блокировки на Server

На **Server** выполните:

```bash
fail2ban-client status sshd
```

**Ожидаемый результат:** в строке `Banned IP list:` появился IP-адрес Gate.

```
|- Currently banned: 1
|- Total banned: 1
|- Banned IP list: 192.168.56.10
```

**Зафиксируйте вывод в отчёт.**

---

### Шаг 5.3. Просмотр журнала fail2ban

```bash
tail -20 /var/log/fail2ban.log
```

**Ожидаемый результат:** строки вида:
```
NOTICE [sshd] Ban 192.168.56.10
```

---

### Шаг 5.4. Просмотр журналов аудита по попыткам входа

```bash
ausearch -m user_auth -ts recent
```

**Ожидаемый результат:** блоки с событиями `user_auth`.

---

### Шаг 5.5. Разблокировка Gate на Server

```bash
fail2ban-client set sshd unbanip $GATE_IP
```

**Ожидаемый результат:** `1` (число разблокированных IP).

Проверьте:

```bash
fail2ban-client status sshd
```

**Ожидаемый результат:** `Banned IP list:` пуст.

---

## ЭТАП 6. Настройка journald и logwatch

### Шаг 6.1. Настройка journald на Server

```bash
nano /etc/systemd/journald.conf
```

Найдите и измените (или добавьте в раздел `[Journal]`):

```
Storage=persistent
SystemMaxUse=512M
SystemKeepFree=1G
SystemMaxFileSize=50M
MaxRetentionSec=7day
Compress=yes
```

Сохраните (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### Шаг 6.2. Создание каталога и перезапуск journald

```bash
mkdir -p /var/log/journal
```

```bash
systemctl restart systemd-journald
```

```bash
journalctl --disk-usage
```

**Ожидаемый результат:** `Archived and active journals take up ... in the file system.`

---

### Шаг 6.3. Установка logwatch

```bash
apt install -y logwatch
```

**Ожидаемый результат:** пакет установлен.

---

### Шаг 6.4. Настройка logwatch

```bash
nano /etc/logwatch/conf/logwatch.conf
```

Вставьте текст:

```
LogDir = /var/log
TmpDir = /var/cache/logwatch
Output = stdout
Format = text
Encode = none
Range = today
Detail = Med
Service = All
```

Сохраните (`Ctrl+O`, `Enter`, `Ctrl+X`).

---

### Шаг 6.5. Генерация отчёта logwatch

```bash
logwatch --output stdout --range today --detail Med | head -60
```

**Ожидаемый результат:** текстовый отчёт с разделами `sshd`, `sudo`, `su` и т.д.

**Зафиксируйте первые 20–30 строк в отчёт.**

---




## Рекомендуемые источники

1. `man auditd`, `man auditctl`, `man ausearch`, `man aureport`
2. `man rsyslog.conf`
3. `man fail2ban`, `man jail.conf`
4. `man journalctl`
5. `man logwatch`
