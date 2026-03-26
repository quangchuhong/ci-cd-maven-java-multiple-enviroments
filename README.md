# CI/CD cho ứng dụng Maven/Java trên EKS với GitLab – Jenkins – SonarQube – Trivy – ECR – Helm – Argo CD

## 1. Mục tiêu

Thiết kế kiến trúc CI/CD hoàn chỉnh cho các ứng dụng **Maven/Java** chạy trên **Amazon EKS**, với:

- 3 môi trường: **test**, **staging**, **prod**
- CI: **GitLab → Jenkins → SonarQube → Trivy → ECR**
- CD (GitOps): **Jenkins → GitOps repo → Argo CD → EKS**
- Deploy: **Helm chart**

---

## 2. Luồng tổng thể

```text
Dev → GitLab → Jenkins CI → ECR + GitOps repo → Argo CD → EKS
```
Chi tiết:
1. Dev push code lên GitLab.
2. GitLab webhook trigger Jenkins pipeline.
3. Jenkins CI:
    - Build & test với Maven.
    - Phân tích chất lượng với SonarQube.
    - Build Docker image, scan Trivy.
    - Push image lên AWS ECR.
    - Cập nhật GitOps repo (Helm values: image tag).
4. Argo CD theo dõi GitOps repo:
Thấy thay đổi values.yaml → sync → deploy version mới lên EKS (theo từng môi trường).
---
## 3. Kiến trúc tổng quan
```text
┌──────────┐      ┌───────────┐      ┌────────┐
│ Developer│  git │  GitLab   │hook  │ Jenkins│
└────┬─────┘      └────┬──────┘────►└────┬───┘
     │                 │                CI│
     │                 │                  │
     │         ┌───────▼───────┐         │
     │         │ app-repo      │         │
     │         │ (code,        │         │
     │         │  Dockerfile,  │         │
     │         │  Jenkinsfile, │         │
     │         │  Helm chart)  │         │
     │         └───────────────┘         │
     │                                    │
     │                                    │
     │     ┌──────────────┐       ┌───────▼────────┐
     │     │ SonarQube    │       │ Trivy          │
     │     └──────────────┘       └───────┬────────┘
     │                                     │
     │                            ┌────────▼─────────┐
     │                            │ AWS ECR (images) │
     │                            └────────┬─────────┘
     │                                     │
     │                           ┌─────────▼──────────┐
     │                           │ gitops-repo        │
     │                           │ (test/staging/prod │
     │                           │  values.yaml)      │
     │                           └─────────┬──────────┘
     │                                     │
     │                           ┌─────────▼─────────┐
     │                           │  Argo CD          │
     │                           └─────────┬─────────┘
     │                                     │
     │   test namespace          staging   │   prod
     │  ┌──────────────┐       ┌──────────▼──────┐
     └─►   EKS cluster │  ...  │   EKS cluster   │ (hoặc chung cluster,
        └──────────────┘       └─────────────────┘  khác namespace)
```
---
## 4. Thiết kế repository & nhánh
### 4.1. Repo ứng dụng (app-repo)
```text
app-repo/
├── src/
├── pom.xml
├── Dockerfile
├── Jenkinsfile
└── helm/
    └── my-maven-app/
        ├── Chart.yaml
        ├── values.yaml          # values chung / mặc định
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── ingress.yaml (nếu dùng)
```
**Branch đề xuất**: 

    - develop → build & deploy tới test
    - release/* → promote/build & deploy tới staging
    - main → promote/build & deploy tới prod

### 4.2. Repo GitOps (gitops-repo)

Repo riêng dùng cho Argo CD:
```text
gitops-repo/
├── test/
│   └── values.yaml
├── staging/
│   └── values.yaml
└── prod/
    └── values.yaml

```
- Mỗi values.yaml override ít nhất:
    - image.repository
    - image.tag
    - cấu hình env, replica, resources theo môi trường.

## 5. Cấu trúc Helm & values
### 5.1. values.yaml chung trong app-repo

`app-repo/helm/my-maven-app/values.yaml`:

```text
image:
  repository: <your-ecr-repo-url>/my-maven-app
  tag: "latest"
  pullPolicy: IfNotPresent

replicaCount: 1

env: []
resources: {}

```
deployment.yaml trích đoạn:
```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-maven-app.fullname" . }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ include "my-maven-app.name" . }}
  template:
    metadata:
      labels:
        app: {{ include "my-maven-app.name" . }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          env:
            {{- range .Values.env }}
            - name: {{ .name }}
              value: {{ .value | quote }}
            {{- end }}

```
### 5.2. values theo môi trường trong gitops-repo

gitops-repo/test/values.yaml:
```text
image:
  repository: <your-ecr-repo-url>/my-maven-app
  tag: "dev-<sha>"   # Jenkins sẽ cập nhật

replicaCount: 1

env:
  - name: SPRING_PROFILES_ACTIVE
    value: "test"

```

