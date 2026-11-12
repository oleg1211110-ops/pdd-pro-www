# pdd-pro.kz — готовый сайт ПДД PRO

Здесь лежит **собранный** сайт pdd-pro.kz. Руками ничего не править: всё перезаписывается при каждой сборке.
Исходники (база правил и вопросов, генератор) — в закрытом репозитории Олега: `pdd/kz/`, `tools/pdd_site.py`, `tools/pdd_publish.py`.

- `site/` — сайт целиком: статические HTML-страницы, PHP и база не нужны.
- Адреса: `/ru/`, `/kk/`, `/en/`, `/<яз>/pdd/punkt-103/` (папка с `index.html`), `sitemap.xml`, `robots.txt`.
- Пока идёт предпросмотр, во всех страницах `noindex`, а в `robots.txt` — `Disallow: /`.

## Сервер: забирает сайт сам (ключей нет)

Раз в 5 минут сервер скачивает этот репозиторий по HTTPS и, если что-то изменилось, выкладывает `site/` в папку сайта.
Пароли и ключи не нужны: репозиторий открытый, только чтение.

`/usr/local/bin/pdd-pull.sh`:

```sh
#!/bin/sh
# pdd-pro.kz: забрать готовый сайт из github.com/oleg1211110-ops/pdd-pro-www и выложить, если изменился
set -eu
REPO=https://github.com/oleg1211110-ops/pdd-pro-www.git
SRC=/srv/pdd-pro.kz-src        # клон репозитория
DST=/var/www/pdd-pro.kz         # root в nginx
if [ -d "$SRC/.git" ]; then OLD=$(git -C "$SRC" rev-parse HEAD); else git clone -q --depth 1 "$REPO" "$SRC"; OLD=none; fi
git -C "$SRC" fetch -q --depth 1 origin main
git -C "$SRC" reset -q --hard FETCH_HEAD
NEW=$(git -C "$SRC" rev-parse HEAD)
[ "$OLD" = "$NEW" ] && [ -f "$DST/index.html" ] && exit 0
[ -f "$SRC/site/index.html" ] || { echo "pdd-pro.kz: в репозитории нет site/index.html — не выкладываю" >&2; exit 1; }
mkdir -p "$DST"
rsync -a --delete "$SRC/site/" "$DST/"
echo "pdd-pro.kz: выложено $NEW"
```

`/etc/systemd/system/pdd-pull.service` и `/etc/systemd/system/pdd-pull.timer`:

```ini
[Unit]
Description=pdd-pro.kz: забрать готовый сайт

[Service]
Type=oneshot
# или пользователь сайта, которому принадлежат SRC и DST
User=www-data
ExecStart=/usr/local/bin/pdd-pull.sh
```

```ini
[Unit]
Description=pdd-pro.kz: проверять обновления каждые 5 минут

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

```sh
apt install -y git rsync      # если ещё не стоят
chmod +x /usr/local/bin/pdd-pull.sh
mkdir -p /srv/pdd-pro.kz-src /var/www/pdd-pro.kz && chown www-data: /srv/pdd-pro.kz-src /var/www/pdd-pro.kz
systemctl daemon-reload && systemctl enable --now pdd-pull.timer
systemctl start pdd-pull.service && journalctl -u pdd-pull.service -n 5   # первая выкладка сразу
```

## nginx

Обычный статический сайт: `root /var/www/pdd-pro.kz; index index.html; location / { try_files $uri $uri/ =404; }`,
`www.pdd-pro.kz` → 301 на `pdd-pro.kz`, HTTPS (Let's Encrypt). Кэш: HTML — короткий (минуты), остальное — дольше.
Главная `/` — выбор языка (noindex), основные страницы — `/ru/`, `/kk/`, `/en/`.
