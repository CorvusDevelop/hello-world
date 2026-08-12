# Gitea 1.18.3 + автодеплой после push

Минимально рабочая схема: **Gitea на своём сервере** шлёт webhook на **сервер приложения**, тот проверяет подпись и делает `git pull` + перезапуск сервиса.

> **Важно для 1.18.3:** Gitea Actions появились только в **1.19**. В 1.18.3 встроенного CI нет — используйте webhook.

---

## 1. Архитектура

```
[разработчик] --git push--> [Gitea 1.18.3]
                                |
                                | POST /hooks/gitea (push event)
                                v
                         [сервер приложения]
                         webhook listener
                                |
                                | git fetch + reset/pull
                                | restart app
                                v
                         [рабочий сайт/сервис]
```

Два сервера:
| Роль | Что крутится |
|------|----------------|
| Gitea-сервер | Gitea 1.18.3 + репозиторий |
| Deploy-сервер | код приложения + webhook-listener |

Сеть: Gitea должна достучаться до deploy-сервера по HTTP/HTTPS (порт listener’а).

---

## 2. Подготовка на сервере приложения

### 2.1. Пользователь и каталог деплоя

```bash
sudo useradd -m -s /bin/bash deploy
sudo mkdir -p /var/www/app
sudo chown deploy:deploy /var/www/app
```

### 2.2. Deploy-ключ (read-only) в Gitea

На deploy-сервере от пользователя `deploy`:

```bash
sudo -u deploy -i
ssh-keygen -t ed25519 -C "deploy@app" -f ~/.ssh/gitea_deploy -N ""
cat ~/.ssh/gitea_deploy.pub
```

В Gitea: репозиторий → **Settings → Deploy Keys → Add Key**  
- Title: `app-server`  
- Key: содержимое `gitea_deploy.pub`  
- **Write Access** — не включать (достаточно чтения)

`~/.ssh/config` у `deploy`:

```
Host gitea.example.com
  HostName gitea.example.com
  User git
  IdentityFile ~/.ssh/gitea_deploy
  IdentitiesOnly yes
```

Проверка:

```bash
ssh -T git@gitea.example.com
```

### 2.3. Клон репозитория

```bash
cd /var/www
sudo -u deploy git clone git@gitea.example.com:org/app.git app
cd /var/www/app
# checkout нужной ветки, например main
git checkout main
```

Дальше приложение должно уметь стартовать из этого каталога (systemd, docker compose и т.д.).

---

## 3. Webhook listener (минимальный)

Простой HTTP-сервер на Python 3: принимает push от Gitea, проверяет HMAC-секрет, деплоит только ветку `main`.

### 3.1. Файлы

`/opt/webhook-deploy/deploy.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail

APP_DIR=/var/www/app
BRANCH=main
LOCK=/tmp/deploy.lock

exec 9>"$LOCK"
flock -n 9 || { echo "deploy already running"; exit 0; }

cd "$APP_DIR"
git fetch origin "$BRANCH"
git reset --hard "origin/$BRANCH"

# --- подставьте свой шаг ---
# npm ci && npm run build
# docker compose up -d --build
# sudo systemctl restart myapp

echo "deploy ok: $(git rev-parse --short HEAD)"
```

`/opt/webhook-deploy/server.py`:

