# Конфигурация виртуальной машины devops-vm

Окружение: macOS (MacBook Air M4, Apple Silicon), виртуализация UTM (QEMU, aarch64),
гостевая ОС Ubuntu Server 26.04.1 LTS (arm64). Методичка рассчитана на VirtualBox и
Ubuntu 24.04, отличия указаны ниже.

## 1. Параметры машины

- Имя ВМ в UTM: `Linux`; имя узла (hostname): `devops-vm`
- Оперативная память: 2048 МБ
- Процессор: 2 ядра
- Диск: 25 ГБ (динамический), разметка Use an entire disk (LVM);
  под `/` выделено около 11 ГБ, остальное свободно в группе `ubuntu-vg`
- Образ установки: `ubuntu-26.04.1-live-server-arm64.iso`; тип установки Ubuntu Server
  (не minimized); OpenSSH server отмечен; Featured snaps не выбирались
- После установки ISO отключён в настройках CD/DVD
- Установка обновлений безопасности в установщике зависала, остановлена вручную;
  система обновлена после первого запуска: `sudo apt update && sudo apt upgrade -y`

## 2. Сетевые интерфейсы

| Интерфейс | Режим UTM | Адрес | Назначение |
|---|---|---|---|
| enp0s1 | Эмулируемый VLAN (аналог NAT), virtio-net-pci | 10.0.2.15/24 | доступ в Интернет, проброс порта |
| enp0s2 | Только хост (Host Only), Default (private), virtio-net-pci | 192.168.64.3/24 | доступ с Mac напрямую |

Примечание: подсеть Host-only в UTM может смениться после перезапуска ВМ
(сначала был 192.168.128.3, затем 192.168.64.3). Актуальный адрес проверять командой
`ip -brief address`. Для второго адаптера MAC-адрес должен отличаться от первого
(кнопка «Случайный»).

## 3. Правило проброса портов

Настраивается в UTM: Сеть (первый адаптер) → Перенаправление портов. Правила нельзя
редактировать, только удалить и создать заново. Менять только при выключенной ВМ.

| Протокол | Адрес хоста | Порт хоста | Адрес гостя | Порт гостя |
|---|---|---|---|---|
| TCP | 127.0.0.1 | 2222 | (пусто) | 2222 |

До задания 2 порт гостя был 22, после отключения ssh.socket и смены порта SSH
изменён на 2222.

## 4. Учётные записи

| Имя | Группы | Аутентификация |
|---|---|---|
| ss | sudo (создан установщиком вместо student) | пароль, только для консоли UTM и первого входа |
| devops | devops, sudo, users | только по ключу ed25519 |

Ключ создаётся на Mac:
`ssh-keygen -t ed25519 -C "devops-vm-key" -f ~/.ssh/devops_vm`,
открытый ключ копируется: `ssh-copy-id -i ~/.ssh/devops_vm.pub -p 2222 devops@127.0.0.1`.
Права: `~/.ssh` 700, `authorized_keys` 600.
Создание пользователя: `sudo adduser devops && sudo usermod -aG sudo devops`.

Файл `~/.ssh/config` на Mac:
```
Host devops
 HostName 127.0.0.1
 Port 2222
 User devops
 IdentityFile ~/.ssh/devops_vm
 IdentitiesOnly yes

Host devops.local
 HostName devops.local
 Port 2222
 User devops
 IdentityFile ~/.ssh/devops_vm
 IdentitiesOnly yes
```

## 5. Служба SSH

- Основной файл `/etc/ssh/sshd_config` не изменяется, резервная копия:
  `/etc/ssh/sshd_config.backup`
- Дополнительный файл: `/etc/ssh/sshd_config.d/99-hardening.conf`:
```
Port 2222
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
LoginGraceTime 30
AllowUsers devops
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```
- Конфликт: в `/etc/ssh/sshd_config.d/50-cloud-init.conf` была строка
  `PasswordAuthentication yes` (файл читается раньше 99-, первое значение побеждает).
  Строка закомментирована:
  `sudo sed -i 's/^PasswordAuthentication yes/#PasswordAuthentication yes/' /etc/ssh/sshd_config.d/50-cloud-init.conf`
