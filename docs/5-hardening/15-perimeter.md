# §15. Периметр

<sub>[Главная](../../README.md) · [Часть 5](README.md) · [§16. Наблюдение →](16-monitoring.md)</sub>

В установке из [§1–§5](../1-install/README.md) периметр открыт: firewall не настроен, `fail2ban` не работает, вход по паролю для основного пользователя разрешён, а у Pi есть глобальный IPv6-адрес. Внутри домашней сети это терпимо. Как только хранилище оказывается в зоне, куда может постучаться кто-то ещё, — это дыра.

Здесь три независимых шага. Каждый можно сделать отдельно, и каждый проверяется до того, как вы закроете текущую сессию.

## Шаг 1: firewall по умолчанию наружу закрыт

```bash
sudo cp /etc/nftables.conf /etc/nftables.conf.bak 2>/dev/null || true
sudo tee /etc/nftables.conf >/dev/null <<'CONF'
#!/usr/sbin/nft -f
flush ruleset

define lan = 192.168.1.0/24

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;

        iif lo accept
        ct state established,related accept
        ct state invalid drop

        udp sport 67 udp dport 68 accept          # DHCP: без этого можно остаться без адреса после перезагрузки
        ip protocol icmp accept
        ip6 nexthdr ipv6-icmp accept

        ip saddr $lan tcp dport 22 accept          # SSH только из своей сети, и только по IPv4
    }

    chain forward { type filter hook forward priority filter; policy drop; }
    chain output  { type filter hook output  priority filter; policy accept; }
}
CONF

sudo nft -c -f /etc/nftables.conf     # проверка синтаксиса, ничего не применяя
sudo nft -f /etc/nftables.conf        # применить
sudo systemctl enable nftables
```

Что это даёт: входящие соединения снаружи отбрасываются целиком, включая IPv6. Текущая SSH-сессия не рвётся — она попадает под `established,related`.

> [!WARNING]
> Если после применения SSH перестал подключаться — не закрывайте открытую сессию: `sudo nft flush ruleset` вернёт всё как было. Локальная консоль или монитор с клавиатурой должны быть под рукой, пока вы не убедились, что новый вход работает.

Вторая половина шага — не дать клиенту уйти по IPv6. В `rclone.conf` в разделе `pi` адрес должен быть IPv4-литералом (`host = 192.168.1.50`), иначе rclone может предпочесть AAAA-запись и упереться в закрытый порт.

## Шаг 2: вход по паролю выключен

Сначала убедитесь, что вход по ключу работает — **дважды**, в новой сессии:

```bash
ssh -i ~/.ssh/id_ed25519 pi@192.168.1.50 'echo ok'
```

Потом:

```bash
sudo tee /etc/ssh/sshd_config.d/00-hardening.conf >/dev/null <<'CONF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
CONF
sudo sshd -t && sudo systemctl reload ssh
```

Префикс `00-` не косметика: OpenSSH берёт **первое** встреченное значение, а Debian включает `sshd_config.d/*.conf` в алфавитном порядке. Файл `00-` выигрывает у любого `50-cloud-init.conf`, который иногда возвращает пароли обратно.

Проверка:

```bash
sudo sshd -T | grep -Ei '^(passwordauthentication|kbdinteractiveauthentication|permitrootlogin)'
# ожидается: no / no / no
```

Откат — из открытой сессии или с консоли:

```bash
sudo rm /etc/ssh/sshd_config.d/00-hardening.conf && sudo systemctl reload ssh
```

## Шаг 3 (по желанию): fail2ban

С ключами и закрытым периметром он почти не нужен: подбирать нечего. Польза остаётся одна — не засорять журнал и снять нагрузку от сканеров, если порт всё-таки виден.

```bash
sudo apt install -y fail2ban
sudo systemd-analyze security fail2ban.service --no-pager | head -3   # необязательно
sudo fail2ban-client status sshd                                      # тюрьма sshd активна
```

Настройки держите в `jail.local`, а не в `jail.conf`: последний перезаписывается при обновлении пакета.

## Чек-лист периметра

| что | команда | ожидание |
|---|---|---|
| снаружи не стучится | `nmap -Pn -p22 <внешний-IP>` с чужой сети | `filtered` |
| из своей сети SSH работает | `ssh -i <ключ> pi@<IP> 'echo ok'` | `ok` |
| пароль отключён | `sshd -T \| grep passwordauthentication` | `no` |
| root не входит | `sshd -T \| grep permitrootlogin` | `no` |
| правила живут после перезагрузки | `reboot`, затем `nft list ruleset` | правила на месте |
| IPv6 не открыт | `nmap -6 -Pn -p22 <адрес>` | `filtered` |

---

### Источники

- [nftables wiki](https://wiki.nftables.org/wiki-nftables/index.php/Main_Page), [`nft(8)`](https://www.mankier.com/8/nft).
- [`sshd_config(5)`](https://man.openbsd.org/sshd_config): порядок применения значений, `Include`.
- [fail2ban](https://github.com/fail2ban/fail2ban).
- [Raspberry Pi: IPv6 и сеть](https://www.raspberrypi.com/documentation/computers/configuration.html).

---

<sub>[← Часть 5](README.md) · [Оглавление](../README.md) · [§16. Наблюдение →](16-monitoring.md)</sub>
