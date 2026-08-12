# Gitea 1.18.3 + деплой через SSH (`git pull`)

Самый простой вариант: код лежит на сервере приложения, после пуша заходите по SSH и делаете `git pull`.

```
[разработчик] --git push--> [Gitea 1.18.3]
                                    ↑
[сервер приложения] --git pull------+
```

---

## 1. Один раз: ключ и клон на сервере

### SSH-ключ для Gitea

На сервере приложения:

```bash
ssh-keygen -t ed25519 -C "server-deploy" -f ~/.ssh/gitea_deploy -N ""
cat ~/.ssh/gitea_deploy.pub
```

В Gitea: репозиторий → **Settings → Deploy Keys → Add Key**  
- вставьте содержимое `.pub`  
- Write Access не нужен

`~/.ssh/config`:

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

### Клон проекта

```bash
cd /var/www   # или ваш каталог
git clone git@gitea.example.com:org/app.git
cd app
git checkout main
```

Дальше настройте приложение (env, зависимости, systemd/nginx) как обычно.

---

## 2. Каждый деплой (вручную)

После `git push` в Gitea:

```bash
ssh user@app-server
cd /var/www/app
git pull --ff-only origin main
```

Если сервис нужно перезапустить:

```bash
sudo systemctl restart myapp
# или
docker compose up -d --build
```

Готово.

---

## 3. Одной командой с вашей машины

```bash
ssh user@app-server 'cd /var/www/app && git pull --ff-only origin main && sudo systemctl restart myapp'
```

---

## 4. Если `git pull` ругается

На сервере не должно быть незакоммиченных правок в деплой-каталоге. Чтобы жёстко взять то, что в Gitea:

```bash
cd /var/www/app
git fetch origin main
git reset --hard origin/main
```

`reset --hard` сотрёт локальные правки в этом каталоге.

| Симптом | Что сделать |
|---------|-------------|
| `Permission denied (publickey)` | deploy key / `IdentityFile` в `~/.ssh/config` |
| local changes would be overwritten | убрать правки или `reset --hard` |
| wrong branch | `git checkout main` |

---

## 5. Чеклист

- [ ] Deploy key добавлен в Gitea
- [ ] `ssh -T git@gitea.example.com` проходит
- [ ] Репозиторий склонирован на сервер
- [ ] После push: `cd … && git pull --ff-only origin main` обновляет код

Webhook и автодеплой здесь не нужны.
