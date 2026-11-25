# §10. Замеры

<sub>[Главная](../../README.md) · [Часть 3](README.md) · [§11. Где узкое место →](11-bottlenecks.md)</sub>

## Конфигурация

| | |
|---|---|
| Одноплатник | Raspberry Pi 5 Model B Rev 1.1, 2 ГБ ОЗУ |
| Система | Debian 13 (trixie), ядро 6.18.34, aarch64 |
| Диск | NVMe 1 ТБ (HS-SSD-FUTURE 1024G), ext4 |
| Шина | PCIe 5 GT/s ×1 (Gen 2 ×1), штатный режим Pi 5 |
| Сеть | eth0, 1000 Мбит/с, полный дуплекс, 2 мс в локальной сети |
| Клиент | Windows 11 Pro 25H2, rclone v1.75.0, WinFsp 2.1.25156 |

## Методика

Каждая строка — отдельный прогон. Там, где это имеет смысл, кэши обойдены (`oflag=direct` / `iflag=direct`). Объём 0.5–2 ГБ, чтобы разгон и завершение не искажали картину. Это одиночные прогоны, а не средние: разброс между запусками — единицы процентов, и на выводы он не влияет.

## Слой за слоем

| что меряем | команда | результат |
|---|---|---|
| SSD на Pi, запись мимо кэша | `dd if=/dev/zero of=... bs=1M count=2048 oflag=direct conv=fsync` | **430 МБ/с** |
| SSD на Pi, чтение мимо кэша | `dd if=... of=/dev/null bs=1M iflag=direct` | **460 МБ/с** |
| канал SSH, Windows → Pi | `dd if=/dev/zero bs=1M \| ssh pi 'dd of=/dev/null bs=1M'` | **103 МБ/с** |
| канал SSH, Pi → Windows | `ssh pi 'dd if=/dev/zero bs=1M' \| dd of=/dev/null` | **113 МБ/с** |

## Сквозной результат

| операция | результат |
|---|---|
| загрузка 1 ГБ, `rclone copyto` на `secret:` | **99 МиБ/с** (≈ 104 МБ/с) |
| скачивание 1 ГБ, `rclone copyto` с `secret:` | **105 МиБ/с** (≈ 110 МБ/с) |
| запись 1 ГБ через диск `X:` (как видит проводник) | **65 МБ/с** |
| холодное чтение 407 МБ через `X:` (файла нет в кэше) | **108 МБ/с** |
| повторное чтение того же файла (уже в кэше на `C:`) | **≈ 2100 МБ/с** |
| 200 файлов по 64 КБ, 8 потоков | 1 с — **≈ 200 файлов/с** |
| 200 файлов по 64 КБ, 1 поток | 2 с — **≈ 100 файлов/с** |

## Как переснять у себя

```bash
# диск на Pi
dd if=/dev/zero of=/srv/cloud/data/dd.bin bs=1M count=2048 oflag=direct conv=fsync
dd if=/srv/cloud/data/dd.bin of=/dev/null bs=1M iflag=direct
rm /srv/cloud/data/dd.bin

# канал: с клиента на Pi и обратно
dd if=/dev/zero bs=1M count=1024 | ssh pi 'dd of=/dev/null bs=1M'
```

```powershell
# сквозной путь: шифрование + сеть + диск
rclone copyto D:\big.bin secret:_bench\big.bin --stats-one-line
rclone copyto secret:_bench\big.bin D:\big-back.bin --stats-one-line
rclone deletefile secret:_bench\big.bin

# запись через букву диска: мерить до появления файла на Pi, а не до закрытия окна копирования
Measure-Command { Copy-Item D:\big.bin X:\big.bin }
```

Осторожно с последней строкой: `Copy-Item` возвращает управление, когда файл записан в локальный кэш. Отправка на Pi идёт после закрытия и паузы `--vfs-write-back`. Поэтому «скорость записи через диск» — это два разных числа, и путать их легко ([§11](11-bottlenecks.md)).

---

### Источники

- [`dd(1)`](https://man7.org/linux/man-pages/man1/dd.1.html), флаги `direct`/`of`/`if`.
- [`rclone copyto`](https://rclone.org/commands/rclone_copyto/), [`rclone about`](https://rclone.org/commands/rclone_about/).
- [`rclone mount`](https://rclone.org/commands/rclone_mount/): `--vfs-write-back`, режимы кэша.

---

<sub>[← Часть 3](README.md) · [Оглавление](../README.md) · [§11. Где узкое место →](11-bottlenecks.md)</sub>
