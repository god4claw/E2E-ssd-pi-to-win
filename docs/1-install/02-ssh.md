# §2. Доступ только по ключу

<sub>[Главная](../../README.md) · [Часть 1](README.md) · [← §1](01-pi.md) · [§3. Windows →](03-windows.md)</sub>

Транспортный слой — единственное, что на Pi слушает сеть. Поэтому его настройка — не «разрешить вход», а «разрешить ровно один вход и ровно в одну папку».

Ключевая идея: учётная запись-хранитель не имеет ни пароля, ни оболочки, ни домашней папки с вашими правами. Даже полный перехват ключа даёт доступ только к шифротексту.

## Ключ

На **Windows** создайте отдельную пару — не переиспользуйте основную:

```powershell
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\id_ed25519_cloud -C "cloud@win"
```

Публичную половину (`id_ed25519_cloud.pub`) перенесите на Pi. Список ключей лежит **вне** домашней папки пользователя и принадлежит root: даже если что-то в хранилище пойдёт не так, дописать себе новый ключ через SFTP невозможно.

```bash
sudo mkdir -p /etc/ssh/authorized_keys.d
printf 'restrict ssh-ed25519 AAAA<...ваш ключ...> cloud@win\n' | sudo tee /etc/ssh/authorized_keys.d/cloud
sudo chown root:root /etc/ssh/authorized_keys.d/cloud
sudo chmod 644 /etc/ssh/authorized_keys.d/cloud
```

Префикс `restrict` перед ключом отключает сразу всё лишнее: проброс портов, агента, туннели, X11, выделение терминала. Это не замена настройкам `sshd` ниже, а второй независимый слой: даже если правило `Match` однажды отвалится, ключ не даст сделать из учётки туннель.

## Правило для sshd

Создайте `/etc/ssh/sshd_config.d/50-cloud.conf`:

```
# Хранилище: SFTP-only, заперт в /srv/cloud, без шелла и без пробросов.
Match User cloud
    ChrootDirectory /srv/cloud
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
```

| строка | что делает |
|---|---|
| `Match User cloud` | всё ниже действует только на этого пользователя, остальных не трогает |
| `ChrootDirectory /srv/cloud` | для него корень мира — эта папка, выше не выйти |
| `ForceCommand internal-sftp -d /data` | что бы ни попросил клиент, запускается встроенный SFTP и сразу в `/data`; отдельная программа `sftp-server` не запускается вовсе |
| `AuthorizedKeysFile /etc/ssh/authorized_keys.d/%u` | список ключей вне досягаемости пользователя |
| `PasswordAuthentication no` | пароль подобрать нельзя, его просто нет |
| `AllowTcpForwarding no` и соседи | учётку нельзя превратить в туннель во внутреннюю сеть |

Проверка и применение:

```bash
sudo sshd -t && sudo systemctl reload ssh
```

`sshd -t` проверяет синтаксис. Перезагружать конфигурацию без этой проверки — способ потерять SSH на удалённой машине.

> [!WARNING]
> Не закрывайте текущую сессию, пока не убедитесь, что новый вход работает. Держите открытым второй терминал: если правило сломало доступ, откат — `sudo rm /etc/ssh/sshd_config.d/50-cloud.conf && sudo systemctl reload ssh` из уже открытой сессии или с консоли.

## Проверка

```bash
# правило применилось к пользователю
sudo sshd -T -C user=cloud 2>/dev/null | grep -E 'chrootdirectory|forcecommand|permitopen|allowtcpforwarding|permittty'

# оболочки нет: команда не выполняется, соединение закрывается
ssh -i ~/.ssh/id_ed25519_cloud cloud@<IP> 'id'

# SFTP работает
sftp -i ~/.ssh/id_ed25519_cloud cloud@<IP> <<< 'pwd'
```

Ожидаемо: в первом выводе — `chrootdirectory /srv/cloud`, `forcecommand internal-sftp -d /data`, `allowtcpforwarding no`, `permittty no`; во втором — сообщение вида «This service allows sftp connections only»; в третьем — `/data`.

| симптом | причина |
|---|---|
| вход отбивается сразу после аутентификации | `/srv/cloud` не принадлежит root или доступен на запись группе (§1) |
| `ssh` даёт приглашение | правило `Match` не подхватилось: проверьте `Include` в `sshd_config` и `sshd -T` |
| SFTP видит пустоту | `-d /data` указывает не туда либо `DATA_DIR` не создан |
| ключ не принимается | файл в `authorized_keys.d` не 644 или принадлежит не root |

---

### Источники

- [`sshd_config(5)`](https://man.openbsd.org/sshd_config): `Match`, `ChrootDirectory`, `ForceCommand`, `AuthorizedKeysFile`.
- [`authorized_keys(5)`](https://man.openbsd.org/authorized_keys): опция `restrict`.
- [`sftp-server(8)`](https://man.openbsd.org/sftp-server): `internal-sftp`, ключ `-d`.

---

<sub>[← §1](01-pi.md) · [Оглавление](../README.md) · [§3. Windows →](03-windows.md)</sub>