```python
#!/usr/bin/env python3
import hashlib
import hmac
import json
import os
import subprocess
from http.server import BaseHTTPRequestHandler, HTTPServer

SECRET = os.environ["WEBHOOK_SECRET"].encode()
BRANCH = os.environ.get("DEPLOY_BRANCH", "refs/heads/main")
DEPLOY_SCRIPT = os.environ.get("DEPLOY_SCRIPT", "/opt/webhook-deploy/deploy.sh")
HOST = os.environ.get("LISTEN_HOST", "0.0.0.0")
PORT = int(os.environ.get("LISTEN_PORT", "9000"))


def verify(signature_header: str, body: bytes) -> bool:
    # Gitea шлёт: X-Gitea-Signature: <hex hmac-sha256>
    if not signature_header:
        return False
    expected = hmac.new(SECRET, body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature_header.strip())


class Handler(BaseHTTPRequestHandler):
    def do_POST(self):
        if self.path.rstrip("/") != "/hooks/gitea":
            self.send_response(404)
            self.end_headers()
            return

        length = int(self.headers.get("Content-Length", "0"))
        body = self.rfile.read(length)
        sig = self.headers.get("X-Gitea-Signature", "")

        if not verify(sig, body):
            self.send_response(401)
            self.end_headers()
            self.wfile.write(b"bad signature")
            return

        try:
            payload = json.loads(body.decode("utf-8"))
        except json.JSONDecodeError:
            self.send_response(400)
            self.end_headers()
            return

        ref = payload.get("ref", "")
        if ref != BRANCH:
            self.send_response(200)
            self.end_headers()
            self.wfile.write(f"ignored ref {ref}".encode())
            return

        # деплой в фоне, чтобы Gitea быстро получила 200
        subprocess.Popen([DEPLOY_SCRIPT], stdout=subprocess.DEVNULL, stderr=subprocess.STDOUT)

        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"accepted")

    def log_message(self, fmt, *args):
        print("[%s] %s" % (self.log_date_time_string(), fmt % args))


if __name__ == "__main__":
    print(f"listening on {HOST}:{PORT}")
    HTTPServer((HOST, PORT), Handler).serve_forever()
```

Права:

```bash
sudo mkdir -p /opt/webhook-deploy
sudo cp deploy.sh server.py /opt/webhook-deploy/
sudo chmod 750 /opt/webhook-deploy/deploy.sh
sudo chmod 640 /opt/webhook-deploy/server.py
# deploy.sh должен уметь читать/писать /var/www/app
sudo chown -R deploy:deploy /opt/webhook-deploy
```

Если в `deploy.sh` нужен `systemctl restart`, добавьте sudoers-правило без пароля только на нужный юнит:

```bash
echo 'deploy ALL=(root) NOPASSWD: /bin/systemctl restart myapp' | sudo tee /etc/sudoers.d/deploy-myapp
sudo chmod 440 /etc/sudoers.d/deploy-myapp
```

### 3.2. systemd-юнит

`/etc/systemd/system/webhook-deploy.service`:

```ini
[Unit]
Description=Gitea webhook deploy listener
After=network.target

[Service]
Type=simple
User=deploy
Group=deploy
WorkingDirectory=/opt/webhook-deploy
Environment=WEBHOOK_SECRET=замените_на_длинный_секрет
Environment=DEPLOY_BRANCH=refs/heads/main
Environment=DEPLOY_SCRIPT=/opt/webhook-deploy/deploy.sh
Environment=LISTEN_HOST=127.0.0.1
Environment=LISTEN_PORT=9000
ExecStart=/usr/bin/python3 /opt/webhook-deploy/server.py
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now webhook-deploy
sudo systemctl status webhook-deploy
```

### 3.3. Nginx перед listener (рекомендуется)

Не открывайте 9000 наружу напрямую. Проксируйте через HTTPS:

```nginx
server {
    listen 443 ssl;
    server_name deploy.example.com;

    # ssl_certificate / ssl_certificate_key — ваши сертификаты

    location /hooks/gitea {
        proxy_pass http://127.0.0.1:9000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        # тело нужно целиком для HMAC
        proxy_request_buffering on;
        client_max_body_size 2m;
    }
}
```

Файрвол: с Gitea-сервера должен быть доступен только `443` (или ваш порт) на deploy-хосте.

---

## 4. Настройка webhook в Gitea 1.18.3

Репозиторий → **Settings → Webhooks → Add Webhook → Gitea**

| Поле | Значение |
|------|----------|
| Target URL | `https://deploy.example.com/hooks/gitea` |
| HTTP Method | `POST` |
| POST Content Type | `application/json` |
| Secret | тот же, что `WEBHOOK_SECRET` |
| Trigger On | **Push Events** |
| Branch Filter | `main` (если поле есть; иначе фильтр в listener) |
| Active | ✓ |

Сохраните → **Test Delivery**.

Ожидание:
- в Recent Deliveries — HTTP **200**
- в логах `journalctl -u webhook-deploy -f` — запрос принят
- для Test Delivery ref может быть синтетическим; полный цикл проверяйте реальным `git push`

---

## 5. Проверка end-to-end

На машине разработчика:

```bash
git clone git@gitea.example.com:org/app.git
cd app
echo "deploy test $(date -Is)" >> DEPLOY_CHECK.txt
git add DEPLOY_CHECK.txt
git commit -m "chore: test auto-deploy"
git push origin main
```

