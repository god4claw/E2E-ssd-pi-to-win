# AGENTS.md — автоматическое развёртывание

Сценарий для агента, который разворачивает установку из [README.md](README.md) без участия человека. Ручной документ описывает **ту же самую** систему; при расхождении верным считается README.

Документ построен так, чтобы каждый шаг был **идемпотентным** (повторный запуск ничего не ломает) и **проверяемым** (после шага есть команда, дающая однозначный ответ «получилось / нет»). Агент не переходит к следующему шагу, пока проверка предыдущего не прошла.

---

## 0. Границы и правила

**Что агенту разрешено:** размечать указанный диск, создавать системного пользователя, писать в `/etc/ssh/sshd_config.d/`, `/etc/fstab`, `/etc/ssh/authorized_keys.d/`, ставить пакеты на клиенте, создавать задание автозапуска.

**Что агенту запрещено без отдельного подтверждения человека:**

1. Размечать диск, на котором есть данные. Перед `mklabel` обязательна проверка из шага A2; непустой диск — остановка со звонком человеку.
2. Печатать в журнал, вывод или отчёт содержимое `rclone.conf`, приватных ключей и паролей `crypt`. В логах допустимы только маскированные значения.
3. Менять глобальный `/etc/ssh/sshd_config`. Все правила — только отдельным файлом в `sshd_config.d/`.
4. Перезагружать `sshd`, не пройдя `sshd -t`.
5. Трогать существующие задания автозапуска и разделы `rclone.conf` с другими именами.
6. Считать развёртывание успешным без прохождения гейта из раздела 6.

**Обращение с секретами.** Пароли `crypt` порождаются на клиенте, никуда не передаются и не покидают машину. Агент обязан вывести их человеку **один раз**, в явном блоке «запишите и сохраните вне компьютера», и больше нигде не повторять. Если канал вывода агента логируется — генерацию паролей выполняет человек, агент только проверяет, что раздел `secret` появился.

---

## 1. Входные параметры

| имя | смысл | пример | обязательный |
|---|---|---|---|
| `PI_HOST` | адрес Raspberry Pi | `192.168.1.50` | да |
| `PI_ADMIN` | учётка с `sudo` на Pi | `pi` | да |
| `PI_ADMIN_KEY` | приватный ключ для входа под `PI_ADMIN` | `~/.ssh/id_ed25519` | да |
| `DISK` | устройство SSD | `/dev/nvme0n1` | да |
| `STORE_USER` | системный пользователь-хранитель | `cloud` | нет, по умолчанию `cloud` |
| `STORE_ROOT` | каталог-тюрьма | `/srv/cloud` | нет |
| `DATA_DIR` | каталог данных внутри тюрьмы | `/srv/cloud/data` | нет |
| `WIN_DRIVE` | буква диска на клиенте | `X:` | нет |
| `CACHE_SIZE` | потолок локального кэша | `8G` | нет |

Инвариант: `DATA_DIR` всегда лежит внутри `STORE_ROOT`, иначе запирание пользователя работать не будет.

---

## 2. Предусловия (проверить до изменений)

```bash
# П1. Pi достижима и админ имеет sudo без пароля
ssh -i "$PI_ADMIN_KEY" -o BatchMode=yes "$PI_ADMIN@$PI_HOST" 'sudo -n true' && echo OK-П1

# П2. это действительно Raspberry Pi с systemd и OpenSSH
ssh ... 'test -f /proc/device-tree/model && systemctl --version >/dev/null && sshd -V 2>&1 | head -1' && echo OK-П2

# П3. диск существует и это блочное устройство
ssh ... "test -b $DISK" && echo OK-П3

# П4. на диске нет разделов с данными  ← БЛОКИРУЮЩАЯ
ssh ... "lsblk -no NAME,FSTYPE,MOUNTPOINT $DISK"
# пусто → можно размечать; есть строки → ОСТАНОВКА, спросить человека

# П5. свободно не меньше 1 ГБ на системном разделе Pi
ssh ... 'df -BM --output=avail / | tail -1'
```

На клиенте:

```powershell
# П6. Windows 10/11 и права на создание заданий
[System.Environment]::OSVersion.Version
# П7. winget доступен
winget --version
```

Любое непройденное предусловие — остановка без изменений.

