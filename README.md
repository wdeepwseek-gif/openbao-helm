# OpenBao Helm Chart

![Version: 0.7.0](https://img.shields.io/badge/Version-0.7.0-informational?style=flat-square) ![AppVersion: v2.1.0](https://img.shields.io/badge/AppVersion-v2.1.0-informational?style=flat-square)

Официальный Helm Chart для развертывания OpenBao в Kubernetes.

## Описание

OpenBao - это форк HashiCorp Vault, открытая система для управления секретами и защиты данных. Этот Helm Chart позволяет легко развернуть OpenBao в Kubernetes-кластере с поддержкой различных режимов работы.

## Возможности

- **Режимы развертывания**:
  - Standalone (одиночный экземпляр)
  - High Availability (HA) с Raft storage
  - Development режим

- **Компоненты**:
  - OpenBao Server
  - Agent Injector (автоматическое внедрение агента в поды)
  - CSI Provider (для монтирования секретов через CSI)
  - Web UI

- **Интеграции**:
  - Kubernetes Auth Method
  - Prometheus метрики
  - Service Registration

## Требования

- Kubernetes `>= 1.22.0`
- Helm `>= 3.0.0`
- Для HA режима с Raft требуется PersistentVolume

## Быстрый старт

### Установка

```bash
# Установка в режиме standalone
helm install openbao . -n openbao --create-namespace

# Или с кастомным values файлом
helm install openbao . -f values.yaml -n openbao --create-namespace
```

### Обновление

```bash
helm upgrade openbao . -n openbao
```

### Удаление

```bash
helm uninstall openbao -n openbao
```

## Конфигурация

### Основные параметры

Основные параметры конфигурации находятся в файле `values.yaml`. Вот некоторые ключевые настройки:

#### Глобальные настройки

```yaml
global:
  enabled: true
  namespace: ""
  tlsDisable: true
  openshift: false
```

#### Режимы работы сервера

**Standalone режим** (одиночный экземпляр):
```yaml
server:
  standalone:
    enabled: true
```

**HA режим с Raft** (высокая доступность):
```yaml
server:
  ha:
    enabled: true
    raft:
      enabled: true
    replicas: 3
```

**Dev режим** (только для разработки):
```yaml
server:
  dev:
    enabled: true
    devRootToken: "root"
```

#### Хранилище данных

```yaml
server:
  dataStorage:
    enabled: true
    size: 10Gi
    storageClass: null  # Использует default StorageClass
    mountPath: "/openbao/data"
```

#### Agent Injector

```yaml
injector:
  enabled: true
  replicas: 1
  authPath: "auth/kubernetes"
```

#### CSI Provider

```yaml
csi:
  enabled: false  # Требует установки secrets-store-csi-driver
```

### Примеры конфигурации

#### Standalone с UI

```yaml
server:
  standalone:
    enabled: true
  
ui:
  enabled: true
  serviceType: LoadBalancer
```

#### HA с Raft

```yaml
server:
  ha:
    enabled: true
    raft:
      enabled: true
    replicas: 3
    
  dataStorage:
    enabled: true
    size: 10Gi
```

## Инициализация и разблокировка

После установки OpenBao нужно инициализировать и разблокировать:

```bash
# Войти в под
kubectl exec -it openbao-0 -n openbao -- /bin/sh

# Инициализация
openbao operator init -key-shares=1 -key-threshold=1

# Сохранить ключи разблокировки и токен

# Разблокировать (повторить для каждого пода в HA режиме)
openbao operator unseal <unseal-key>
```

## Доступ к Web UI

Если UI включен, вы можете получить доступ:

```bash
# Получить URL сервиса
kubectl get svc openbao-ui -n openbao

# Или через port-forward
kubectl port-forward svc/openbao-ui 8200:8200 -n openbao

# Внимание: порт 8200 может быть занят, используйте другой порт при необходимости:
# kubectl port-forward svc/openbao-ui 8201:8200 -n openbao
```

Затем откройте в браузере по адресу, который будет отображен в выводе команды `port-forward`.

## Безопасность

⚠️ **Важные рекомендации по безопасности:**

1. **Никогда не используйте dev режим в продакшене**
2. **Включите TLS** (`global.tlsDisable: false`) и настройте сертификаты
3. **Используйте внешнее хранилище ключей** (auto-unseal) для HA режима
4. **Настройте сетевые политики** для ограничения доступа
5. **Регулярно обновляйте** образы OpenBao
6. **Используйте Kubernetes Secrets** для хранения чувствительных данных конфигурации
7. **Не храните ключи разблокировки и токены в открытом виде**

## Мониторинг

### Prometheus метрики

Для включения метрик Prometheus:

```yaml
serverTelemetry:
  serviceMonitor:
    enabled: true
    interval: 30s

server:
  standalone:
    config: |
      listener "tcp" {
        telemetry {
          unauthenticated_metrics_access = "true"
        }
      }
      telemetry {
        prometheus_retention_time = "30s"
        disable_hostname = true
      }
```

## Troubleshooting

### Проверка статуса подов

```bash
kubectl get pods -n openbao
```

### Просмотр логов

```bash
# Логи сервера
kubectl logs openbao-0 -n openbao

# Логи injector
kubectl logs -l app.kubernetes.io/name=openbao-agent-injector -n openbao
```

### Проверка статуса OpenBao

```bash
kubectl exec -it openbao-0 -n openbao -- openbao status
```

### Частые проблемы

**Проблема**: Pod не запускается или в статусе CrashLoopBackOff

**Решение**: Проверьте логи пода и убедитесь, что все необходимые конфигурации заданы правильно. Для HA режима убедитесь, что OpenBao инициализирован и разблокирован.

**Проблема**: Не удается получить доступ к UI

**Решение**: Проверьте настройки Service и Ingress. Убедитесь, что UI включен в values.yaml.

## Документация

Для получения дополнительной информации о настройке и использовании OpenBao обратитесь к официальной документации проекта.

## Лицензия

Этот проект основан на официальном Helm Chart для OpenBao.

## Поддержка

Для получения помощи и поддержки обратитесь к официальным источникам проекта OpenBao.
