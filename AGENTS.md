# AGENTS.md — автоматическое развёртывание

Сценарий для агента, который разворачивает установку из [docs/1-install](docs/1-install/README.md) без участия человека. Ручной документ описывает **ту же самую** систему; при расхождении верными считаются `docs/`.

Каждый шаг **идемпотентен** (повторный запуск ничего не ломает) и **проверяем** (после шага есть команда с однозначным ответом «получилось / нет»). Агент не переходит к следующему шагу, пока проверка предыдущего не прошла.

---

## 0. Границы и правила

**Что агенту разрешено:** размечать указанный диск, создавать системного пользователя, писать в `/etc/ssh/sshd_config.d/`, `/etc/fstab`, `/etc/ssh/authorized_keys.d/`, ставить пакеты на клиенте, создавать задание автозапуска.

**Что агенту запрещено без отдельного подтверждения человека:**

1. Размечать диск, на котором есть данные. Перед `mklabel` обязательна проверка из П4; непустой диск — остановка со звонком человеку.
2. Печатать в журнал, вывод или отчёт содержимое `rclone.conf`, приватных ключей и паролей `crypt`. В логах допустимы только маскированные значения.
3. Менять глобальный `/etc/ssh/sshd_config`. Все правила — только отдельным файлом в `sshd_config.d/`.
4. Перезагружать `sshd`, не пройдя `sshd -t`.
5. Трогать существующие задания автозапуска и разделы `rclone.conf` с другими именами.
6. Считать развёртывание успешным без прохождения гейта из раздела 6.

**Обращение с секретами — главное правило.** Пароли `crypt` (и основной, и соль) генерирует и вводит **человек**. Агент их не порождает, не запоминает, не передаёт параметрами командной строки и не печатает. Его работа — сказать человеку, что делать (`rclone config`), и потом проверить результат: раздел `secret:` существует, `rclone lsd secret:` отвечает.

> [!WARNING]
> В репозиториях встречаются сценарии вида `rclone obscure (rclone genautopassword)`. Команды `genautopassword` в rclone **нет** — это выдумка, которая молча создаёт пустой пароль, а вместе с флагом `--obscure` ещё и маскирует его дважды. Данные ложатся под ключ, которого никто не видел. Здесь этот путь закрыт: пароль приходит от человека.

---

## 1. Входные параметры

| имя | смысл | пример | обязательный |
|---|---|---|---|
| `PI_HOST` | адрес Raspberry Pi (IPv4-литерал) | `192.168.1.50` | да |
| `PI_ADMIN` | учётка с `sudo` на Pi | `pi` | да |
| `PI_ADMIN_KEY` | приватный ключ для входа под `PI_ADMIN` | `~/.ssh/id_ed25519` | да |
| `DISK` | устройство SSD | `/dev/nvme0n1` | да |
| `STORE_USER` | системный пользователь-хранитель | `cloud` | нет, по умолчанию `cloud` |
| `STORE_ROOT` | каталог-тюрьма | `/srv/cloud` | нет |
| `DATA_DIR` | каталог данных внутри тюрьмы | `/srv/cloud/data` | нет |
| `WIN_DRIVE` | буква диска на клиенте | `X:` | нет |
| `CACHE_SIZE` | потолок локального кэша | `8G` | нет |

Инвариант: `DATA_DIR` всегда лежит внутри `STORE_ROOT`, иначе запирание пользователя работать не будет.

Имя раздела зависит от устройства: `nvme0n1` → `nvme0n1p1`, `sda` → `sda1`, `mmcblk0` → `mmcblk0p1`. Дальше используется переменная `PART`, а не строка с `p1`.

```bash
case "$DISK" in
  *nvme*|*mmcblk*|*loop*) PART="${DISK}p1" ;;
  *)                      PART="${DISK}1"  ;;
esac
```

---

## 2. Предусловия (проверить до изменений)

```bash
# П1. Pi достижима и админ имеет sudo без пароля
ssh -i "$PI_ADMIN_KEY" -o BatchMode=yes "$PI_ADMIN@$PI_HOST" 'sudo -n true' && echo OK-П1

# П2. это Raspberry Pi с systemd и OpenSSH
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
[System.Environment]::OSVersion.Version                 # П6. Windows 10/11
winget --version                                        # П7. winget доступен
(Get-Command ssh-keygen -ErrorAction SilentlyContinue)   # П8. OpenSSH-клиент есть
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
sudo mkfs.ext4 -F -m 0 -L cloud "$PART"
```

Идемпотентность: если `blkid -L cloud` уже возвращает устройство — шаг пропускается целиком. Повторное форматирование стирает данные, поэтому проверка обязательна **до** выполнения.

Проверка: `blkid -L cloud` возвращает непустую строку.

### A2. Монтирование

