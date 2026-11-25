# §4. Автозапуск, который переживает выключенную Pi

<sub>[Главная](../../README.md) · [Часть 1](README.md) · [← §3](03-windows.md) · [§5. Приёмка →](05-acceptance.md)</sub>

Монтирование — долгоживущий процесс, а не команда. Поэтому его место — планировщик заданий, и у этого решения есть два подводных камня, из-за которых «настроенный автозапуск» тихо перестаёт работать: лимит времени выполнения и однократность попытки.

## Что не так с наивной настройкой

```powershell
# так делать не надо
Register-ScheduledTask -TaskName 'RcloneCloudMount' -Action $action -Trigger $trigger -RunLevel Limited
```

Три проблемы:

1. **Лимит выполнения.** У задания по умолчанию задано «остановить, если выполняется дольше трёх суток». Монтирование живёт неделями — значит, на четвёртый день диск исчезнет без внятной причины.
2. **Одна попытка.** Если Pi в момент входа в систему выключена, rclone завершится с ошибкой и больше не запустится до ручного вмешательства.
3. **Нет условий питания.** На ноутбуке задание может не стартовать от батареи, а при переходе на неё — остановиться.

## Правильная настройка

Обёртка, которая ждёт Pi и перезапускает rclone после падения. Файл `%USERPROFILE%\rclone-mount.ps1`:

```powershell
$rclone = 'C:\Program Files\rclone\rclone.exe'
$log    = "$env:LOCALAPPDATA\rclone\mount.log"

# ждём, пока хранилище ответит: до 10 минут, шаг 15 секунд
$deadline = (Get-Date).AddMinutes(10)
while ((Get-Date) -lt $deadline) {
    & $rclone lsd secret: 2>$null | Out-Null
    if ($LASTEXITCODE -eq 0) { break }
    Start-Sleep -Seconds 15
}

# держим монтирование: упало — поднимаем через 20 секунд
while ($true) {
    & $rclone mount secret: X: `
        --vfs-cache-mode full --vfs-cache-max-size 8G --vfs-cache-min-free-space 10G `
        --vfs-cache-max-age 72h --vfs-write-back 5s --dir-cache-time 30s `
        --volname "Cloud" --transfers 8 --no-console --log-level NOTICE `
        --log-file $log
    Start-Sleep -Seconds 20
}
```

Задание:

```powershell
$action = New-ScheduledTaskAction -Execute 'powershell.exe' `
  -Argument '-NoProfile -ExecutionPolicy Bypass -WindowStyle Hidden -File "%USERPROFILE%\rclone-mount.ps1"'

$trigger = New-ScheduledTaskTrigger -AtLogOn
$trigger.Delay = 'PT20S'          # сеть успевает подняться раньше попытки монтирования

$settings = New-ScheduledTaskSettingsSet `
  -ExecutionTimeLimit ([TimeSpan]::Zero) `
  -RestartCount 999 -RestartInterval (New-TimeSpan -Minutes 1) `
  -MultipleInstances IgnoreNew `
  -AllowStartIfOnBatteries -DontStopIfGoingOnBatteries `
  -StartWhenAvailable

Register-ScheduledTask -TaskName 'RcloneCloudMount' `
  -Action $action -Trigger $trigger -Settings $settings -RunLevel Limited
```

| настройка | зачем |
|---|---|
| `-ExecutionTimeLimit ([TimeSpan]::Zero)` | снимает лимит: без него задание убивается через трое суток |
| `-RestartCount` / `-RestartInterval` | страховка на случай, если обёртка всё-таки завершится |
| `-MultipleInstances IgnoreNew` | не запускать второй rclone поверх первого |
| `-AllowStartIfOnBatteries` / `-DontStopIfGoingOnBatteries` | не зависеть от питания |
| `-StartWhenAvailable` | догнать пропущенный запуск после сна/гибернации |
| `-RunLevel Limited` | монтировать без повышения прав: диск, созданный от администратора, не виден обычным программам |

`-WindowStyle Hidden` скрывает окно PowerShell, но не окно консоли, которое может мигнуть при старте; `--no-console` внутри скрывает уже сам rclone.

## Проверка

```powershell
Get-ScheduledTask RcloneCloudMount | Get-ScheduledTaskInfo    # LastTaskResult = 0, NextRunTime задан
Start-ScheduledTask RcloneCloudMount
# в течение 30 секунд, циклом с опросом:
while (-not (Test-Path 'X:\')) { Start-Sleep -Seconds 2 }
```

WinFsp поднимает том не мгновенно: одной паузы в 8 секунд не хватает, ожидание должно быть циклом.

Дальше — главная проверка, которую обычно пропускают: **выключите Pi, перезагрузите Windows и включите Pi обратно**. Диск должен появиться сам, без ручного вмешательства. Если нет — обёртка не отработала, смотрите `mount.log`.

## Вариант «диск для всех»

Если диск нужен службам и другим пользователям, монтируют от `SYSTEM` — через [WinFsp.Launcher](https://github.com/winfsp/winfsp/wiki/WinFsp-Service-Architecture) или задание с `-User 'SYSTEM'`. Два последствия: rclone не найдёт конфигурацию из вашего профиля (нужен явный `--config`), а владельцем тома станет `SYSTEM` — права на запись для остальных определяются группой/others.

---

### Источники

- [`Register-ScheduledTask`](https://learn.microsoft.com/powershell/module/scheduledtasks/register-scheduledtask), [`New-ScheduledTaskSettingsSet`](https://learn.microsoft.com/powershell/module/scheduledtasks/new-scheduledtasksettingsset).
- [`rclone mount`](https://rclone.org/commands/rclone_mount/): оговорки про Windows и буквы дисков.
- [WinFsp.Launcher](https://github.com/winfsp/winfsp/wiki/WinFsp-Service-Architecture).

---

<sub>[← §3](03-windows.md) · [Оглавление](../README.md) · [§5. Приёмка →](05-acceptance.md)</sub>
