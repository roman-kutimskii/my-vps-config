# Minimal VPS Setup

Минимальный Ansible-проект для первичной настройки одного Ubuntu 24.04 VPS под удаленную разработку.

## Архитектура

- `bootstrap.yml` подготавливает доступ: создает пользователя `dev`, настраивает SSH-ключ и `sudo`
- `site.yml` настраивает базовый хост, Docker, Caddy и OpenCode
- `OpenCode` запускается как `systemd`-сервис и слушает только `127.0.0.1:4096`
- `Authelia` + `Redis` запускаются через `docker compose`
- `Authelia` использует file-based users backend, `SQLite` для storage, `Redis` для session storage
- `Caddy` завершает TLS, публикует `auth.<root_domain>` и защищает `opencode.<root_domain>` через `forward_auth`

## Доменная модель

Публично заполняется только `root_domain`.

- `auth.<root_domain>` -> `Authelia`
- `opencode.<root_domain>` -> `Caddy` -> `OpenCode`

Оба публичных адреса вычисляются внутри playbook и шаблонов из `root_domain`.

## Допущения

- целевой сервер: Ubuntu 24.04 LTS
- первый вход возможен как `root` по SSH-паролю
- DNS-записи для `auth.<root_domain>` и `opencode.<root_domain>` уже указывают на VPS перед запуском `site.yml`

## Что заполнить

`inventory/hosts.yml`

- `ansible_host`
- при необходимости `ansible_port`

`group_vars/all/main.yml`

- `root_domain`
- `dev_authorized_key`
- `server_timezone`
- `swapfile_size_mb`
- `authelia_username`
- при желании `authelia_display_name`
- `authelia_dir`

`group_vars/all/vault.yml`

- `vault_root_ssh_password`
- `vault_dev_password`
- `vault_dev_password_hash`
- `vault_authelia_password_hash`
- `vault_authelia_session_secret`
- `vault_authelia_storage_encryption_key`
- `vault_authelia_redis_password`

## Secrets

`vault_dev_password_hash` должен содержать готовый SHA-512 crypt hash пароля Linux-пользователя `dev`, например вида `$6$...`.

Сгенерировать его можно локально так:

```bash
openssl passwd -6
```

`vault_authelia_password_hash` должен содержать hash пароля единственного пользователя `Authelia`.

Сгенерировать его можно так:

```bash
docker run --rm authelia/authelia:latest authelia crypto hash generate argon2 --password 'replace-me'
```

В `vault.yml` нужно сохранить строку после `Digest:`.

Остальные секреты для `Authelia` можно сгенерировать локально так:

```bash
openssl rand -hex 32
```

Эта команда подходит для:

- `vault_authelia_session_secret`
- `vault_authelia_storage_encryption_key`
- `vault_authelia_redis_password`

После заполнения secrets зашифровать vault:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

Установить коллекции:

```bash
ansible-galaxy collection install -r requirements.yml
```

## Запуск

1. Bootstrap:

```bash
ansible-playbook bootstrap.yml --ask-vault-pass
```

2. Основная настройка:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Порядок всегда один: сначала `bootstrap.yml`, потом `site.yml`.

## Auth Flow

1. Пользователь открывает `https://opencode.<root_domain>`
2. `Caddy` отправляет auth-check в `Authelia` через `/api/authz/forward-auth`
3. Если сессии нет, браузер получает redirect на `https://auth.<root_domain>`
4. Пользователь логинится в `Authelia` по `username` и паролю
5. `Authelia` выставляет session cookie для `root_domain`
6. Следующий запрос к `opencode.<root_domain>` проходит `forward_auth`, после чего `Caddy` проксирует его в локальный `OpenCode`

Встроенный пароль `OpenCode` больше не используется.

## Результат

После завершения:

- вход по SSH идет как `dev` по ключу
- `dev` может использовать `sudo` по паролю
- на сервере работают Docker, Caddy, OpenCode, Authelia и Redis
- каталог `/home/dev/projects` создан для проектов
- `Authelia` доступна по `https://auth.<root_domain>`
- `OpenCode` доступен по `https://opencode.<root_domain>` только после логина в `Authelia`
