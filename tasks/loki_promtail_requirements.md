# Требования к внедрению логирования (Loki + Promtail)

## Общая задача
Развернуть стек для сбора и просмотра логов: Grafana Loki (хранилище) и Promtail (агент сбора). Интегрировать с существующей Grafana.

## Детальные требования по компонентам

### 1. Переменные (`group_vars/all/vars.yml`)
1.  **Mount Folders**: Добавить путь для persistence Loki (например, `loki: /etc/loki_data`).
2.  **Services List**: Добавить `loki` и `promtail` в список `services`.
3.  **Non-build Services**: Добавить `loki` и `promtail` в `non_build_services`, так как будут использоваться официальные образы.

### 2. Сервис Loki (`roles/deploy/services/loki`)
1.  **Директория**: Создать структуру папок.
2.  **Main Task (`main.yml`)**:
    *   Deploy через `community.docker.docker_swarm_service`.
    *   Image: `grafana/loki:latest`.
    *   Network: `{{ network_name }}` (app_network).
    *   Ports: 3100 (можно не публиковать наружу, если доступ только из Grafana внутри сети).
    *   Config: Подложить `loki-config.yaml` (создать шаблон) через Docker Config или Secret.
    *   Mounts: Bind mount для данных (`mount_folders.loki` -> `/loki` или куда пишет Loki).
    *   User: Учесть UID/GID (Loki часто запускается от 10001 или root, проверить).

### 3. Сервис Promtail (`roles/deploy/services/promtail`)
1.  **Директория**: Создать структуру папок.
2.  **Main Task (`main.yml`)**:
    *   Deploy через `community.docker.docker_swarm_service`.
    *   **Mode**: `global` (запуск на каждой ноде Swarm).
    *   Image: `grafana/promtail:latest`.
    *   Network: `{{ network_name }}`.
    *   Config: Подложить `promtail-config.yaml` (создать шаблон). Конфиг должен быть настроен на `docker_sd_configs` или чтение `/var/lib/docker/containers`.
    *   Mounts:
        *   `/var/lib/docker/containers` (RO) -> для чтения логов.
        *   `/var/run/docker.sock` (если требуется для service discovery).
3.  **Конфигурация (promtail-config.yaml)**:
    *   Настроить отправку логов в `http://loki:3100/loki/api/v1/push`.
    *   Настроить relabeling для получения имен сервисов Swarm (через docker socket или метаданные файлов).

### 4. Grafana
1.  **Datasource**: Проверить возможность автоматического добавления Loki как Datasource. Если текущая роль Grafana не поддерживает provisioning, добавить инструкцию по ручной настройке или доработать роль (монтирование `datasources.yaml` в `/etc/grafana/provisioning/datasources`).

## План действий
1.  Создать конфигурационные файлы (шаблоны) для Loki и Promtail.
2.  Обновить `vars.yml`.
3.  Создать задачи развертывания (`main.yml` для каждого сервиса).
4.  Протестировать деплой на staging/local env.
5.  Проверить поступление логов в Grafana.