На deploy-сервере:

```bash
# webhook пришёл
journalctl -u webhook-deploy -n 50 --no-pager

# код обновился
sudo -u deploy git -C /var/www/app log -1 --oneline
cat /var/www/app/DEPLOY_CHECK.txt
```

---

## 6. Настройки Gitea, без которых webhook «молчит»

### 6.1. Исходящие webhook’и

В `app.ini` на Gitea-сервере:

```ini
[webhook]
ALLOWED_HOST_LIST = deploy.example.com, *.example.com
SKIP_TLS_VERIFY = false
```

После правок — рестарт Gitea.

`ALLOWED_HOST_LIST` обязателен, если цель не публичный «обычный» хост / если Gitea блокирует private IP. Для IP вроде `10.x` явно укажите хост/IP.

### 6.2. Docker ≥ 20.10.6

На Gitea 1.18.x в Docker известна поломка push-hooks при старом Docker Engine (< 20.10.6): **Test Delivery работает**, реальный push — нет. Обновите Docker на хосте Gitea.

### 6.3. Resync git hooks

Site Administration → **Resynchronize pre-receive, update and post-receive hooks of all repositories**.

Нужно, если после апгрейда activity/webhooks перестали срабатывать на push.

### 6.4. Сеть

С контейнера/процесса Gitea:

```bash
curl -vk https://deploy.example.com/hooks/gitea
```

Должен ответить ваш listener (хотя бы 401/404 без подписи — значит, маршрут живой).

---

## 7. Безопасность (минимум)

1. **HMAC-секрет** длинный (≥ 32 байт), один и тот же в Gitea и `WEBHOOK_SECRET`.
2. Listener только на `127.0.0.1`, снаружи — Nginx + TLS.
3. Deploy-ключ **read-only**.
4. В listener деплой **только** с нужного `ref` (`refs/heads/main`).
5. `flock` в `deploy.sh`, чтобы параллельные push не ломали checkout.
6. Не запускайте listener от root.
7. Логируйте delivery в Gitea (Recent Deliveries) и `journalctl`.

---

## 8. Типичные поломки

| Симптом | Что проверить |
|---------|----------------|
| Test Delivery 200, push — тишина | Docker ≥ 20.10.6; resync hooks; `ALLOWED_HOST_LIST` |
| 401 bad signature | Secret не совпал; Nginx меняет/обрезает body; неверный header (`X-Gitea-Signature`) |
| 200, код не обновился | другая ветка (`ref`); ошибка в `deploy.sh`; права на `/var/www/app` |
| Permission denied (publickey) | deploy key не добавлен / неверный `IdentityFile` |
| Gitea не достучится | firewall, DNS, TLS, webhook URL |

Подпись в 1.18.x: header **`X-Gitea-Signature`**, значение — hex HMAC-SHA256 от **сырого** тела запроса.

---

## 9. Вариант без Python: webhook → ssh

Если listener не хотите, можно держать на Gitea (или отдельном хосте) скрипт, который по webhook делает:

```bash
ssh deploy@app-server '/opt/webhook-deploy/deploy.sh'
```

Но нужен тот же HTTP-приёмник webhook’а. Чистый «только SSH без HTTP» из Gitea 1.18.3 штатно не делается — точка входа всё равно webhook (или внешний CI вроде Drone/Woodpecker).

Для полноценного CI на 1.18.3 обычно ставят **Woodpecker** или **Drone** рядом с Gitea. Для «просто выкатить после push» достаточно схемы из этого гайда.

---

## 10. Чеклист «минимально работает»

- [ ] Репозиторий склонирован на deploy-сервер под `deploy`
- [ ] Deploy key read-only добавлен в Gitea
- [ ] `deploy.sh` вручную отрабатывает: `sudo -u deploy /opt/webhook-deploy/deploy.sh`
- [ ] `webhook-deploy.service` active
- [ ] Nginx/TLS проксирует `/hooks/gitea`
- [ ] В Gitea webhook Active + Secret совпадает
- [ ] `ALLOWED_HOST_LIST` содержит deploy-хост
- [ ] Test Delivery → 200
- [ ] `git push origin main` обновляет `/var/www/app`

После этого каждый push в `main` автоматически выкатывает код на сервер приложения.
