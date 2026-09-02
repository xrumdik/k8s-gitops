# k8s-gitops — манифесты для Argo CD

Это репозиторий **только с k8s-манифестами** — "источник истины" в GitOps-модели. За ним следит Argo CD и применяет изменения в кластер автоматически. Код приложения и Dockerfile — в отдельном репозитории `social-eng-app`.

## Структура

```
.
├── apps/
│   └── social-eng-app/
│       ├── namespace.yaml
│       ├── deployment.yaml       # Deployment (replicas: 2) + Service, с topologySpreadConstraints
│       └── kustomization.yaml
└── argocd-application.yaml       # Application-ресурс — применяется ОДИН РАЗ вручную (см. ниже)
```

Если у вас уже есть репозиторий `k8s-gitops` (например, с демо-приложением из гайда по Argo CD) — просто добавьте в него папку `apps/social-eng-app/` из этого архива, ничего в существующей структуре менять не нужно.

## Как развернуть — шаг за шагом

### 1. Замените плейсхолдеры логина

В двух файлах замените `<ваш-github-username>` на ваш реальный логин GitHub:

- `apps/social-eng-app/deployment.yaml` — строка `image: ghcr.io/<ваш-github-username>/social-eng-app:latest`
- `argocd-application.yaml` — строка `repoURL: https://github.com/<ваш-github-username>/k8s-gitops.git`

**PowerShell (Windows)**:
```powershell
(Get-Content apps\social-eng-app\deployment.yaml) -replace '<ваш-github-username>', 'ваш_реальный_логин' | Set-Content apps\social-eng-app\deployment.yaml
(Get-Content argocd-application.yaml) -replace '<ваш-github-username>', 'ваш_реальный_логин' | Set-Content argocd-application.yaml
```

**Git Bash / Linux**:
```bash
sed -i 's/<ваш-github-username>/ваш_реальный_логин/g' apps/social-eng-app/deployment.yaml argocd-application.yaml
```

### 2. Создайте репозиторий на GitHub (если ещё не создан)

`github.com → New repository → k8s-gitops`.

Если репозиторий уже существует — просто клонируйте его и скопируйте туда содержимое папки `apps/social-eng-app/` из этого архива (плюс `argocd-application.yaml` в корень, если там ещё нет своего).

### 3. Загрузите репозиторий

Если репозиторий новый:

```bash
git init
git add .
git commit -m "Add social-eng-app manifests"
git branch -M main
git remote add origin https://github.com/ваш_логин/k8s-gitops.git
git push -u origin main
```

Если репозиторий уже существовал — просто добавьте новые файлы и запушьте:

```bash
git add apps/social-eng-app argocd-application.yaml
git commit -m "Add social-eng-app manifests"
git push
```

### 4. Создайте Application в Argo CD

На **master** вашего k8s-кластера (скопируйте туда файл `argocd-application.yaml`, например через `scp`, либо просто наберите вручную командой ниже):

```bash
kubectl apply -f argocd-application.yaml
```

Или прямо inline, без файла:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: social-eng-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/ваш_логин/k8s-gitops.git
    targetRevision: main
    path: apps/social-eng-app
  destination:
    server: https://kubernetes.default.svc
    namespace: social-eng
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF
```

Проверка:

```bash
kubectl get application -n argocd
argocd app get social-eng-app
```

### 5. Проверка распределения по нодам

```bash
kubectl get pods -n social-eng -o wide
```

Ожидаем ровно по одному поду на `k8s-node1` и `k8s-node2` — гарантируется `topologySpreadConstraints` в `deployment.yaml`.

### 6. Открыть сайт

```
http://ip-address:30902
```

(или через любой из трёх узлов — сервис `NodePort`)

Обновите страницу несколько раз — строка "Обслужено подом" внизу должна чередоваться между двумя репликами.
