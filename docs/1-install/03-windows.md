# §3. Windows: rclone и диск

<sub>[Главная](../../README.md) · [Часть 1](README.md) · [← §2](02-ssh.md) · [§4. Автозапуск →](04-autostart.md)</sub>

На клиенте работают две программы: WinFsp показывает чужое хранилище буквой диска, rclone шифрует и говорит по SFTP. Никакой третьей программы нет — и это важно: каждое лишнее звено на клиенте видит открытые данные.

## Программы

```powershell
winget install --silent --accept-package-agreements --accept-source-agreements WinFsp.WinFsp
winget install --silent --accept-package-agreements --accept-source-agreements Rclone.Rclone
```

`--accept-source-agreements` нужен, если `winget` на этой машине запускается впервые: без него установка останавливается на соглашении об источнике пакетов и ждёт ответа, которого в автоматическом режиме не будет.

## Отпечаток сервера

Чтобы клиент не соглашался молча на любой сервер, зафиксируйте отпечаток. На Pi:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

На Windows:

```powershell
ssh-keyscan -t ed25519 <IP-ПАЙКИ> > $env:USERPROFILE\.ssh\known_hosts_cloud
ssh-keygen -lf $env:USERPROFILE\.ssh\known_hosts_cloud
```

Сверьте отпечатки глазами. Совпали — дальше; не совпали — вы говорите не с той машиной.

## Настройка rclone

Файл `%APPDATA%\rclone\rclone.conf` — два раздела: транспорт и шифрование поверх него.

```ini
[pi]
type = sftp
host = <IP-ПАЙКИ>
user = cloud
key_file = C:\Users\<ВЫ>\.ssh\id_ed25519_cloud
known_hosts_file = C:\Users\<ВЫ>\.ssh\known_hosts_cloud
shell_type = none
disable_hashcheck = true
md5sum_command = none
sha1sum_command = none

[secret]
type = crypt
remote = pi:/data
filename_encryption = standard
directory_name_encryption = true
filename_encoding = base32768
password = <замаскированный пароль>
password2 = <замаскированная соль>
```

Пояснения к неочевидному:

- `shell_type = none`, `disable_hashcheck`, `md5sum_command = none` — обязательны. Пользователь заперт и не имеет оболочки, значит rclone не может запустить на той стороне `md5sum`. Без этих строк он будет пытаться и спотыкаться.
- `filename_encryption = standard` — шифруются и имена файлов; `directory_name_encryption = true` — и имена папок. Иначе структура каталогов останется на виду.
- `filename_encoding = base32768` — плотная запись зашифрованного имени: 15 бит на символ вместо 5 у base32. Символов в имени становится меньше, но каждый занимает в UTF-8 три байта, поэтому на диске имя всё равно длиннее исходного. Предел длины при этом остаётся — [§7](../2-crypto/07-format.md).
- `known_hosts_file` — отдельный файл, чтобы не смешивать отпечатки с обычным SSH.

**Пароль задавайте через мастер, а не руками:**

```powershell
rclone config
```

`n` → имя `secret` → тип `crypt` → `remote` = `pi:/data` → шифрование имён `standard` → шифрование папок `yes` → пароль: `g` (сгенерировать), длина 128 → второй пароль (соль): тоже сгенерировать.

> [!WARNING]
> В файле конфигурации лежат **замаскированные** значения, а не открытый текст: `rclone obscure` обратим. Если вписать туда пароль руками, rclone прочитает его как уже замаскированный — и данные окажутся нечитаемыми. Меняйте пароль только через `rclone config`, не редактором.

**Запишите оба пароля на бумагу и положите отдельно от компьютера.** Это не вежливая рекомендация: без них данные не восстановит никто, включая вас (см. [§14](../4-ops/14-recovery.md)). Соль (`password2`) обязательна к сохранению наравне с основным паролем: без неё ключ не выводится.

## Проверка связи

```powershell
rclone lsd secret:
rclone mkdir secret:test
rclone rmdir secret:test
```

Если каталоги перечисляются — все три слоя работают.

## Монтирование

```powershell
rclone mount secret: X: `
  --vfs-cache-mode full `
  --vfs-cache-max-size 8G `
  --vfs-cache-min-free-space 10G `
  --vfs-cache-max-age 72h `
  --vfs-write-back 5s `
  --dir-cache-time 30s `
  --volname "Cloud" `
  --transfers 8 `
  --no-console `
  --log-file "$env:LOCALAPPDATA\rclone\mount.log" --log-level NOTICE
```

| ключ | смысл | последствие |
|---|---|---|
| `--vfs-cache-mode full` | файл целиком проходит через локальный кэш | работают программы, которые правят файл кусками (архиваторы, редакторы, базы); ценой записи в кэш и потом в сеть |
| `--vfs-cache-max-size 8G` | потолок кэша на `C:` | старое вытесняется, диск не переполняется |
| `--vfs-cache-min-free-space 10G` | не занимать последние гигабайты `C:` | кэш уступает место системе, а не наоборот |
| `--vfs-cache-max-age 72h` | сколько держать неиспользуемое | попадание в кэш = чтение на скорости локального диска |
| `--vfs-write-back 5s` | пауза перед отправкой после закрытия файла | редактор сохранил десять раз — уедет одна версия; но каждый крупный файл ждёт эти 5 секунд ([§11](../3-perf/11-bottlenecks.md)) |
| `--dir-cache-time 30s` | как долго верить списку файлов | компромисс: меньше — свежее, больше — меньше обращений |
| `--transfers 8` | сколько файлов отправлять параллельно | вдвое ускоряет пачки мелких файлов (замерено: 200 против 100 файлов/с) |
| `--log-level NOTICE` | журнал без перечисления файлов | INFO пишет открытые имена в файл журнала ([§9](../2-crypto/09-limits.md)) |

> [!NOTE]
> `--checkers` в командах для `mount` почти всегда лишний: на VFS он не влияет, параллельность задаёт `--transfers` — [документация rclone](https://rclone.org/commands/rclone_mount/). Флаг вреда не наносит, но и пользы не приносит.

## Если программы капризничают

Буква диска, созданная обычным (не повышенным) процессом, видна обычным программам, но не обязательно — запущенным от администратора. Обратное тоже верно: диск, созданный от администратора, не виден остальным.

- Если диск нужен только интерактивной работе — монтируйте из обычного сеанса, как в [§4](04-autostart.md).
- Если нужен всем и на уровне системы — либо `--network-mode` (диск представляется как сетевой, `\\server\share`), либо запуск от `SYSTEM` через WinFsp.Launcher, с явным `--config`.
- Проверка «видно ли от администратора» входит в приёмку ([§5](05-acceptance.md)).

---

### Источники

- [`rclone mount`](https://rclone.org/commands/rclone_mount/): режимы кэша, `--vfs-write-back`, оговорки про Windows.
- [`rclone sftp`](https://rclone.org/sftp/): `shell_type`, `disable_hashcheck`, `known_hosts_file`.
- [`rclone crypt`](https://rclone.org/crypt/): `filename_encryption`, `filename_encoding`.
- [WinFsp](https://winfsp.dev/), [WinFsp.Launcher](https://github.com/winfsp/winfsp/wiki/WinFsp-Service-Architecture).

---

<sub>[← §2](02-ssh.md) · [Оглавление](../README.md) · [§4. Автозапуск →](04-autostart.md)</sub>
