# Содержание
* [Деплой приложений](#деплой-приложенийgitops--flux)
* [Паттерн А - простое приложение(один образ)](#паттерн-а---простое-приложениеодин-образ)
* [Паттерн Б - авто-обновление образа(Image Automation)](#паттерн-б---авто-обновление-образаimage-automation)
* [Паттерн С - Helm-чарт `webapp` / внешний репозиторий](#паттерн-с---helm-чарт-webapp--внешний-репозиторий)
* [Проверка после деплоя](#проверка-после-деплоя)
# Деплой приложений(GitOps / Flux)
Кластер управляется через **Flux v2**: все что в ветке `main` этого репозитория, автоматически применяется в кластер.

## Как Flux находит приложения
```
flux-system/kustomization.yaml
└── infrastructure.yaml          # Flux-Kustomization "apps" → path: ./kubernetes/apps
    └── apps/kustomization.yaml   # список всех приложений
        └── <app>/kustomization.yaml
```
Чтобы приложенгие подтянулось, его папку нужно добавить в `apps/kustomization.yaml`.
Все приложения живут в namespace `apps`(создается в `apps/namespace.yaml`).
После пуша Flux подхватит все изменения в течении ~10 минут автоматически.
Подтянуть изменения вручную:
```bash
flux reconcile kustomization apps --with-source
flux get kustomizations                         # статус
```

### Паттерн А - простое приложение(один образ)
Подходит для статики и одиночных контейнеров(пример: `фронты`).
1. Создать папку `kubernetes/apps/<app_name>/` с четырьмя файлами:

#### `deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
    name: <app_name>
    namespace: apps
spec:
    replicas: 1
    selector:
        matchLabels:
            app: <app_name>
    template:
        metadata:
            labels:
                app: <app_name>
        spec:
            containers:
                - name: <app_name>
                  image: ghcr.io/mark-grigorev/<app_name>:latest
                  ports:
                    - containerPort: 80
                  resources:
                    requests: { cpu: 10m, memory: 32Mi }
                    limits: { cpu: 20m, memory: 64Mi }
```
#### `service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
    name: <app_name>
    namespace: apps
spec:
    selector:
        app: <app_name>
    ports:
        - port: 80
          targetPort: 80
```
#### `ingress.yaml` (TLS - через cert-manager + Let's Encrypt)
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
    name: <app_name>
    namespace: apps
    annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
    ingressClassName: nginx
    tls:
        - hosts: [<app_name>.example.com]
          http:
            paths:
                - path: /
                  pathType: Prefix
                  backend:
                    service:
                        name: <app_name>
                        port:
                            number: 80
```
#### `kustomization.yaml`
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
    - deployment.yaml
    - service.yaml
    - ingress.yaml
```

2. Зарегистрировать приложение в `kubernetes/apps/kustomization.yaml`:
```yaml
resources:
    - namespace.yaml
    - alexstav/
    - <app_name>/
```

3. Закоммитить и запушить в `main`.
> Если образ в **приватном** GHCR - добавить в `deployment.yaml` (spec.template.spec):
```yaml
imagePullSecrets:
    -  name: ghcr-mark # или иной
```

---

### Паттерн Б - авто-обновление образа(Image Automation)
Flux сам следит за новыми тегами в реестре, обновляет тег в манифесте и коммитит в репо.
1. Развернуть приложение по *Паттерну А*
2. В `deployment.yaml` пометить образ сеттером(обязательно):
```yaml
image: ghcr.io/mark-grigorev/<app_name>:latest #{"$imagepolicy": "flux-system: <app_name>"}
```
3. Создать `kubernetes/infrastructure/image-automation/<app_name>/` с тремя файлами:
    - `image-repository.yaml` - `ImageRepository`(какой образ опрашивать, `secretRef : ghcr-mark`);
    - `image-policy.yaml` - `ImagePolicy`(правило выбора тега, например `semver: '>=0.0.1'`);
    - `image-update-automation.yaml` - `ImageUpdateAutomation`( `update.path: ./kubernetes/apps/<app_name>`, push в main);
    - `kustomization.yaml` - со списком этих трех файлов.
4. Зарегистрировать в `kubernetes/infrastructure/image-automation/kustomization.yaml`:
```yaml
resources:
    - mark-alldevops/
    - <app_name>/
```

---

### Паттерн С - Helm-чарт `webapp` / внешний репозиторий

Для приложений посложнее(env, команда запуска, Vault-сайдкар, nginx-сайдкар) есть generic-чарт `kubernetes/charts/webapp`(пример: `vacancy-parser`).
Два варианта:
 - **Манифесты из внешнего репо** - `GitRepository` + Flux-`Kustomization`:
(`path: ./deploy`, `targetNamespace: apps`), см. `vacancy-parser/gitrepository.yaml` и `flux-kustomization.yaml`.
 - **HelmRelease на чарте `webapp`** - `helmrelease.yaml` с `valuesFrom`(ConfigMap), параметры чарта смотри в `kubernetes/charts/webapp/values.yaml`(`image`, `ingress.host`, `env`, `vault.*`, `nginx.*`).

Папка приложения собирается своим `kustomization.yaml` и так же добавляется в `apps/kustomization.yaml`.

### Проверка после деплоя
```bash
flux get kustomizations         # применился ли набор
flux get helmreleases -n apps   # для паттерна С
kubectl -n apps get pods,ingress
kubectl -n apps describe ingress <app_name> # выпуск TLS-сертификата
```