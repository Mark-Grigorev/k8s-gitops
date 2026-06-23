# k8s-gitops

Пет-проект для изучения IaC (Ansible) и GitOps-подхода.  
Цель — поднять production-like окружение на одном удалённом сервере с несколькими реальными приложениями.

---

## Архитектура
<details>

```
                          Internet
                             │
                          80/443
                             │
                      ┌──────▼──────┐
                      │Nginx Ingress│  hostNetwork, TLS termination
                      └──────┬──────┘
                             │
       ┌─────────────────────┼──────────────────────┬──────────────────────┐
       │                     │                      │                      │
┌──────▼──────┐       ┌──────▼──────┐       ┌──────▼──────┐       ┌──────▼──────┐
│   fingo     │       │   gotanks   │       │  alexstav   │       │mark-alldevops│
│ Go + nginx  │       │  Go (game)  │       │   nginx     │       │    nginx     │
│ fingo.ink   │       │tanks.fingo  │       │ alexstav.dev│       │mark.alldevops│
│             │       │    .ink     │       │             │       │    .ru       │
└──────┬──────┘       └──────┬──────┘       └─────────────┘       └─────────────┘
       │                     │
       └──────────┬──────────┘
                  │
           ┌──────▼──────┐         ┌──────────────┐
           │ PostgreSQL  │         │    Vault     │  секреты (DB, API keys, JWT)
           │  (на хосте) │         │  (injector)  │
           └─────────────┘         └──────────────┘

  WireGuard VPN (внутренний доступ)
  ┌──────────────────────────────────────────────────┐
  │  grafana.314vko  →  Grafana (на хосте, :3000)   │
  │  vault.314vko    →  Vault UI (в кластере)        │
  │  PostgreSQL      →  прямой доступ к БД           │
  └──────────────────────────────────────────────────┘
```

**GitOps-слой (Flux v2)** следит за этим репозиторием и применяет изменения в кластере автоматически.  
**Flux Image Automation** отслеживает новые образы в GHCR и автоматически обновляет теги в репо.  
**Ansible** отвечает за первичную подготовку сервера (конфигурация ОС, установка k8s, Grafana на хосте).

</details>

---

## Компоненты
<details>

**Инфраструктура кластера**

