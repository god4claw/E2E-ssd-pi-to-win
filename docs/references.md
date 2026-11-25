# Источники

Ссылки на документацию, по которой сверялись команды, форматы и ограничения. Версии — на дату снятия замеров (ноябрь 2026).

---

## rclone

- Общая документация и флаги — [rclone.org/docs](https://rclone.org/docs/), полный список флагов — [rclone.org/flags](https://rclone.org/flags/).
- **crypt** — [rclone.org/crypt](https://rclone.org/crypt/): XSalsa20-Poly1305 (NaCl secretbox), scrypt для ключа, `filename_encryption`, `directory_name_encryption`, `filename_encoding` (в том числе base32768).
- **SFTP** — [rclone.org/sftp](https://rclone.org/sftp/): `shell_type`, `disable_hashcheck`, `md5sum_command`, `known_hosts_file`, `chunk_size`.
- **mount** — [rclone.org/commands/rclone_mount](https://rclone.org/commands/rclone_mount/): режимы кэша (`off|minimal|writes|full`), `--vfs-write-back`, `--vfs-cache-max-size`, `--vfs-cache-min-free-space`, `--transfers` и явное указание, что `--checkers` на VFS не влияет; поведение WinFsp на Windows и оговорка про буквы, созданные от администратора.
- **config create** — [rclone.org/commands/rclone_config_create](https://rclone.org/commands/rclone_config_create/): автоматическое маскирование паролей, флаги `--obscure` / `--no-obscure` и предупреждение про 22+ символа base64.
- **obscure** — [rclone.org/commands/rclone_obscure](https://rclone.org/commands/rclone_obscure/): маскирование обратимо, это не шифрование.

## OpenSSH

- `sshd_config(5)` — [man.openbsd.org/sshd_config](https://man.openbsd.org/sshd_config): `Match`, `ChrootDirectory` (требования к владельцу и правам), `ForceCommand`, `AuthorizedKeysFile`, `PermitTTY`, `AllowTcpForwarding`.
- `authorized_keys(5)` — [man.openbsd.org/authorized_keys](https://man.openbsd.org/authorized_keys): опция `restrict` и её составляющие.
- `sftp-server(8)` — [man.openbsd.org/sftp-server](https://man.openbsd.org/sftp-server): `internal-sftp`, ключ `-d`.

## Windows

- **WinFsp** — [winfsp.dev](https://winfsp.dev/), [репозиторий](https://github.com/winfsp/winfsp): драйвер пользовательской ФС, FUSE-совместимость, служба WinFsp.Launcher.
- **Планировщик заданий** — [`Register-ScheduledTask`](https://learn.microsoft.com/powershell/module/scheduledtasks/register-scheduledtask), [`New-ScheduledTaskSettingsSet`](https://learn.microsoft.com/powershell/module/scheduledtasks/new-scheduledtasksettingsset): `ExecutionTimeLimit` (по умолчанию — трое суток), `RestartCount`, `MultipleInstances`, условия питания.
- **Длинные пути** — [Maximum Path Length Limitation](https://learn.microsoft.com/windows/win32/fileio/maximum-file-path-limitation): `LongPathsEnabled`, префикс `\\?\`.

## Файловая система и диск

- **ext4** — [kernel.org: ext4](https://www.kernel.org/doc/html/latest/filesystems/ext4/index.html), `mke2fs(8)` ([man7](https://man7.org/linux/man-pages/man8/mke2fs.8.html)): `-m` (резерв под root), `-L`. Предел длины имени компонента — 255 байт.
- **fstab(5)** — [man7.org/linux/man-pages/man5/fstab.5.html](https://man7.org/linux/man-pages/man5/fstab.5.html): `nofail`, `x-systemd.*`, `noatime`.
- **fstrim** — `fstrim(8)`, `fstrim.timer` в systemd: TRIM по расписанию вместо опции `discard`.
- **smartmontools** — [smartmontools.org](https://www.smartmontools.org/), `smartctl(8)`: атрибуты NVMe (`Percentage_Used`, `Media and Data Integrity Errors`, температура).

## Сеть и периметр

- **nftables** — [wiki.nftables.org](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page), `nft(8)`.
- **fail2ban** — [github.com/fail2ban/fail2ban](https://github.com/fail2ban/fail2ban).
- **Raspberry Pi 5** — [документация](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html): PCIe Gen 2 ×1 по умолчанию, `dtparam=pciex1_gen=3`, требования к питанию.

## Прочее

- **NaCl / secretbox** — [документация libsodium](https://doc.libsodium.org/secret-key_cryptography/secretbox), XSalsa20-Poly1305.
- **scrypt** — RFC 7914.
- **base32768** — [github.com/qntm/base32768](https://github.com/qntm/base32768): 15 бит на символ, компактнее base32 почти втрое.

---

<sub>[← Оглавление](README.md) · [На главную](../README.md)</sub>