```bash
UUID=$(sudo blkid -s UUID -o value "$PART")
sudo mkdir -p "$STORE_ROOT"
grep -q "$UUID" /etc/fstab || \
  echo "UUID=$UUID $STORE_ROOT ext4 defaults,noatime,nofail,x-systemd.device-timeout=10s 0 2" | sudo tee -a /etc/fstab
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

Ключ порождается **на клиенте** (B1) и передаётся сюда только публичной половиной.

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
winget install --silent --accept-package-agreements --accept-source-agreements WinFsp.WinFsp
winget install --silent --accept-package-agreements --accept-source-agreements Rclone.Rclone

$key = "$env:USERPROFILE\.ssh\id_ed25519_cloud"
if (-not (Test-Path $key)) { ssh-keygen -t ed25519 -f $key -N '""' -C "cloud@win" }
Get-Content "$key.pub"     # ← это значение уходит в A4 как $PUBKEY
```

Идемпотентность: ключ создаётся только если его нет; `winget` повторную установку пропускает.

`--accept-source-agreements` обязателен при первом запуске `winget`: иначе установка встаёт на соглашении об источнике и ждёт ответа, которого не будет.

### B2. Отпечаток сервера

```powershell
ssh-keyscan -t ed25519 $PI_HOST > "$env:USERPROFILE\.ssh\known_hosts_cloud"
ssh-keygen -lf "$env:USERPROFILE\.ssh\known_hosts_cloud"
```

Агент обязан сверить полученный отпечаток с выводом `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` на Pi и остановиться при расхождении. Молчаливое принятие любого ключа лишает смысла весь транспортный слой.

### B3. Разделы rclone

Транспортный раздел агент создаёт сам:

```powershell
rclone config create pi sftp `
  host=$PI_HOST user=cloud `
  key_file="$env:USERPROFILE\.ssh\id_ed25519_cloud" `
  known_hosts_file="$env:USERPROFILE\.ssh\known_hosts_cloud" `
  shell_type=none disable_hashcheck=true md5sum_command=none sha1sum_command=none
```

Раздел `secret` создаёт **человек** — ему в руки передаётся `rclone config` с точным набором ответов:

| вопрос мастера | ответ |
|---|---|
| имя | `secret` |
| тип | `crypt` |
| `remote` | `pi:/data` |
| `filename_encryption` | `standard` |
| `directory_name_encryption` | `true` |
| `filename_encoding` | `base32768` |
| пароль | `g` (сгенерировать), длина 128 → **записать на бумагу** |
| второй пароль (соль) | `g`, тоже 128 → **записать на бумагу** |

Идемпотентность: если `rclone listremotes` уже содержит `secret:`, раздел **не пересоздаётся**. Новые пароли сделают ранее записанные данные нечитаемыми.

Проверка агентом: `rclone listremotes` содержит `secret:`; `rclone lsd secret:` завершается с кодом 0.

> [!NOTE]
> Если человек настаивает на полной автоматизации, он сам выполняет `rclone config create secret crypt … password=<открытый> password2=<открытый> --obscure` в своём терминале: `--obscure` маскирует открытое значение ровно один раз. Передавать туда уже замаскированные значения нельзя — маскировка применится повторно, и ключ будет другим. Агент в любом случае значения не видит.

### B4. Монтирование и автозапуск

Обёртка `%USERPROFILE%\rclone-mount.ps1` — ждёт Pi и перезапускает rclone после падения:

```powershell
# WIN_DRIVE и CACHE_SIZE подставьте из раздела 1 (по умолчанию X: и 8G)
$rclone = (Get-Command rclone).Source
$deadline = (Get-Date).AddMinutes(10)
while ((Get-Date) -lt $deadline) {
    & $rclone lsd secret: 2>$null | Out-Null
    if ($LASTEXITCODE -eq 0) { break }
    Start-Sleep -Seconds 15
}
while ($true) {
    & $rclone mount secret: X: `
        --vfs-cache-mode full --vfs-cache-max-size 8G --vfs-cache-min-free-space 10G `
        --vfs-cache-max-age 72h --vfs-write-back 5s --dir-cache-time 30s `
        --volname "Cloud" --transfers 8 --no-console --log-level NOTICE `
        --log-file "$env:LOCALAPPDATA\rclone\mount.log"
    Start-Sleep -Seconds 20
}
```

Задание — с явными настройками. Без них оно умрёт на четвёртые сутки (лимит по умолчанию — 3 дня) и не поднимется после выключенной Pi:

```powershell
if (-not (Get-ScheduledTask -TaskName 'RcloneCloudMount' -ErrorAction SilentlyContinue)) {
    $action  = New-ScheduledTaskAction -Execute 'powershell.exe' `
      -Argument '-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "%USERPROFILE%\rclone-mount.ps1"'
    $trigger = New-ScheduledTaskTrigger -AtLogOn
    $trigger.Delay = 'PT20S'
    $settings = New-ScheduledTaskSettingsSet `
      -ExecutionTimeLimit ([TimeSpan]::Zero) `
      -RestartCount 999 -RestartInterval (New-TimeSpan -Minutes 1) `
      -MultipleInstances IgnoreNew `
      -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries -StartWhenAvailable
    Register-ScheduledTask -TaskName 'RcloneCloudMount' -Action $action -Trigger $trigger `
      -Settings $settings -RunLevel Limited
}
Start-ScheduledTask -TaskName 'RcloneCloudMount'
```

