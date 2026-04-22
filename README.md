# k8s-gitops

Пет-проект для изучения IaC (Ansible) и GitOps-подхода.  
Цель — поднять production-like окружение на одном удалённом сервере.

---

## Архитектура
<details>

```
                        Internet
                           │
                        80/443
                           │
                    ┌──────▼──────┐
                    │Nginx Ingress│  Ingress / TLS termination (hostNetwork)
                    └──────┬──────┘
                           │
          ┌────────────────┼────────────────┐
          │                │                │
   ┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
   │  Nginx      │  │   Go app    │  │  (будущие   │
   │  (static)   │  │  (backend)  │  │   сервисы)  │
   └─────────────┘  └──────┬──────┘  └─────────────┘
                           │
                    ┌──────▼──────┐
                    │ PostgreSQL  │  основная БД
                    └─────────────┘

```

**GitOps-слой (Flux v2)** следит за этим репозиторием и применяет изменения в кластере автоматически.  
**Ansible** отвечает за первичную подготовку сервера (конфигурация ОС, установка k8s).

</details>

---

## Компоненты
<details>

| Компонент         | Роль                                      | ~RAM        |
|-------------------|-------------------------------------------|-------------|
| k8s (kubeadm)     | Kubernetes (single-node, bare-metal)      | 500 MB      |
| Cilium            | CNI, сеть между подами                    | 100 MB      |
| Nginx Ingress     | Ingress-контроллер, TLS (Let's Encrypt)   | 60 MB       |
| Flux v2           | GitOps-оператор (sync этого репо)         | 150 MB      |
| cert-manager      | Автоматические TLS сертификаты            | 60 MB       |
| Go app            | Основной бэкенд                           | 80 MB       |
| PostgreSQL        | Основная реляционная БД                   | 300 MB      |
| Nginx             | Раздача статики                           | 30 MB       |
| **Итого**         |                                           | **~1.3 GB** |

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

- **Конфигурация ОС**: Ansible
- **Оркестрация**: Kubernetes (kubeadm, single-node)
- **Container runtime**: containerd + crun
- **CNI**: Cilium
- **GitOps**: Flux v2 — Kustomize + HelmRelease
- **Ingress**: Nginx Ingress Controller
- **TLS**: cert-manager + Let's Encrypt
- **БД**: PostgreSQL (in-cluster, PVC)
- **Бэкенд**: Go-приложение
- **Статика**: Nginx

---

## Структура репозитория

```
k8s-gitops/
├── ansible/            # Конфигурация ОС и установка k8s компонентов
└── kubernetes/
    ├── flux-system/    # Bootstrapped Flux manifests
    ├── apps/           # HelmRelease / Kustomize для сервисов
    └── infrastructure/ # Nginx Ingress, cert-manager, storage classes
```


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
