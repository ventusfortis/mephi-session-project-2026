# Сессионный проект по администрированию GNU/Linux (МИФИ, 2026)

Стенд: РЕД ОС 8.0.3 (Standard Desktop), ядро 6.12, SELinux Enforcing, виртуальная машина KVM/Proxmox.
Работа выполнялась от имени `root`. Все настройки хранятся в конфигурационных файлах и метаданных файловых систем
и проверены на сохранение после перезагрузки (hostname, автозапуск nginx, монтирование по метке).

## Раздел 1. Установка дистрибутива

Сетевой интерфейс `ens18` получает адрес по DHCP (`ipv4.method auto` в NetworkManager).
Имя хоста задано через `hostnamectl`, связность проверена `ping` до 8.8.8.8.

```bash
nmcli con mod ens18 ipv4.method auto
hostnamectl set-hostname mephi-2026.domain.local
ping -c 4 8.8.8.8 > ~/ping.out
```

Подтверждение: `ping.out` (4 пакета отправлено, 4 получено, 0% потерь).

## Раздел 2. Управление программным обеспечением

Система обновлена целиком, из репозиториев установлены `nginx` и `libcap-ng-utils`.
Пакет `tcpdump` скачан в `/tmp` менеджером `dnf download` и установлен командой `rpm`.

```bash
dnf -y update
dnf -y install nginx libcap-ng-utils
dnf download --arch x86_64 tcpdump --destdir /tmp
rpm -ivh /tmp/tcpdump-4.99.5-1.red80.x86_64.rpm
dnf history > ~/dnf.out
```

Примечание: без `--arch x86_64` команда `dnf download` скачивает также i686-вариант пакета,
и `rpm -ivh /tmp/tcpdump-*.rpm` завершается ошибкой зависимостей. Это видно в `history.out`.

Подтверждение: `dnf.out` (транзакции обновления и установки), файл `tcpdump-4.99.5-1.red80.x86_64.rpm`.

## Раздел 3. Управление файловыми системами

На втором диске `/dev/sdb` (16 ГБ, SATA) создана таблица разделов GPT и один раздел на весь диск.
Раздел отформатирован в ext4 с меткой `MEPHI_WEB`, точка монтирования `/mephi-web`,
в `/etc/fstab` добавлена запись по метке тома.

```bash
parted -s /dev/sdb mklabel gpt
parted -s /dev/sdb mkpart mephi-web ext4 0% 100%
mkfs.ext4 -L MEPHI_WEB /dev/sdb1
mkdir -p /mephi-web
echo 'LABEL=MEPHI_WEB  /mephi-web  ext4  defaults  0 2' >> /etc/fstab
systemctl daemon-reload
mount /mephi-web
```

Подтверждение: `fstab` (последняя строка), `stat.out` (каталог `/mephi-web` на устройстве 8/17, то есть `/dev/sdb1`).

## Раздел 4. Управление сервисами

nginx запущен и добавлен в автозагрузку одной командой. Журнал сервиса за текущую загрузку сохранён.

```bash
systemctl enable --now nginx
journalctl -u nginx -b > ~/journalctl.out
```

Подтверждение: `journalctl.out` (автоматический старт при загрузке в 22:11:56 и перезапуск после смены корневого каталога).

## Раздел 5. Управление доступом

### 5.1. Дискреционное управление доступом (DAC)

Созданы разработчики `user1`, `user2`, `user3` (UID 5501–5503) в группе `developers` (GID 5500)
и кураторы `curator1`, `curator2` в группе `curators` (GID 4444).

На каталоге `/data/mephi-2026` установлен владелец `root:developers`, права `2770` (бит setgid, чтобы новые файлы
наследовали группу `developers`), а также ACL и ACL по умолчанию, которые распространяются на все создаваемые файлы:

```bash
chown root:developers /data/mephi-2026
chmod 2770 /data/mephi-2026
setfacl -m g:developers:rwx,g:curators:r-x,o::--- /data/mephi-2026
setfacl -d -m u::rwx,g::rwx,g:developers:rwx,g:curators:r-x,o::--- /data/mephi-2026
```

Проверено реальными операциями: `user1` создал файл, `user2` дописал в него, `user3` создал подкаталог,
`curator1` прочитал файл, но не смог записать (`Отказано в доступе`), `nobody` не может даже открыть каталог.