Проверка: в течение 60 секунд `Test-Path "$WIN_DRIVE\"` становится истиной. WinFsp поднимает том не мгновенно — ожидание должно быть циклом с опросом, а не одной паузой.

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

Развёртывание успешно, только если выполнены **все** применимые пункты. Иначе — статус «не завершено», с указанием непройденного номера. Пункт 11 применяется, только если в этом прогоне настраивалась копия; иначе он снимается с явной пометкой в отчёте.

| № | что проверяется | команда | ожидание |
|---|---|---|---|
| 1 | диск смонтирован и переживает перезагрузку | `findmnt -n $STORE_ROOT` | строка с `ext4`, `noatime` |
| 2 | права на запирание корректны | `stat -c '%U:%G %a' $STORE_ROOT $DATA_DIR` | `root:root 755` и `cloud:cloud 700` |
| 3 | правило SSH применено | `sudo sshd -T -C user=cloud \| grep chrootdirectory` | путь `STORE_ROOT` |
| 4 | оболочка недоступна | `ssh -i <ключ> cloud@$PI_HOST 'id'` | приглашения нет, команда не выполняется |
| 5 | хранилище отвечает | `rclone lsd secret:` | код выхода 0 |
| 6 | диск смонтирован | `Test-Path "$WIN_DRIVE\"` | `True` |
| 7 | шифрование настоящее | фаза C | все 4 условия |
| 8 | автозапуск зарегистрирован и настроен | `Get-ScheduledTask RcloneCloudMount` | состояние `Ready`, лимит выполнения снят |
| 9 | скорость в рамках ожидаемого | копирование 1 ГБ через `rclone copyto` | ≥ 80 МБ/с на гигабитной сети |
| 10 | диск виден из программы от администратора | `Test-Path X:\` в консоли от админа | `True` либо зафиксировано обратное и включён `--network-mode` |
| 11 | восстановление по бумажке | отдельная конфигурация из паролей человека | файл читается, хеш совпадает |

Пункт 9 — не придирка: результат заметно ниже 80 МБ/с означает, что соединение ушло на Wi-Fi, порт согласовался на 100 Мбит/с или кабель плохой. Диагностика: `ethtool eth0 | grep Speed` на Pi.

Пункт 8 проверяется как настройка, а не «просто состояние `Ready`»: у задания с лимитом по умолчанию диск исчезнет на четвёртые сутки, и это не будет выглядеть поломкой.

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
| `SSH_FX_BAD_MESSAGE` при копировании | зашифрованное имя длиннее 255 байт | укоротить исходное имя либо упаковать папку в архив |
| `couldn't connect SSH: dial tcp ... connectex` при входе в систему | Pi выключена или не поднялась к моменту запуска | обёртка с повторами (B4) |
| диск исчез через несколько дней | у задания лимит выполнения 3 суток | `-ExecutionTimeLimit ([TimeSpan]::Zero)` (B4) |
| rclone спотыкается на подсчёте контрольных сумм | забыты `shell_type=none` / `md5sum_command=none` | дописать в раздел `pi` |
| диск монтируется, но пуст | `remote` указывает не на `pi:/data`, либо не тот пароль | сверить раздел `secret`, **не пересоздавая** его вслепую |
| скорость ~11 МБ/с | соединение ушло на 100 Мбит/с | проверить `ethtool eth0`, кабель, порт роутера |
| скорость ~40 МБ/с и скачет | работа идёт через Wi-Fi | перевести на витую пару |
| приложения от администратора не видят диск | буква создана обычным процессом | зафиксировать в отчёте, при необходимости `--network-mode` |

---

## 9. Что этот сценарий сознательно не делает

Соответствует [§18](docs/5-hardening/18-gaps.md) и не должно молча появляться в автоматике:

- не настраивает firewall, `fail2ban` и не отключает вход по паролю для основного пользователя ([§15](docs/5-hardening/15-perimeter.md));
- не организует доступ снаружи локальной сети;
- не заводит резервное копирование, зеркалирование и версионирование ([§17](docs/5-hardening/17-backup.md));
- не ставит расписание проверок целостности и наблюдения за состоянием диска ([§16](docs/5-hardening/16-monitoring.md));
- не включает PCIe Gen 3;
- не защищает `rclone.conf` паролем конфигурации;
- не делает LUKS на SSD;
- не настраивает уведомления о сбоях.

Агент, который добавляет что-то из этого списка, обязан явно сообщить человеку, что вышел за границы описанного сценария.
