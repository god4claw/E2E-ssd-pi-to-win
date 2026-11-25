# §16. Наблюдение

<sub>[Главная](../../README.md) · [Часть 5](README.md) · [← §15](15-perimeter.md) · [§17. Бэкап шифротекста →](17-backup.md)</sub>

Сейчас о заполнении диска вы узнаете в момент отказа записи, а о смерти SSD — в момент, когда диск уже не читается. Наблюдение стоит одного скрипта и одного таймера.

## Что смотреть

| что | почему | как |
|---|---|---|
| свободное место | заполнение — самый частый отказ | `df -h /srv/cloud` |
| «жив ли диск» | отвалившийся NVMe не мешает Pi работать | `findmnt -n /srv/cloud` |
| здоровье SSD | износ, ошибки целостности, температура | `smartctl -a /dev/nvme0n1` |
| возраст копии | самая коварная поломка: копия есть, но полугодовой давности | дата последнего успешного бэкапа |
| журнал монтирования | ошибки отправки видны только там | `mount.log` |

## Скрипт

`/usr/local/sbin/cloud-health.sh`:

```bash
#!/bin/sh
set -eu
warn=0

findmnt -n /srv/cloud >/dev/null 2>&1 || { echo "cloud: диск не смонтирован"; warn=1; }
use=$(df --output=pcent /srv/cloud 2>/dev/null | tail -1 | tr -dc '0-9')
[ "${use:-100}" -lt 90 ] || { echo "cloud: занято ${use}%"; warn=1; }

smartctl -H /dev/nvme0n1 | grep -q PASSED || { echo "cloud: SMART не PASSED"; warn=1; }
used=$(smartctl -A /dev/nvme0n1 | awk '/Percentage_Used/ {gsub("%","",$NF); print $NF; exit}')
[ "${used:-0}" -lt 90 ] || { echo "cloud: износ ${used}%"; warn=1; }

errs=$(smartctl -A /dev/nvme0n1 | awk '/Media and Data Integrity Errors/ {print $NF; exit}')
[ "${errs:-0}" -eq 0 ] || { echo "cloud: ошибки целостности данных: ${errs}"; warn=1; }

[ "$warn" -eq 0 ] && echo "cloud: ok · занято ${use}% · износ ${used:-?}%"
exit "$warn"
```

```bash
sudo chmod 755 /usr/local/sbin/cloud-health.sh
sudo /usr/local/sbin/cloud-health.sh
```

Выход `0` — всё в порядке, `1` — есть замечание; текст замечания уходит в журнал.

## Таймер

`/etc/systemd/system/cloud-health.service`:

```ini
[Unit]
Description=Проверка хранилища cloud

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/cloud-health.sh
```

`/etc/systemd/system/cloud-health.timer`:

```ini
[Unit]
Description=Ежедневная проверка хранилища

[Timer]
OnCalendar=daily
RandomizedDelaySec=1h
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now cloud-health.timer
systemctl list-timers cloud-health.timer --no-pager
journalctl -u cloud-health --no-pager -n 20
```

`Persistent=true` — если Pi была выключена, проверка выполнится при следующем старте, а не будет пропущена.

## Журнал на клиенте

rclone не умеет ротацию, а журнал растёт. Два правила:

1. `--log-level NOTICE` — без открытых имён файлов в журнале ([§9](../2-crypto/09-limits.md)).
2. Обрезка по размеру — маленькое задание раз в неделю:

```powershell
$log = "$env:LOCALAPPDATA\rclone\mount.log"
if ((Test-Path $log) -and (Get-Item $log).Length -gt 50MB) {
    Move-Item $log "$log.old" -Force
}
```

## Что считать тревогой

- занято больше 90 %;
- `findmnt` пуст — диск отвалился;
- износ SSD больше 90 % или появились ошибки целостности;
- копия старше двух недель;
- в `mount.log` повторяются `failed to upload` или `corrupted on transfer`.

Первые три ловит таймер, четвёртое — [§17](17-backup.md), пятое — глазами при разборе очередного «странно работает».

---

### Источники

- [smartmontools](https://www.smartmontools.org/), `smartctl(8)`; атрибуты NVMe: `Percentage_Used`, `Media and Data Integrity Errors`, `Available Spare`.
- [`systemd.timer(5)`](https://www.freedesktop.org/software/systemd/man/systemd.timer.html), [`systemd.service(5)`](https://www.freedesktop.org/software/systemd/man/systemd.service.html).
- [`df(1)`](https://man7.org/linux/man-pages/man1/df.1.html), [`findmnt(8)`](https://man7.org/linux/man-pages/man8/findmnt.8.html).

---

<sub>[← §15](15-perimeter.md) · [Оглавление](../README.md) · [§17. Бэкап шифротекста →](17-backup.md)</sub>
