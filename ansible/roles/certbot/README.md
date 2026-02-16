# Certbot

Installs Certbot (Let's Encrypt)
https://certbot.eff.org/

https://eff-certbot.readthedocs.io/en/latest/using.html#manual

Auto-renew is handled by a snap timer. View with `systemctl list-timers` and test with `certbot renew --dry-run`.

This will actually update the magic acme records on acme-dns
```bash
sudo certbot renew --dry-run
```

## Requirements

* Certbot requires snapd be installed

## View snap services

```bash
snap services certbot.renew
snap logs certbot
```
