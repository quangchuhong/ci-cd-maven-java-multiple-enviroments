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