---

## 3. Фаза A — сторона Raspberry Pi

Все команды выполняются через `ssh -i "$PI_ADMIN_KEY" "$PI_ADMIN@$PI_HOST"`.

### A1. Диск

```bash
sudo parted -s "$DISK" mklabel gpt
sudo parted -s -a optimal "$DISK" mkpart cloud ext4 1MiB 100%
sleep 2
sudo mkfs.ext4 -F -m 0 -L cloud "${DISK}p1"
```

Идемпотентность: если `blkid -L cloud` уже возвращает устройство — шаг пропускается целиком. Повторное форматирование стирает данные, поэтому проверка обязательна **до** выполнения.

Проверка: `blkid -L cloud` возвращает непустую строку.

### A2. Монтирование

```bash
UUID=$(sudo blkid -s UUID -o value "${DISK}p1")
sudo mkdir -p "$STORE_ROOT"
grep -q "$UUID" /etc/fstab || \
  echo "UUID=$UUID $STORE_ROOT ext4 defaults,noatime,nofail 0 2" | sudo tee -a /etc/fstab
sudo mount -a
```

Идемпотентность обеспечивается `grep -q` — строка в `fstab` не дублируется.

Проверка: `findmnt -n "$STORE_ROOT"` даёт строку с `ext4` и опцией `noatime`.

### A3. Пользователь и права

```bash
id "$STORE_USER" >/dev/null 2>&1 || \
  sudo adduser --system --group --home "$STORE_ROOT" --shell /usr/sbin/nologin "$STORE_USER"
sudo chown root:root "$STORE_ROOT"
sudo chmod 755 "$STORE_ROOT"
sudo mkdir -p "$DATA_DIR"
sudo chown "$STORE_USER:$STORE_USER" "$DATA_DIR"
sudo chmod 700 "$DATA_DIR"
```

Проверка (обе строки обязаны совпасть):

```bash
stat -c '%U:%G %a' "$STORE_ROOT"   # ожидается: root:root 755
stat -c '%U:%G %a' "$DATA_DIR"     # ожидается: cloud:cloud 700
```

Если `STORE_ROOT` окажется не root'овым или доступным на запись группе — запирание не заработает, и вход будет отбиваться без внятной причины. Это первая причина неудачных развёртываний; проверять обязательно.

### A4. Ключ

Ключ порождается **на клиенте** (фаза B1) и передаётся сюда только публичной половиной.

```bash
sudo mkdir -p /etc/ssh/authorized_keys.d
printf 'restrict %s\n' "$PUBKEY" | sudo tee "/etc/ssh/authorized_keys.d/$STORE_USER" >/dev/null
sudo chown root:root "/etc/ssh/authorized_keys.d/$STORE_USER"
sudo chmod 644 "/etc/ssh/authorized_keys.d/$STORE_USER"
```

Проверка: файл существует, владелец root, режим 644, первая лексема строки — `restrict`.

### A5. Правило SSH

```bash
sudo tee /etc/ssh/sshd_config.d/50-cloud.conf >/dev/null <<CONF
# Хранилище: SFTP-only, заперт в $STORE_ROOT, без шелла и без пробросов.
Match User $STORE_USER
    ChrootDirectory $STORE_ROOT
    ForceCommand internal-sftp -d /data
    AuthorizedKeysFile /etc/ssh/authorized_keys.d/%u
    PubkeyAuthentication yes
    PasswordAuthentication no
    KbdInteractiveAuthentication no
    AllowTcpForwarding no
    AllowAgentForwarding no
    X11Forwarding no
    PermitTunnel no
    PermitTTY no
CONF
sudo sshd -t && sudo systemctl reload ssh
```

Порядок критичен: `sshd -t` **до** перезагрузки конфигурации. Если проверка не прошла — удалить созданный файл и остановиться, иначе можно потерять удалённый доступ.

Проверка: `sudo sshd -T -C user=$STORE_USER 2>/dev/null | grep -E 'chrootdirectory|forcecommand'` показывает ожидаемые значения.

---

## 4. Фаза B — сторона Windows

### B1. Программы и ключ

