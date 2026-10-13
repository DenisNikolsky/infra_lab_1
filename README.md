# Лабораторная работа №1 — Ansible-роль для установки OpenSearch 3.8 (single-node)

## 1. Цель работы

Разработать Ansible-роль для установки и настройки **OpenSearch 3.8** и **OpenSearch Dashboards 3.8** в режиме **single-node** на удалённой Linux-машине без использования Docker и сторонних ролей.

---

## 2. Используемое окружение

| Компонент | Версия / параметры |
|---|---|
| Управляющая машина (control node) | Ubuntu 22.04 |
| Ansible | 2.15+ |
| Целевая машина (managed node) | Ubuntu 22.04 LTS |
| Python на target-машине | 3.10 (уже установлен) |
| OpenSearch | 3.8.0 |
| OpenSearch Dashboards | 3.8.0 |
| Java | встроенный JDK из состава архива OpenSearch |
| Режим работы | single-node |
| Тип установки | tar.gz, напрямую на хост (без Docker) |

---

## 3. Структура роли

Все стандартные директории роли задействованы по назначению:

- `defaults/` — параметры роли, которые можно переопределить без правки задач;
- `tasks/` — логика установки, настройки, запуска и проверки;
- `handlers/` — перезапуск сервисов только при изменении конфигурации;
- `templates/` — Jinja2-шаблоны конфигов и systemd-юнитов;
- `meta/` — метаданные роли.

---

## 4. Реализация роли

### 4.1. install.yml

- Создание системного пользователя и группы `opensearch`.
- Создание каталогов `/opt`, `/etc/opensearch`, `/etc/opensearch-dashboards`, `/var/lib/...`, `/var/log/...`.
- Загрузка tarball-архивов OpenSearch 3.8 и OpenSearch Dashboards 3.8 с официального артефакт-сервера.
- Распаковка в `/opt` и создание симлинков `/opt/opensearch`, `/opt/opensearch-dashboards`.
- Копирование дефолтных конфигов в `/etc/opensearch` и `/etc/opensearch-dashboards`.

### 4.2. configure.yml

- Настройка `vm.max_map_count = 262144` через `ansible.posix.sysctl`.
- Генерация `opensearch.yml`, `jvm.options`, `opensearch_dashboards.yml` из Jinja2-шаблонов.
- При изменении любого конфига срабатывает handler перезапуска соответствующего сервиса.

### 4.3. service.yml

- Создание systemd-юнитов `opensearch.service` и `opensearch-dashboards.service` из Jinja2-шаблонов.
- `enabled: true` — автоматический запуск после перезагрузки машины.
- `state: started` — запуск сервисов сразу после установки.

### 4.4. verify.yml

- Проверка готовности OpenSearch через API `GET /_cluster/health`.
- Ограниченное количество повторных попыток (`retries` + `delay`).
- Ожидаемый статус кластера — `green` или `yellow`.

---

## 5. Параметризация роли

Пользователь может менять параметры в `defaults/main.yml` или переопределять их через `vars` / `--extra-vars` **без редактирования задач и шаблонов**.

| Параметр | Значение по умолчанию | Назначение |
|---|---|---|
| `opensearch_version` | `3.8.0` | Версия OpenSearch |
| `opensearch_dashboards_version` | `3.8.0` | Версия OpenSearch Dashboards |
| `opensearch_arch` | `x64` | Архитектура (`x64` / `arm64`) |
| `opensearch_cluster_name` | `opensearch-single` | Имя кластера |
| `opensearch_node_name` | `{{ inventory_hostname }}` | Имя ноды |
| `opensearch_bind_host` | `0.0.0.0` | Адрес прослушивания HTTP |
| `opensearch_http_port` | `9200` | HTTP-порт OpenSearch |
| `opensearch_transport_port` | `9300` | Transport-порт |
| `opensearch_heap_size` | `1g` | Размер heap JVM (`-Xms`/`-Xmx`) |
| `opensearch_dashboards_host` | `0.0.0.0` | Адрес Dashboards |
| `opensearch_dashboards_port` | `5601` | Порт Dashboards |
| `opensearch_security_disabled` | `true` | Отключение security-плагина (для лабораторной проверки) |
| `opensearch_wait_retries` | `30` | Количество повторных попыток проверки API |
| `opensearch_wait_delay` | `5` | Пауза между попытками (сек) |

### Обоснование выбора параметров

- **Версия вынесена в параметр** — требование лабораторной: роль должна устанавливать «выбранную версию».
- **`cluster_name` и `node_name`** — нужны для идентификации кластера и ноды, удобно менять при развёртывании в разных окружениях.
- **Порты HTTP/Transport и Dashboards** — часто возникает конфликт с уже занятыми портами, поэтому вынесены наружу.
- **`opensearch_heap_size`** — критичный параметр JVM, зависит от объёма RAM на хосте; по умолчанию `1g` безопасен почти везде.
- **`opensearch_security_disabled`** — упрощает проверку в учебной среде; в production выключать нельзя.
- **`opensearch_wait_retries` и `opensearch_wait_delay`** — дают гибкость на слабых машинах, где OpenSearch стартует дольше.
- **`opensearch_arch`** — позволяет использовать роль и на ARM-серверах.

---

## 6. Использование handlers

Handlers применяются для перезапуска сервисов **только при фактических изменениях**:

- `restart opensearch` — срабатывает при изменении `opensearch.yml`, `jvm.options` или systemd-юнита.
- `restart opensearch-dashboards` — срабатывает при изменении `opensearch_dashboards.yml` или его systemd-юнита.
- `reload systemd` — срабатывает при изменении systemd-юнитов.

Если ни один конфиг не изменился, перезапуска не будет.

---

## 7. Автозапуск после перезагрузки

Оба сервиса включаются с `enabled: true` через модуль `ansible.builtin.systemd`. После `reboot` машины OpenSearch и Dashboards поднимутся автоматически без участия Ansible.

---

## 8. Команды запуска

### 8.1. Установка зависимостей

```bash
ansible-galaxy collection install ansible.posix