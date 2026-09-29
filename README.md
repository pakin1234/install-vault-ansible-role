# Ansible role `vault`

Ansible-роль предназначена для установки и базовой конфигурации HashiCorp Vault на Ubuntu-сервере. В качестве внутреннего хранилища используется Integrated Storage на базе Raft.

Роль рассчитана на первоначальное развертывание **одиночного Vault-узла** на одной виртуальной машине кафедры.

> Важно: один узел Raft обеспечивает персистентное хранение данных, но не обеспечивает отказоустойчивость. При недоступности виртуальной машины Vault также будет недоступен.

## Назначение роли

Роль выполняет следующие действия:

- устанавливает последнюю стабильную версию Vault из официального APT-репозитория HashiCorp;
- проверяет, что целевая система — Ubuntu;
- создает системного пользователя и группу `vault`;
- создает каталоги конфигурации и данных;
- формирует конфигурационный файл Vault;
- настраивает Integrated Storage `raft`;
- настраивает локальный TCP listener;
- отключает `mlock` в соответствии с конфигурацией Integrated Storage;
- запускает Vault как systemd-сервис;
- настраивает базовые правила UFW;
- проверяет состояние systemd-сервиса и health endpoint.

Официальный APT-репозиторий HashiCorp используется для установки Vault на Ubuntu. [developer.hashicorp](https://developer.hashicorp.com/vault/install)

## Требования

Перед запуском роли необходимо обеспечить:

- Ubuntu 24.04 LTS или другую поддерживаемую Ubuntu;
- Python на целевой машине;
- Ansible с правами `become`;
- доступ VM к `apt.releases.hashicorp.com`;
- доступ к официальному GPG-ключу HashiCorp;
- persistent-диск для `/var/lib/vault`;
- свободные порты `8200` и `8201`;
- DNS-имя или IP-адрес для доступа к Vault;
- установленный Nginx, если он используется как reverse proxy.


## Использование

Запуск:

```bash
ansible-playbook \
  -i inventory/hosts.yml \
  playbooks/vault.yml
```

## Таблица основных переменных

| Переменная | Значение по умолчанию | Назначение |
|---|---:|---|
| `vault_package_name` | `vault` | Имя APT-пакета |
| `vault_package_state` | `present` | Состояние пакета |
| `vault_user` | `vault` | Системный пользователь |
| `vault_group` | `vault` | Системная группа |
| `vault_config_dir` | `/etc/vault.d` | Каталог конфигурации |
| `vault_config_file` | `/etc/vault.d/vault.hcl` | Файл конфигурации |
| `vault_data_dir` | `/var/lib/vault` | Каталог Raft data |
| `vault_api_addr` | `http://127.0.0.1:8200` | Рекламируемый API-адрес |
| `vault_cluster_addr` | `http://127.0.0.1:8201` | Адрес cluster traffic |
| `vault_node_id` | `inventory_hostname` | Уникальный ID Raft-узла |
| `vault_listener_address` | `127.0.0.1:8200` | Адрес API listener |
| `vault_cluster_listener_address` | `127.0.0.1:8201` | Адрес cluster listener |
| `vault_tls_disable` | `true` | Отключение TLS в Vault |
| `vault_disable_mlock` | `true` | Отключение системного `mlock` |
| `vault_ui_enabled` | `true` | Включение Vault UI |
| `vault_key_shares` | `3` | Количество unseal shares при ручном init |
| `vault_key_threshold` | `2` | Число shares для unseal |
| `vault_raft_snapshot_threshold` | `8192` | Число Raft entries до внутреннего snapshot |
| `vault_raft_snapshot_interval` | `120s` | Интервал проверки необходимости snapshot |

## Установка Vault

Роль:

1. устанавливает prerequisites;
2. получает Ubuntu codename;
3. получает архитектуру через `dpkg --print-architecture`;
4. добавляет официальный HashiCorp repository;
5. обновляет APT cache;
6. проверяет наличие пакета `vault`;
7. устанавливает последнюю стабильную версию.


## `disable_mlock`

В роли используется:

```hcl
disable_mlock = true
```

Это отключает вызов `mlock`, который запрещает операционной системе выгружать память процесса в swap.

Для Integrated Storage Raft это практичный вариант, поскольку Raft использует memory-mapped files. При включенном `mlock` большой объем данных может быть удержан в RAM и привести к нехватке памяти. HashiCorp рекомендует учитывать этот параметр при использовании Integrated Storage и отдельно защищать систему от нешифрованного swap. [developer.hashicorp](https://developer.hashicorp.com/vault/docs/concepts/tune-server-performance)


## Первичная инициализация

Роль не выполняет `operator init`. После деплоя и запуска сервиса оператор выполняет:

```bash
export VAULT_ADDR="http://{{ vault_addr }}:8200"

vault operator init \
  -key-shares=3 \
  -key-threshold=2
```

Инициализация подготавливает storage backend Vault к работе и создает unseal shares и initial root token. [developer.hashicorp](https://developer.hashicorp.com/vault/docs/commands/operator/init)


Полученные значения необходимо передать держателям через защищенный канал

## Первичный unseal

Для схемы `3-of-2` два держателя по очереди выполняют:

```bash
vault operator unseal
vault operator unseal
```

После unseal:

```bash
vault status
```

Ожидается:

```text
Initialized    true
Sealed         false
Storage Type   raft
HA Enabled     true
```