```powershell
winget install --silent --accept-package-agreements WinFsp.WinFsp
winget install --silent --accept-package-agreements Rclone.Rclone

$key = "$env:USERPROFILE\.ssh\id_ed25519_cloud"
if (-not (Test-Path $key)) { ssh-keygen -t ed25519 -f $key -N '""' -C "cloud@win" }
Get-Content "$key.pub"     # ← это значение уходит в шаг A4 как $PUBKEY
```

Идемпотентность: ключ создаётся только если его нет; `winget` повторную установку пропускает.

### B2. Отпечаток сервера

```powershell
ssh-keyscan -t ed25519 $PI_HOST > "$env:USERPROFILE\.ssh\known_hosts_cloud"
```

Агент обязан сверить полученный отпечаток с выводом `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` на Pi и остановиться при расхождении. Молчаливое принятие любого ключа лишает смысла весь транспортный слой.

### B3. Разделы rclone

```powershell
rclone config create pi sftp `
  host=$PI_HOST user=cloud `
  key_file="$env:USERPROFILE\.ssh\id_ed25519_cloud" `
  known_hosts_file="$env:USERPROFILE\.ssh\known_hosts_cloud" `
  shell_type=none disable_hashcheck=true md5sum_command=none sha1sum_command=none

# пароли: сгенерировать на месте, показать человеку один раз, никуда не писать
$p1 = rclone obscure (rclone genautopassword 2>$null)   # либо ввод от человека
rclone config create secret crypt `
  remote=pi:/data `
  filename_encryption=standard directory_name_encryption=true `
  filename_encoding=base32768 `
  password=$p1 password2=$p2 --obscure
```

Идемпотентность: если раздел `secret` уже существует — **не пересоздавать**. Новые пароли сделают ранее записанные данные нечитаемыми. Проверка перед созданием: `rclone listremotes` не содержит `secret:`.

Проверка: `rclone lsd secret:` завершается с кодом 0.

### B4. Монтирование и автозапуск

```powershell
$rclone = (Get-Command rclone).Source
$arguments = "mount secret: $WIN_DRIVE --vfs-cache-mode full --vfs-cache-max-size $CACHE_SIZE " +
             '--vfs-cache-max-age 72h --vfs-write-back 5s --dir-cache-time 30s ' +
             '--volname "Cloud" --transfers 8 --checkers 8 --no-console ' +
             "--log-file `"$env:LOCALAPPDATA\rclone\mount.log`" --log-level INFO"

if (-not (Get-ScheduledTask -TaskName 'RcloneCloudMount' -ErrorAction SilentlyContinue)) {
    $action  = New-ScheduledTaskAction -Execute $rclone -Argument $arguments
    $trigger = New-ScheduledTaskTrigger -AtLogOn
    $trigger.Delay = 'PT20S'
    Register-ScheduledTask -TaskName 'RcloneCloudMount' -Action $action -Trigger $trigger -RunLevel Limited
}
Start-ScheduledTask -TaskName 'RcloneCloudMount'
```

Проверка: в течение 30 секунд `Test-Path "$WIN_DRIVE\"` становится истиной. WinFsp поднимает том не мгновенно — 8 секунд может не хватить, ожидание должно быть циклом с опросом, а не одной паузой.

---

## 5. Фаза C — проверка сквозного шифрования

Обязательная фаза. Развёртывание без неё не принимается: без этой проверки утверждение «данные зашифрованы» ничем не подтверждено.

```powershell
$marker = "E2E-CHECK-" + [guid]::NewGuid().ToString("N")
mkdir "$WIN_DRIVE\_selfcheck" -Force | Out-Null
$marker | Out-File -Encoding utf8 "$WIN_DRIVE\_selfcheck\probe.txt"
Start-Sleep -Seconds 10          # дождаться отложенной записи
```

На Pi:

```bash
f=$(sudo find "$DATA_DIR" -newermt "-3 minutes" -type f | head -1)
test -n "$f"                                    # файл доехал
sudo head -c 8 "$f" | grep -q 'RCLONE'          # формат тот
test "$(sudo grep -ac "$MARKER" "$f")" = "0"    # открытого текста нет
basename "$f" | grep -qv "probe"                # имя не совпадает с исходным
```

Все четыре условия обязаны выполниться. После проверки:

```powershell
Remove-Item -Recurse -Force "$WIN_DRIVE\_selfcheck"
```

---

## 6. Гейт приёмки