- Проверка: `sudo sshd -t` и `sudo sshd -T | grep -Ei "^(port|permitrootlogin|passwordauthentication|maxauthtries|allowusers)"`
- Отключена активация по сокету, иначе порт остаётся 22:
```
sudo systemctl disable --now ssh.socket
sudo systemctl enable --now ssh.service
sudo systemctl restart ssh
sudo ss -tlnp | grep sshd    # ожидается 0.0.0.0:2222
```
- После этого выключить ВМ и изменить порт гостя в проброске UTM с 22 на 2222.
- Порядок: сессию SSH не закрывать, проверять вход в отдельном терминале.

## 6. Правила межсетевого экрана (UFW)

Политики: `default deny incoming`, `default allow outgoing`.
Порядок: сначала правила, потом `enable`.
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw allow 443/tcp comment 'HTTPS'
sudo ufw enable
sudo ufw delete allow 2222/tcp
sudo ufw limit 2222/tcp comment 'SSH rate-limited'
sudo ufw logging medium
```
Итоговые правила: 2222/tcp LIMIT, 80/tcp ALLOW, 443/tcp ALLOW (IPv4 и IPv6).
Не использовать `ufw allow ssh` (это порт 22).
Журнал блокировок: для появления `/var/log/ufw.log` установлен rsyslog
(`sudo apt install rsyslog -y`, `sudo systemctl restart rsyslog`).
Проверка: `sudo grep 'UFW BLOCK' /var/log/ufw.log | tail -5`.

## 7. Имя узла и DNS

- На ВМ: `sudo hostnamectl set-hostname devops-vm`, в `/etc/hosts` строка
  `127.0.1.1 devops-vm.devops.local devops-vm`; `hostname -f` даёт `devops-vm.devops.local`.
- На Mac в `/etc/hosts`: `192.168.64.3 devops.local`, затем
  `sudo dscacheutil -flushcache && sudo killall -HUP mDNSResponder`.
  Первый запрос к `.local` на macOS может быть медленным.

## 8. Снимки состояния

В UTM вместо снимков использованы клоны ВМ (рабочая ВМ: `Linux`).

| Наименование | Момент создания |
|---|---|
| Linux-01-clean-install : после установки и обновления, до создания пользователя devops |
| Linux-02-keys-configured : после настройки ключей и ~/.ssh/config |
| Linux-03-ssh-hardened : после hardening SSH |

## 9. Скрипт проверки

`scripts/audit.sh` проверяет: root-вход запрещён, пароль отключён, порт не 22,
MaxAuthTries равен 3, UFW активен, политика incoming равна deny. Код возврата 0 при
успехе, 1 при ошибках.


## 10. Веб-сервер

### 10.1 Пакет и служба

- Пакет: `nginx` (версия 1.28.3, Ubuntu), установка: `sudo apt update && sudo apt install -y nginx`.
- Служба включена в автозапуск (`enabled`), главный процесс работает от root, рабочие процессы от `www-data`.
- Проблема при установке: установщик оставил источник apt `/etc/apt/sources.list.d/cdrom.sources`. После отключения ISO `apt update` завершался ошибкой, и `apt install` не выполнялся. Решение:
  `sudo mv /etc/apt/sources.list.d/cdrom.sources /etc/apt/sources.list.d/cdrom.sources.disabled`
- Служба после установки оказалась остановлена (`inactive (dead)`), запущена вручную: `sudo systemctl start nginx`.
- Профили UFW `Nginx HTTP/HTTPS/Full` не используются: правила для портов 80 и 443 уже заданы (раздел 6).

### 10.2 Конфигурация ресурса

- Файл: `/etc/nginx/sites-available/devops-site`, активирован ссылкой:
  `sudo ln -s /etc/nginx/sites-available/devops-site /etc/nginx/sites-enabled/`
- Стандартный ресурс отключён: удалена ссылка `/etc/nginx/sites-enabled/default` (файл в `sites-available` сохранён).
- Два блока `server`:
  - порты 80 (IPv4 и IPv6), `server_name devops.local`, `return 301 https://$host$request_uri;`;
  - порты 443 ssl (IPv4 и IPv6), `server_name devops.local`, `ssl_protocols TLSv1.2 TLSv1.3`, `root /var/www/devops-site`, `index index.html`, `try_files $uri $uri/ =404`, `error_page 404 /404.html`.
