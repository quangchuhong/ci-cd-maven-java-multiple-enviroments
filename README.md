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
