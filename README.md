# Kubernetes кластер с Cluster API на Proxmox

Данный репозиторий содержит манифесты Cluster API для развертывания Kubernetes кластера на Proxmox Virtual Environment (PVE).

## Обзор

Конфигурация создает Kubernetes кластер с:
- 1 узел Control Plane (4 ядра, 4ГБ ОЗУ)
- 1 рабочий узел (2 ядра, 4ГБ ОЗУ)
- Версия Kubernetes: v1.32.0
- Pod CIDR: 10.244.0.0/16
- Виртуальный IP для HA: 192.168.1.30

Если нужен высокодоступный кластер, увеличте количество нод control-plane до трех

## Предварительные требования

### Требования к инфраструктуре
- Кластер Proxmox VE с узлом `pimox2`
- Шаблон ВМ с ID `5002` (рекомендуется Ubuntu/Debian cloud образ)
- Настроенный сетевой мост `vmbr0`
- Доступный диапазон IP: 192.168.1.31-192.168.1.39
- Шлюз: 192.168.1.1
- DNS серверы: 8.8.8.8, 8.8.4.4

### Управляющий кластер
- Kubernetes кластер с установленным Cluster API
- Установленный Cluster API Proxmox Provider (CAPMOX)
- kubectl настроенный для доступа к управляющему кластеру

### Подготовка шаблона ВМ
Ваш Proxmox шаблон (ID 5002) должен содержать:
- Включенную поддержку Cloud-init
- Установленный qemu-guest-agent
- Container runtime (рекомендуется containerd)


## Установка

### 1. Установка компонентов управления Cluster API

```bash
# Установка clusterctl
curl -L https://github.com/kubernetes-sigs/cluster-api/releases/latest/download/clusterctl-linux-amd64 -o clusterctl
chmod +x clusterctl
sudo mv clusterctl /usr/local/bin/clusterctl

# Инициализация Cluster API
clusterctl init --infrastructure proxmox --ipam in-cluster
```

### 2. Настройка учетных данных Proxmox

Создайте токен для доаступа на Proxmox. Зайдите на ноду Proxmox и выполните команды:

```bash
pveum user add capmox@pve
pveum aclmod / -user capmox@pve -role PVEVMAdmin
pveum user token add capmox@pve capi -privsep 0
```

### 3. Развертывание кластера

```bash
kubectl apply -f cluster.yaml
```

### 4. Мониторинг создания кластера

```bash
# Отслеживание статуса кластера
kubectl get clusters
kubectl get machines
```

### 5. Получение kubeconfig кластера

```bash
# Получение kubeconfig для рабочего кластера
clusterctl get kubeconfig myCluster > myCluster-kubeconfig.yaml

# Использование kubeconfig
export KUBECONFIG=myCluster-kubeconfig.yaml
kubectl get nodes
```

## Детали конфигурации

### Сетевая конфигурация
- **Endpoint Control Plane**: 192.168.1.30:6443
- **Сеть подов**: 10.244.0.0/16 (требует установки CNI)
- **Сеть сервисов**: Стандартный Kubernetes service CIDR
- **IP узлов**: Автоматически назначаются из диапазона 192.168.1.31-192.168.1.39

### Высокая доступность
- **kube-vip**: Обеспечивает виртуальный IP для HA control plane
- **Виртуальный IP**: 192.168.1.30
- **Leader Election**: Включен с продолжительностью lease 15 секунд

### Безопасность
- SSH доступ настроен с предоставленным публичным ключом
- Интеграция Provider ID с метаданными Proxmox
- Пользователь root с правами sudo

## Действия после установки

### 1. Установка Container Network Interface (CNI)
```bash
# Пример с Flannel
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Или с Cilium
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --version 1.14.5
```

### 2. Проверка состояния кластера
```bash
kubectl get nodes
kubectl get pods -A
kubectl cluster-info
```

### 3. Масштабирование рабочих узлов (опционально)
```bash
kubectl patch machinedeployment myCluster-workers \
  --type='merge' \
  -p='{"spec":{"replicas":3}}'
```

## Кастомизация

### Масштабирование Control Plane
Для обеспечения высокой доступности control plane, обновите `KubeadmControlPlane`:

```yaml
spec:
  replicas: 3  # Изменить с 1 на 3
```

### Изменение ресурсов ВМ
Обновите спецификации `ProxmoxMachineTemplate`:

```yaml
spec:
  template:
    spec:
      memoryMiB: 8192    # Увеличить память
      numCores: 4        # Увеличить количество ядер
      disks:
        bootVolume:
          sizeGb: 80     # Увеличить размер диска
```

### Другая версия Kubernetes
Обновите версию в `KubeadmControlPlane` и `MachineDeployment`:

```yaml
spec:
  version: v1.31.0  # Изменить версию
```

## Архитектура

```
┌─────────────────┐    ┌─────────────────┐
│  Управляющий    │    │   Рабочий       │
│   кластер       │───▶│   кластер       │
│  (Cluster API)  │    │  (myCluster)    │
└─────────────────┘    └─────────────────┘
                               │
                     ┌─────────┴─────────┐
                     │                   │
              ┌──────▼───-─┐   ┌─────────▼──┐
              │Control     │   │  Рабочие   │
              │Plane       │   │   узлы     │
              │192.168.1.30│   │192.168.1.x │
              └───────────-┘   └────────────┘
                    │                 │
              ┌─────▼─────────────────▼─────┐
              │   Кластер Proxmox VE        │
              │         (pimox2)            │
              └─────────────────────────────┘
```

## Структура файлов

```
.
├── cluster-manifests.yaml    # Основные манифесты кластера
├── README.md                 # Данный файл
└── docs/                     # Дополнительная документация
    ├── networking.md         # Настройка сети
    ├── security.md           # Рекомендации по безопасности
    └── troubleshooting.md    # Подробное руководство по устранению неисправностей
```

## Мониторинг и обслуживание

### Регулярные проверки
```bash
# Проверка состояния узлов
kubectl get nodes -o wide

# Проверка состояния подов системы
kubectl get pods -n kube-system

# Проверка ресурсов кластера
kubectl top nodes
kubectl top pods -A
```

### Обновление кластера
```bash
# Обновление версии Kubernetes
kubectl patch kubeadmcontrolplane myCluster-control-plane \
  --type='merge' \
  -p='{"spec":{"version":"v1.32.1"}}'

# Обновление рабочих узлов
kubectl patch machinedeployment myCluster-workers \
  --type='merge' \
  -p='{"spec":{"template":{"spec":{"version":"v1.32.1"}}}}'
```

### Резервное копирование
```bash
# Создание резервной копии etcd (выполнить на control plane узле)
sudo etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

## Безопасность

### Рекомендации
- Регулярно обновляйте SSH ключи
- Используйте сетевые политики для изоляции подов
- Настройте RBAC для ограничения доступа
- Включите аудит логирование
- Используйте Pod Security Standards

### Пример сетевой политики
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

## Лицензия

Данная конфигурация предоставляется под лицензией Apache 2.0. См. файл LICENSE для подробностей.