| Компонент              | Роль                                         | ~RAM      |
|------------------------|----------------------------------------------|-----------|
| k8s (kubeadm)          | Kubernetes (single-node, bare-metal)         | 500 MB    |
| Cilium                 | CNI, сеть между подами                       | 100 MB    |
| Nginx Ingress          | Ingress-контроллер, hostNetwork (80/443)     | 300 MB    |
| Flux v2                | GitOps-оператор (kustomize + helm controller)| 150 MB    |
| Flux Image Automation  | Автообновление тегов образов в GitOps репо   | —         |
| cert-manager           | Автоматические TLS сертификаты (Let's Encrypt)| 60 MB   |
| HashiCorp Vault        | Управление секретами + Agent Injector        | 256 MB    |
| kube-prometheus-stack  | Prometheus (мониторинг, retention 4d)        | ~300 MB   |
| local-path provisioner | hostPath PV для Vault и Prometheus           | —         |

**Приложения (namespace: apps)**

| Приложение     | Описание                                    | Домен              | ~RAM  |
|----------------|---------------------------------------------|--------------------|-------|
| fingo          | Go backend + nginx sidecar, секреты из Vault| fingo.ink          | 250 MB|
| gotanks        | Go multiplayer-игра (tanks), секреты из Vault| tanks.fingo.ink   | 250 MB|
| alexstav       | Статический сайт (nginx)                    | alexstav.dev       | 64 MB |
| mark-alldevops | Статический сайт (nginx), auto-image-update | mark.alldevops.ru  | 64 MB |

**На хосте (вне кластера)**

| Компонент  | Роль                                                          |
|------------|---------------------------------------------------------------|
| PostgreSQL | Основная БД для fingo и gotanks                               |
| Redis      | Кэш / брокер на хосте (Ansible, systemd), доступен из подов   |
| Grafana    | Дашборды (управляется через Ansible), Ingress → grafana.314vko|
| WireGuard  | VPN для внутреннего доступа к Grafana, Vault UI, PostgreSQL   |


</details>

---

## Установка кластера вручную

<details>
### Подготовка ОС

Перед установкой Kubernetes необходимо подготовить систему.

**1. Отключить swap**
```bash
swapoff -a
sed -i 's/^\(.*\sswap\s.*\)$/# \1/' /etc/fstab
```
Kubernetes требует отключённого swap — иначе kubelet отказывается стартовать. При активном swap планировщик k8s не может корректно управлять памятью подов.

**2. Загрузить модули ядра**
```bash
modprobe overlay
modprobe br_netfilter

cat <<EOF > /etc/modules-load.d/kubernetes.conf
overlay
br_netfilter
EOF
```
- `overlay` — нужен containerd для работы с layered filesystem (образы контейнеров хранятся слоями)
- `br_netfilter` — позволяет iptables обрабатывать трафик на уровне bridge между подами

**3. Настроить sysctl**
```bash
cat <<EOF > /etc/sysctl.d/kubernetes.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
```
- `bridge-nf-call-iptables` — без этого трафик между подами обходит iptables и network policies не работают
- `ip_forward` — разрешает серверу пересылать пакеты между интерфейсами (необходимо для pod networking)

---

### Установка компонентов

**4. containerd** — container runtime, запускает контейнеры
```bash
curl -LO https://github.com/containerd/containerd/releases/download/v2.1.0/containerd-2.1.0-linux-amd64.tar.gz
tar -C /usr/local -xzf containerd-2.1.0-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
mv containerd.service /etc/systemd/system/
systemctl daemon-reload && systemctl enable --now containerd
```

Конфигурация containerd:
```bash
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
# Включаем systemd cgroup driver (должен совпадать с kubelet)
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
# Указываем crun как OCI runtime
sed -i 's|runtime_type = "io.containerd.runc.v2"|runtime_type = "io.containerd.runc.v2"\n            BinaryName = "/usr/local/bin/crun"|' /etc/containerd/config.toml
systemctl restart containerd
```

**5. crun** — OCI runtime (вместо runc). Написан на C, потребляет меньше памяти и быстрее стартует контейнеры
```bash
curl -LO https://github.com/containers/crun/releases/download/1.20/crun-1.20-linux-amd64
install -m 755 crun-1.20-linux-amd64 /usr/local/bin/crun
```

**6. runc** — fallback OCI runtime, kubeadm проверяет его наличие при preflight
```bash
curl -LO https://github.com/opencontainers/runc/releases/download/v1.2.6/runc.amd64
install -m 755 runc.amd64 /usr/local/bin/runc
```

**7. CNI plugins** — низкоуровневые сетевые плагины, containerd использует их для настройки сети контейнеров
```bash
mkdir -p /opt/cni/bin
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz
tar -C /opt/cni/bin -xzf cni-plugins-linux-amd64-v1.6.2.tgz
```

**8. kubeadm, kubelet, kubectl**
```bash
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubeadm"
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubelet"
curl -LO "https://dl.k8s.io/release/v1.33.0/bin/linux/amd64/kubectl"
install -m 755 kubeadm kubelet kubectl /usr/local/bin/
```
- `kubeadm` — инструмент для bootstrap кластера
- `kubelet` — агент на каждом узле, запускает поды через container runtime
- `kubectl` — CLI для управления кластером

Настройка kubelet как systemd сервиса:
```bash
curl -LO https://raw.githubusercontent.com/kubernetes/release/v0.18.0/cmd/krel/templates/latest/kubelet/kubelet.service
sed -i 's|/usr/bin/kubelet|/usr/local/bin/kubelet|' kubelet.service
mv kubelet.service /etc/systemd/system/
mkdir -p /etc/systemd/system/kubelet.service.d
curl -LO https://raw.githubusercontent.com/kubernetes/release/v0.18.0/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf
sed -i 's|/usr/bin/kubelet|/usr/local/bin/kubelet|' 10-kubeadm.conf
mv 10-kubeadm.conf /etc/systemd/system/kubelet.service.d/
systemctl daemon-reload && systemctl enable kubelet
```
kubelet не запускается вручную — `kubeadm init` поднимет его сам.

**9. Bootstrap кластера**
```bash
kubeadm init --pod-network-cidr=10.244.0.0/16 --cri-socket=unix:///run/containerd/containerd.sock --kubernetes-version=v1.33.0
```
- `--pod-network-cidr` — подсеть для подов, Cilium будет использовать её
- `--cri-socket` — явно указываем containerd как runtime
- После успешного init настроить kubectl:
```bash
echo 'export KUBECONFIG=/etc/kubernetes/admin.conf' >> ~/.bashrc
```

**10. Cilium CNI**
```bash
curl -LO https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz
tar -C /usr/local/bin -xzf cilium-linux-amd64.tar.gz
cilium install --version 1.19.3
cilium status --wait
```
Без CNI нода остаётся в статусе `NotReady` — поды не могут общаться между собой.

**11. Nginx Ingress Controller**
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.15.1/deploy/static/provider/baremetal/deploy.yaml
```

На single-node кластере нужно снять taint с control-plane ноды чтобы поды могли на ней запускаться:
```bash
kubectl taint nodes <node-name> node-role.kubernetes.io/control-plane:NoSchedule-
```

Переключаем ingress на `hostNetwork` чтобы nginx слушал напрямую на портах 80/443 ноды:
```bash
kubectl patch deployment ingress-nginx-controller -n ingress-nginx --type=json -p='[{"op":"add","path":"/spec/template/spec/hostNetwork","value":true}]'
```

</details>

---

## Стек

- **Конфигурация ОС**: Ansible (firewall, Grafana, Redis, k8s bootstrap)
- **Оркестрация**: Kubernetes (kubeadm, single-node, bare-metal)
- **Container runtime**: containerd + crun
- **CNI**: Cilium
- **GitOps**: Flux v2 — Kustomize + HelmRelease
- **Image Automation**: Flux Image Automation (auto-update image tags в GitOps репо)
- **Ingress**: Nginx Ingress Controller (hostNetwork)
- **TLS**: cert-manager + Let's Encrypt
- **Секреты**: HashiCorp Vault + Vault Agent Injector
- **Мониторинг**: kube-prometheus-stack (Prometheus) + Grafana на хосте
- **БД**: PostgreSQL на хосте
- **Кэш**: Redis на хосте(Ansible, systemd)
- **Хранилище**: hostPath PV (local-path provisioner)
- **VPN**: WireGuard (внутренний доступ к Grafana, Vault UI, PostgreSQL)
- **Приложения**: fingo (Go + nginx), gotanks (Go), alexstav (nginx), mark-alldevops (nginx)

---

## Структура репозитория

```
k8s-gitops/
├── ansible/                  # Конфигурация ОС, firewall, Grafana, Redis
│   ├── inventory/
│   ├── playbooks/
│   └── roles/
│       ├── firewall/
│       ├── grafana/
│       └── redis/
│       └── kubernetes/
└── kubernetes/
    ├── flux-system/          # Bootstrapped Flux manifests + патчи ресурсов контроллеров
    ├── apps/                 # Приложения
    │   ├── fingo/            # Go backend + nginx sidecar (fingo.ink)
    │   ├── gotanks/          # Go multiplayer игра (tanks.fingo.ink)
    │   ├── alexstav/         # Статический сайт (alexstav.dev)
    │   └── mark-alldevops/   # Статический сайт (mark.alldevops.ru), auto-image-update
    └── infrastructure/
        ├── ingress-nginx/    # Ingress-контроллер (hostNetwork)
        ├── cert-manager/     # TLS сертификаты
        ├── vault/            # HashiCorp Vault + Injector + internal Ingress
        ├── monitoring/       # kube-prometheus-stack + Grafana Ingress (internal)
        ├── image-automation/ # Flux Image Automation (GHCR → GitOps auto-update)
        └── local-path/       # hostPath PV для Vault и Prometheus
```


## TODO

- [ ] Переход с Nginx Ingress Controller на Kubernetes Gateway API (замена `ingress-nginx` на нативный Gateway API с Cilium в роли реализации)

---

## Разработка
### Необходимо
 - Python 3.10+
 - `ansible` & `ansible-lint`
    ```bash
       pip install ansible ansible-lint
    ```
 - Коллекция `community.general`:
   ```bash
       ansible-galaxy collection install community.general
    ```
 - `pre-commit`:
    ```bash
       pip install pre-commit
       pre-commit install
    ```
После `pre-commit install` - при каждом коммите будет автоматически запускаться `ansible-lint`.
Проверить руками: `pre-commit run --all-files` | `ansible-lint ansible/`
