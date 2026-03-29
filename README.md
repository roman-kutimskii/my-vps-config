# Minimal VPS Setup

Минимальный Ansible-проект для первичной настройки одного Ubuntu 24.04 VPS под удаленную разработку.

## Что делает

- готовит доступ по SSH как `dev`
- включает `sudo` для `dev`
- ставит базовые пакеты, Docker, Caddy и OpenCode
- публикует OpenCode по HTTPS через Caddy

## Допущения

- целевой сервер: Ubuntu 24.04 LTS
- первый вход возможен как `root` по SSH-паролю
- DNS для домена из `opencode_domain` уже указывает на VPS перед запуском `site.yml`

## Что заполнить

`inventory/hosts.yml`

- `ansible_host`
- при необходимости `ansible_port`

`group_vars/all.yml`

- `opencode_domain`
- `dev_authorized_key`
- при необходимости `server_timezone` и `swapfile_size_mb`

`group_vars/all/vault.yml`

- `vault_root_ssh_password`
- `vault_dev_password`
- `vault_opencode_server_password`

После заполнения зашифровать secrets:

```bash
ansible-vault encrypt group_vars/all/vault.yml
```

## Запуск

Bootstrap:

```bash
ansible-playbook bootstrap.yml --ask-vault-pass
```

Основная настройка:

```bash
ansible-playbook site.yml --ask-vault-pass
```

Порядок всегда один: сначала `bootstrap.yml`, потом `site.yml`.

## Результат

После завершения:

- вход по SSH идет как `dev` по ключу
- `dev` может использовать `sudo` по паролю
- на сервере работают Docker, OpenCode и Caddy
- каталог `/home/dev/projects` создан для проектов
- OpenCode доступен по `https://<opencode_domain>`