Подтверждение: `stat.out` (права `2770`, группа `developers`), `extra/getfacl.out`, `extra/id.out`, `passwd`, `group`.

### 5.2. Привилегии (уменьшение количества set-UID-программ)

В РЕД ОС `tcpdump` поставляется без бита set-UID (права `0755`), но захват трафика обычному пользователю недоступен:
`You don't have permission to capture on that device`. Вместо запуска от root файлу выданы только две нужные привилегии:

```bash
setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
sudo -u user1 tcpdump -c 3 -nn -i ens18
```

После настройки `user1` успешно захватывает пакеты. Подтверждение: `getcap.out`.

### 5.3. Мандатное управление доступом (MAC)

SELinux работает в режиме Enforcing (`getenforce`, `SELINUX=enforcing` в `/etc/selinux/config`).
Корневой каталог nginx изменён на `/mephi-web`, для него добавлено постоянное правило контекста
и выполнена перемаркировка. Без неё файлы на новом разделе имели тип `unlabeled_t`, недоступный для nginx.

```bash
sed -i '0,/root\s\+\/usr\/share\/nginx\/html;/s||        root         /mephi-web;|' /etc/nginx/nginx.conf
semanage fcontext -a -t httpd_sys_content_t '/mephi-web(/.*)?'
restorecon -Rv /mephi-web
nginx -t && systemctl restart nginx
```

Подтверждение: `getenforce.out`, `stat.out` (контекст `system_u:object_r:httpd_sys_content_t:s0` у `/mephi-web`),
`extra/ls-Z.out`, `extra/nginx.conf`.

## Раздел 6. Аутентификация

### 6.1. Ограничение входа для кураторов

Запрет локального входа реализован модулем `pam_access` с отдельным файлом правил, подключённым только
к PAM-сервисам локального входа: `login` (консоль) и `sddm` (графический вход). SSH и `su` не затронуты.

```bash
printf '- : curators : ALL\n' > /etc/security/access-console.conf
# в /etc/pam.d/login и /etc/pam.d/sddm после pam_nologin.so:
account    required     pam_access.so accessfile=/etc/security/access-console.conf
```

Проверка выполнена прямым вызовом PAM-стека (скрипт на ctypes): для `curator1` и `curator2` этап `pam_acct_mgmt`
возвращает `Permission denied`, для `user1` — `Success`.

Подтверждение: `extra/access-console.conf`, `extra/pam.d-login`, `extra/pam.d-sddm`.

### 6.2. Управление паролями

```bash
sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS\t90/' /etc/login.defs
for u in user1 user2 user3 curator1 curator2; do chage -M 90 $u; done
sed -i 's/^# minlen = 8/minlen = 12/' /etc/security/pwquality.conf
```

Проверка `pwscore`: пароль из 7 символов отклонён (`должен содержать не менее 12 символов`), пароль из 14 символов принят.

Подтверждение: `shadow` (поле максимального срока 90 у всех пяти пользователей), `pwquality.conf` (`minlen = 12`),
`extra/login.defs`, `extra/chage-user1.out`.

## Раздел 7. Тестирование

```bash
echo 'Hello from Student: 373076' > /mephi-web/index.html
curl http://localhost/
```

Ответ сервера: `Hello from Student: 373076`. Подтверждение: `curl.out`, `mephi-screenshot.png`.

## Раздел 8. Артефакты

- `mephi-screenshot.png` — скриншот с выводом веб-страницы.
- `history.out` — история выполненных команд.
- `ping.out`, `dnf.out`, `stat.out`, `journalctl.out`, `getcap.out`, `getenforce.out`, `curl.out` — выводы команд по заданию.
- `fstab`, `passwd`, `shadow`, `group`, `pwquality.conf` — копии системных файлов из `/etc`.
- `tcpdump-4.99.5-1.red80.x86_64.rpm` — локально установленный пакет.
- `extra/` — дополнительные подтверждения: `nginx.conf`, `login.defs`, `access-console.conf`, `pam.d-login`, `pam.d-sddm`,
  `getfacl.out`, `ls-Z.out`, `id.out`, `chage-user1.out`.
