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