- Журналы: `/var/log/nginx/devops-site.access.log`, `/var/log/nginx/devops-site.error.log`.
- Заголовок HSTS не используется: браузер запомнил бы требование доверенного сертификата, а сертификат самоподписанный.
- Применение изменений только так: `sudo nginx -t && sudo systemctl reload nginx` (`nginx -t` запускать с `sudo`, иначе ошибка записи `/run/nginx.pid`).

### 10.3 Каталог ресурса

- Путь: `/var/www/devops-site`, владелец `devops:devops`.
- Права: каталоги 755, файлы 644. Модель «владелец пишет, веб-сервер читает»: у `www-data` права только на чтение, создать или изменить файл он не может.
- Создание:
```
  sudo mkdir -p /var/www/devops-site
  sudo chown -R devops:devops /var/www/devops-site
```
- Состав: `index.html`, `404.html` (из каталога `site/` репозитория).

### 10.4 Сертификат и ключ TLS

| Файл | Путь | Владелец | Права |
|---|---|---|---|
| Сертификат | `/etc/ssl/certs/devops.crt` | root | 644 |
| Закрытый ключ | `/etc/ssl/private/devops.key` | root | 600 |

- Сертификат самоподписанный (subject и issuer совпадают: `CN=devops.local`), SAN: `DNS:devops.local`, срок действия 365 дней.
- Команда формирования:
```
  sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout /etc/ssl/private/devops.key \
    -out /etc/ssl/certs/devops.crt \
    -subj "/CN=devops.local" \
    -addext "subjectAltName=DNS:devops.local"
```
- Закрытый ключ не копируется за пределы `/etc/ssl/private` и не добавляется в репозиторий. Главный процесс nginx (root) читает ключ при запуске, рабочие процессы ему не нужны.
- Доверие на хосте: сертификат копируется по SSH и передаётся клиенту явно:
```
  scp devops:/etc/ssl/certs/devops.crt ~/devops.crt
  curl --cacert ~/devops.crt -I https://devops.local
```
- Сертификат действует до 19.10.2027, после этого его необходимо выпустить заново.

### 10.5 Доставка содержимого

- Скрипт `scripts/deploy.sh [--dry-run]` синхронизирует каталог `site/` репозитория с `/var/www/devops-site` командой
  `rsync -avz --delete --chmod=D755,F644` через одно SSH-соединение (`BatchMode=yes`).
- Перед доставкой скрипт проверяет наличие `site/index.html` и отсутствие незафиксированных изменений, после доставки проверяет доступность `https://devops.local` с проверкой сертификата (`--cacert`, без `-k`). Код возврата 0 при успехе, 1 при ошибке.
- Параметр `--delete` всегда сначала проверяется с `--dry-run`: при пустом `site/` он удалил бы главную страницу (проверено на практике, результат 403).
- Проблема: системный rsync на macOS не поддерживает `--chmod` (`invalid argument`). Решение: `brew install rsync`, убедиться, что `which rsync` показывает `/opt/homebrew/bin/rsync`.
- Ограничение: `ufw limit` на порту 2222 блокирует источник после шести соединений за 30 секунд, поэтому команды `ssh`, `scp`, `rsync` подряд не запускаются.

### 10.6 Проверка

- Синтаксис: `sudo nginx -t`; порты: `sudo ss -tlnp | grep nginx` (80 и 443).
- Перенаправление: `curl -I http://devops.local` (301, `Location: https://devops.local/`).
- Доступность: `curl --cacert ~/devops.crt -I https://devops.local` (200).
- Права: `ls -la /var/www/devops-site/`, `find /var/www/devops-site -perm -o+w` (пусто), `sudo ls -l /etc/ssl/private/devops.key`.
- Автоматически: `scripts/audit.sh`, раздел `[4] Веб-сервер`.

### 10.7 Снимок состояния

| Наименование | Момент создания |
|---|---|
| Linux-04-nginx-https | после установки nginx, настройки HTTPS и перенаправления, до доставки через rsync |