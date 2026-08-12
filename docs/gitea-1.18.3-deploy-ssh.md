# Gitea 1.18.3 + деплой через SSH (`git pull`)

Самый простой вариант: код лежит на сервере приложения, вы заходите по SSH и обновляете его вручную.

```
[разработчик] --git push--> [Gitea 1.18.3]
                                    ↑
[сервер приложения] --git pull------+
```

---

## 1. Один раз: ключ и клон на сервере

### 1.1. SSH-ключ для доступа к Gitea

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

### 1.2. Клон проекта

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
git pull origin main
```

Если сервис нужно перезапустить:

```bash
# пример
sudo systemctl restart myapp
# или
docker compose up -d --build
```

Готово.

---

## 3. Одной командой с вашей машины

```bash
ssh user@app-server 'cd /var/www/app && git pull origin main && sudo systemctl restart myapp'
```

---

## 4. Если `git pull` ругается на локальные правки

На сервере не должно быть незакоммиченных изменений в деплой-каталоге. Если нужно жёстко взять то, что в Gitea:

```bash
cd /var/www/app
git fetch origin main
git reset --hard origin/main
```

`reset --hard` сотрёт локальные правки в этом каталоге.

---

## 5. Чеклист

- [ ] Deploy key добавлен в Gitea
- [ ] `ssh -T git@gitea.example.com` проходит
- [ ] Репозиторий склонирован на сервер
- [ ] После push: `cd … && git pull origin main` обновляет код

Автоматический webhook для этого варианта не нужен.
