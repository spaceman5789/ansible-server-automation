# Проект: один сервер с Nginx + Docker + HTTPS

Этот проект делает так, чтобы на одной виртуальной машине все работало само:
- ставит нужные программы,
- скачивает твой проект,
- запускает контейнеры,
- ставит Nginx перед приложением,
- включает HTTPS,
- открывает нужные порты в firewall.

Все это делает Ansible одной командой.

---

## 1. Что нужно заранее

На твоем ноутбуке (где запускаешь Ansible):
- `ansible`
- `sshpass` (если вход по паролю)

На виртуалке:
- Debian/Ubuntu
- доступ по SSH

---

## 2. Куда вписать данные сервера

Открой файл `ansible/inventory`.

Пример:

```ini
[web]
app1 ansible_host=192.168.64.2 ansible_user=debian ansible_password=debian ansible_become_password=debian app_repo=https://github.com/spaceman5789/Nginx--SQL-Grafana-Prom app_exec_start="/usr/bin/docker-compose up -d" app_exec_stop="/usr/bin/docker-compose down" app_port=8080
```

Что важно:
- `ansible_host` — IP виртуалки
- `ansible_user` / `ansible_password` — логин и пароль SSH
- `ansible_become_password` — пароль для `sudo`
- `app_repo` — ссылка на твой репозиторий

---

## 3. Одна команда запуска

Из корня проекта запусти:

```bash
ansible-galaxy collection install -r ansible/requirements.yml
ansible-playbook -i ansible/inventory ansible/site.yml -vv
```

---

## 4. Что делает playbook по шагам

1. Ставит пакеты: `nginx`, `git`, `docker`, `docker-compose`, `ufw`, `certbot`
2. Создает пользователя `deploy`
3. Клонирует твой репозиторий в `/opt/demo-app`
4. Создает systemd-сервис `demo-app`
5. Запускает `docker-compose up -d`
6. Настраивает Nginx как reverse proxy
7. Открывает порты `22`, `80`, `443`
8. Делает HTTPS:
   - `selfsigned` (по умолчанию)
   - `letsencrypt` (если есть домен)

---

## 5. Как проверить, что все живо

Проверь сервис:

```bash
ansible app1 -i ansible/inventory -b -m shell -a "systemctl status demo-app --no-pager -l"
```

Проверь контейнеры:

```bash
ansible app1 -i ansible/inventory -b -m shell -a "cd /opt/demo-app && docker-compose ps"
```

Проверь nginx:

```bash
ansible app1 -i ansible/inventory -b -m shell -a "systemctl status nginx --no-pager"
```

---

## 6. Если кажется, что зависло

Обычно это не ошибка.
Первый запуск может быть долгим, потому что Docker скачивает большие образы.

Можно заранее скачать образы:

```bash
ansible app1 -i ansible/inventory -b -m shell -a "cd /opt/demo-app && docker-compose pull"
```

Потом снова запусти playbook.

---

## 7. Где что лежит

- `ansible/site.yml` — главная логика
- `ansible/inventory` — твой сервер и параметры
- `ansible/group_vars/all.yml` — значения по умолчанию
- `ansible/templates/nginx-app.conf.j2` — шаблон nginx
- `ansible/templates/app.service.j2` — шаблон systemd

