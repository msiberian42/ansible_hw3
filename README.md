# Ansible Playbook

Ansible playbook для установки и базовой настройки сервисов:

* **ClickHouse** — устанавливается из RPM-пакетов и запускается как системный сервис.
* **Vector** — устанавливается из официального архива, создаётся конфигурация и systemd-unit.
* **LightHouse** — устанавливается как статический набор файлов, а Nginx настраивается для их публикации по HTTP.

Playbook inventory с группами `clickhouse`, `vector` и `lighthouse`.

## Структура

```text
.
├── site.yml
├── inventory/
│   └── prod.yml
├── group_vars/
│   ├── clickhouse/
│   │   └── vars.yml
│   ├── lighthouse/
│   │   └── vars.yml
│   └── vector/
│       └── vars.yml
└── templates/
    ├── vector.toml.j2
    ├── vector.service.j2
    └── lighthouse.conf.j2
```

## Функции playbook

### ClickHouse

1. Скачивает необходимые RPM-пакеты ClickHouse.
2. При необходимости использует резервный вариант загрузки `clickhouse-common-static`.
3. Устанавливает ClickHouse.
4. Запускает и перезапускает сервис `clickhouse-server` при изменении конфигурации.
5. Создаёт базу данных `logs`.

### Vector

1. Скачивает архив Vector.
2. Создаёт директорию для установки.
3. Распаковывает Vector.
4. Создаёт ссылку на бинарный файл `/usr/local/bin/vector`.
5. Создаёт директории для конфигурации и данных.
6. Разворачивает конфигурацию Vector из шаблона.
7. Создаёт systemd-unit для Vector.
8. Запускает и включает сервис Vector.

Текущая конфигурация Vector читает события из `journald` и выводит их в `console`.

### LightHouse

1. Устанавливает Nginx.
2. Скачивает архив LightHouse.
3. Распаковывает файлы LightHouse.
4. Размещает статические файлы в `/var/www/lighthouse`.
5. Настраивает Nginx для публикации LightHouse.
6. Настраивает SELinux-контекст для файлов LightHouse.
7. Запускает и включает Nginx.

После установки веб-интерфейс LightHouse доступен по HTTP на IP-адресе хоста.

## Параметры

Основные параметры playbook задаются через переменные Ansible.

### ClickHouse

| Переменная            | Назначение                     | Пример                                                               |
| --------------------- | ------------------------------ | -------------------------------------------------------------------- |
| `clickhouse_version`  | Версия ClickHouse              | `22.3.3.44`                                                          |
| `clickhouse_packages` | Список устанавливаемых пакетов | `clickhouse-common-static`, `clickhouse-client`, `clickhouse-server` |

### Vector

| Переменная           | Назначение              | Пример        |
| -------------------- | ----------------------- | ------------- |
| `vector_version`     | Версия Vector           | `0.34.0`      |
| `vector_arch`        | Архитектура Vector      | `x86_64`      |
| `vector_install_dir` | Директория установки    | `/opt/vector` |
| `vector_config_dir`  | Директория конфигурации | `/etc/vector` |

### LightHouse

| Переменная               | Назначение                       | Пример                                                                 |
| ------------------------ | -------------------------------- | ---------------------------------------------------------------------- |
| `lighthouse_version`     | Версия/ветка LightHouse          | `master`                                                               |
| `lighthouse_repo`        | URL архива LightHouse            | `https://github.com/VKCOM/lighthouse/archive/refs/heads/master.tar.gz` |
| `lighthouse_archive`     | Путь к скачанному архиву         | `/tmp/lighthouse.tar.gz`                                               |
| `lighthouse_install_dir` | Директория размещения LightHouse | `/var/www/lighthouse`                                                  |

Также в `inventory/prod.yml` задаются параметры подключения к серверам:

* `ansible_host` — IP-адрес хоста;
* `ansible_user` — пользователь для подключения;
* `ansible_python_interpreter` — интерпретатор Python на удалённом хосте.

## Теги

В текущей версии playbook отдельные теги для задач не используются.

## Запуск

Полный запуск playbook:

```bash
ansible-playbook -i inventory/prod.yml site.yml
```

## Проверка после установки

### ClickHouse

```bash
ansible -i inventory/prod.yml clickhouse \
  -m ansible.builtin.command \
  -a "systemctl is-active clickhouse-server"
```

Проверка базы данных:

```bash
ansible -i inventory/prod.yml clickhouse \
  -m ansible.builtin.command \
  -a "clickhouse-client -q 'SHOW DATABASES'"
```

### Vector

```bash
ansible -i inventory/prod.yml vector \
  -m ansible.builtin.command \
  -a "systemctl is-active vector"
```

Проверка конфигурации:

```bash
ansible -i inventory/prod.yml vector \
  -m ansible.builtin.command \
  -a "sudo /usr/local/bin/vector validate /etc/vector/vector.toml"
```

### LightHouse / Nginx

```bash
ansible -i inventory/prod.yml lighthouse \
  -m ansible.builtin.command \
  -a "systemctl is-active nginx"
```

Проверка конфигурации Nginx:

```bash
ansible -i inventory/prod.yml lighthouse \
  -m ansible.builtin.command \
  -a "nginx -t"
```

## Результат

После успешного выполнения playbook:

* ClickHouse установлен и запущен;
* Vector установлен и запущен;
* LightHouse размещён на веб-сервере;
* Nginx установлен, настроен и запущен;
* все три компонента развёрнуты на отдельных хостах.