Развёртывание успешно, только если выполнены **все** пункты. Иначе — статус «не завершено», с указанием непройденного номера.

| № | что проверяется | команда | ожидание |
|---|---|---|---|
| 1 | диск смонтирован и переживает перезагрузку | `findmnt -n $STORE_ROOT` | строка с `ext4`, `noatime` |
| 2 | права на запирание корректны | `stat -c '%U:%G %a' $STORE_ROOT $DATA_DIR` | `root:root 755` и `cloud:cloud 700` |
| 3 | правило SSH применено | `sudo sshd -T -C user=cloud \| grep chrootdirectory` | путь `STORE_ROOT` |
| 4 | оболочка недоступна | `ssh -i <ключ> cloud@$PI_HOST` | приглашения нет, соединение закрывается |
| 5 | хранилище отвечает | `rclone lsd secret:` | код выхода 0 |
| 6 | диск смонтирован | `Test-Path "$WIN_DRIVE\"` | `True` |
| 7 | шифрование настоящее | фаза C | все 4 условия |
| 8 | автозапуск зарегистрирован | `Get-ScheduledTask RcloneCloudMount` | состояние `Ready` |
| 9 | скорость в рамках ожидаемого | копирование 1 ГБ через `rclone copyto` | ≥ 80 МБ/с на гигабитной сети |

Пункт 9 — не придирка: результат заметно ниже 80 МБ/с означает, что соединение ушло на Wi-Fi, порт согласовался на 100 Мбит/с или кабель плохой. Диагностика: `ethtool eth0 \| grep Speed` на Pi.

---

## 7. Откат

Порядок обратный установке. Данные при откате **не** удаляются — только доступ и обвязка.

```powershell
# клиент
Stop-Process -Name rclone -ErrorAction SilentlyContinue
Unregister-ScheduledTask -TaskName 'RcloneCloudMount' -Confirm:$false
rclone config delete secret
rclone config delete pi
```

```bash
# Pi
sudo rm -f /etc/ssh/sshd_config.d/50-cloud.conf
sudo sshd -t && sudo systemctl reload ssh
sudo rm -f /etc/ssh/authorized_keys.d/cloud
# пользователя и данные оставить; удалять только по явному указанию человека
```

Удаление раздела `secret` из конфигурации **не уничтожает данные**, но без сохранённых паролей восстановить их будет невозможно. Агент обязан предупредить об этом до отката.

---

## 8. Частые отказы и их распознавание

| симптом | причина | что делать |
|---|---|---|
| вход отбивается сразу после успешной аутентификации | `STORE_ROOT` принадлежит не root или доступен на запись группе | вернуть `root:root 755` |
| `SSH_FX_BAD_MESSAGE` при копировании | зашифрованное имя длиннее допустимого | укоротить исходное имя либо упаковать папку в архив |
| `couldn't connect SSH: dial tcp ... connectex` при входе в систему | Pi выключена или не поднялась к моменту запуска задания | смонтировать вручную; долгое решение — повторные попытки вместо разового задания |
| rclone спотыкается на подсчёте контрольных сумм | забыты `shell_type=none` / `md5sum_command=none` | дописать в раздел `pi` |
| диск монтируется, но пуст | `remote` указывает не на `pi:/data`, либо не тот пароль | сверить раздел `secret`, **не пересоздавая** его вслепую |
| скорость ~11 МБ/с | соединение ушло на 100 Мбит/с | проверить `ethtool eth0`, кабель, порт роутера |
| скорость ~40 МБ/с и скачет | работа идёт через Wi-Fi | перевести на витую пару |

---

## 9. Что этот сценарий сознательно не делает

Соответствует разделу «не доработано» в README и не должно молча появляться в автоматике:

- не настраивает firewall, `fail2ban` и не отключает вход по паролю для основного пользователя;
- не организует доступ снаружи локальной сети;
- не заводит резервное копирование, зеркалирование и версионирование;
- не ставит расписание проверок целостности и наблюдения за состоянием диска;
- не включает PCIe Gen 3;
- не защищает `rclone.conf` паролем конфигурации;
- не обеспечивает повторные попытки монтирования и автоперемонтирование.

Агент, который добавляет что-то из этого списка, обязан явно сообщить человеку, что вышел за границы описанного сценария.
