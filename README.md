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
gitops-repo/staging/values.yaml:

```text
image:
  repository: <your-ecr-repo-url>/my-maven-app
  tag: "stg-<sha>"   # Jenkins cập nhật

replicaCount: 2

env:
  - name: SPRING_PROFILES_ACTIVE
    value: "staging"

```
gitops-repo/prod/values.yaml:

```text
image:
  repository: <your-ecr-repo-url>/my-maven-app
  tag: "prod-<sha-or-version>"

replicaCount: 3

env:
  - name: SPRING_PROFILES_ACTIVE
    value: "prod"

```
---
## 6. Jenkins CI Pipeline (Jenkinsfile)
### 6.1. Mục tiêu pipeline
- Trigger từ GitLab webhook trên các branch: develop, staging, main.
- Các bước:
    1. Checkout code
    2. Build & test Maven
    3. SonarQube analysis (+ Quality Gate)
    4. Build Docker image
    5. Scan Trivy
    6. Push image lên ECR
    7. Update GitOps repo (values.yaml đúng môi trường)
    8. (Option) Manual approval trước khi update prod
       
### 6.2. Jenkinsfile mẫu (rút gọn, cần chỉnh lại ID/URL cụ thể)
```text
pipeline {
    agent any

    environment {
        APP_NAME        = "my-maven-app"

        // ECR repo, ví dụ: 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com/my-maven-app
        ECR_REPOSITORY  = "<your-account-id>.dkr.ecr.<region>.amazonaws.com/my-maven-app"

        // GitOps repo chứa các values.yaml theo môi trường
        GITOPS_REPO_URL = "git@gitlab.com:yourgroup/gitops-repo.git"

        SONARQUBE_ENV   = "sonarqube"   // tên server trong Jenkins > Configure System
        AWS_CREDENTIALS = "aws-creds"   // ID Jenkins credential kiểu AWS
        GIT_CREDENTIALS = "gitlab-ssh"  // ID Jenkins credential để push gitops-repo
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    GIT_SHORT_SHA = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build & Test (Maven)') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(SONARQUBE_ENV) {
                    sh """
                       mvn sonar:sonar \
                         -Dsonar.projectKey=${APP_NAME} \
                         -Dsonar.projectName='${APP_NAME}'
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def envPrefix
                    if (env.BRANCH_NAME == "develop") {
                        envPrefix = "dev"
                    } else if (env.BRANCH_NAME == "staging") {
                        envPrefix = "stg"
                    } else if (env.BRANCH_NAME == "main") {
                        envPrefix = "prod"
                    } else {
                        envPrefix = "dev"
                    }

                    IMAGE_TAG = "${envPrefix}-${GIT_SHORT_SHA}"

                    sh """
                       docker build -t ${ECR_REPOSITORY}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Trivy Scan') {
            steps {
                sh """
                   trivy image --exit-code 1 --severity HIGH,CRITICAL ${ECR_REPOSITORY}:${IMAGE_TAG}
                """
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([aws(credentialsId: AWS_CREDENTIALS, region: '<region>')]) {
                    sh """
                       aws ecr get-login-password --region <region> \
                         | docker login --username AWS --password-stdin ${ECR_REPOSITORY%/*}

                       docker push ${ECR_REPOSITORY}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Update GitOps Repo') {
            steps {
                script {
                    def envPath
                    if (env.BRANCH_NAME == "develop") {
                        envPath = "test"
                    } else if (env.BRANCH_NAME == "staging") {
                        envPath = "staging"
                    } else if (env.BRANCH_NAME == "main") {
                        envPath = "prod"
                    } else {
                        envPath = "test"
                    }

                    dir('gitops-repo') {
                        git url: GITOPS_REPO_URL,
                            branch: 'main',
                            credentialsId: GIT_CREDENTIALS

                        // Cập nhật image.tag trong values.yaml (cần có yq trên agent)
                        sh """
                           yq e '.image.tag = "${IMAGE_TAG}"' -i ${envPath}/values.yaml
                        """

                        sh """
                           git config user.email "jenkins-bot@example.com"
                           git config user.name  "Jenkins Bot"
                           git commit -am "Update ${APP_NAME} image tag to ${IMAGE_TAG} for ${envPath}" || echo "No changes"
                           git push origin main
                        """
                    }
                }
            }
        }
    }
}

```
```
Lưu ý:

    - Cần cài yq trên Jenkins agent để chỉnh giá trị YAML.
    - env.BRANCH_NAME phụ thuộc bạn dùng Freestyle hay Multibranch Pipeline.
    - Có thể thêm stage manual approval trước khi update prod.
```
---
## 7. Argo CD (CD – GitOps)
### 7.1. Ứng dụng test

```text
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-maven-app-test
spec:
  project: default
  source:
    repoURL: git@gitlab.com:yourgroup/gitops-repo.git
    targetRevision: main
    path: test
    helm:
      # tuỳ thiết kế: dùng chart local hoặc chart trong repo riêng
      valueFiles:
        - values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: my-maven-app-test
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

```
```
Tương tự cho:

    - my-maven-app-staging (path: staging, namespace: my-maven-app-staging)
    - my-maven-app-prod (path: prod, namespace: my-maven-app-prod)

Với prod, có thể:

    - Để syncPolicy.automated = off (manual sync),
    - Hoặc vẫn auto-sync nhưng pipeline Jenkins có step “approval” trước khi update GitOps repo.
```
---

## 8. Promotion giữa 3 môi trường

**Nguyên tắc**: Build 1 lần, deploy nhiều nơi (không rebuild lại cho staging/prod).

1. Dev push vào develop:

    - Jenkins build → image tag: dev-<sha>
    - Cập nhật test/values.yaml → image.tag = dev-<sha>
    - Argo CD deploy bản này lên test.

2. Khi test OK:

    - Merge/Cherry-pick sang branch staging nhưng vẫn dùng lại <sha> đó:
        - Jenkins pipeline cho staging không bắt buộc phải build lại,
        - chỉ update staging/values.yaml với tag = stg-<sha> hoặc tái sử dụng dev-<sha>.
          
3. Khi staging OK:
   - Tương tự, merge sang main và deploy prod với cùng artifact (image SHA) đã chạy ổn ở staging.
---

## 9. Bảo mật & best practices (tóm tắt)

**- Credentials:**

    - Lưu AWS credential, GitLab SSH key, Sonar token, v.v trong Jenkins Credentials.
    
**- ECR & Image:**

    - Dùng tag bất biến (SHA), tránh latest trong prod.

**- Trivy:**

- Có thể:
    - test: cảnh báo
    - staging/prod: fail nếu HIGH/CRITICAL.

**- SonarQube Quality Gate:**

    - Bắt buộc pass trước khi build image cho staging/prod.
    
**- Argo CD:**

    - Gán quyền chỉ đọc cho Dev, quyền sync/rollback cho DevOps/ops.

