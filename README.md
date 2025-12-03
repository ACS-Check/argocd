# 🚀 ACS-Check / argocd  
**Kubernetes + ArgoCD GitOps 실습 예제**

Nginx ↔ Tomcat 기반 예제 애플리케이션을 Kubernetes에 배포하고,  
ArgoCD를 사용하여 GitOps 방식으로 자동 배포 환경을 구성하는 저장소입니다.

---

# 🧰 0. 시작 전 환경 세팅

이 저장소를 실행하기 전에 아래 환경이 준비되어 있어야 합니다.

---

## ✔ 1) Kubernetes 클러스터 준비

아래 모든 환경에서 적용 가능합니다:

- kubeadm 기반 Kubernetes(예: VMware/Hyper-V 온프레미스)
- Minikube / Kind
- Docker Desktop Kubernetes
- AWS EKS / NCP Kubernetes 등

상태 확인:
```bash
kubectl get nodes
kubectl cluster-info
```

---

## ✔ 2) kubectl 설치

확인:
```bash
kubectl version --client
```

설치:
- Ubuntu: `sudo snap install kubectl --classic`
- Mac: `brew install kubectl`
- Windows: Chocolatey → `choco install kubernetes-cli`

---

## ✔ 3) kubeconfig 설정

기본 경로:
```
~/.kube/config
```

불러오기:
```bash
scp root@<master-ip>:/etc/kubernetes/admin.conf ~/.kube/config
export KUBECONFIG=~/.kube/config
```

---

## ✔ 4) 필수 Add-on 설치

### (1) Metrics Server (HPA 사용 시 필수)
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

### (2) Ingress Controller (Ingress 사용 시 필수)
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

---

## ✔ 5) Git 설치

```bash
git --version
```

---

## ✔ 6) GitHub 토큰 준비 (중요)

ArgoCD는 Public Repo여도 HTTPS 기반 인증 때문에 Token을 요구할 수 있습니다.

필요한 토큰:
- Personal Access Token(Fine-grained)
- Organization Repo 접근 승인 필요

---

# 📦 Repository 구성

```text
📁 manifests/
 ├─ cc-nginx-deploy.yaml
 ├─ cc-nginx-svc.yaml
 ├─ cc-nginx-conf.yaml
 ├─ cc-nginx-hpa.yaml
 ├─ cc-tomcat-deploy.yaml
 ├─ cc-tomcat-svc.yaml
 ├─ cc-tomcat-hpa.yaml
 └─ cc-ingress.yaml    # host/TLS 수정 필수
```

---

# ⚡ Quick Start (kubectl)

### 전체 배포
```bash
kubectl apply -f .
```

### 상태 확인
```bash
kubectl get pods,svc,deploy,hpa -o wide
```

### 전체 삭제
```bash
kubectl delete -f .
```

---

# 🎯 ArgoCD GitOps 구성

## ✔ 1) ArgoCD 설치
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

---

# 🔐 GitHub Token 설정  
(PAT + Organization Repo 인증)

## ✔ Personal Access Token 생성
- Repository read  
- Metadata read  

## ✔ Organization Repo Token 승인
- Allow fine-grained tokens  
- Allow PAT  
- Repo Read 권한  
- Token 승인 필요  

---

## ✔ ArgoCD에 Repo Secret 생성

```bash
kubectl create secret generic repo-auth   -n argocd   --from-literal=username="<GITHUB_USERNAME>"   --from-literal=password="<GITHUB_PAT_OR_ORG_TOKEN>"
```

---

# 🎨 ArgoCD UI 기반 GitOps Workflow

## ✔ Step 1) ArgoCD 접속
```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
접속: `https://localhost:8080`

---

## ✔ Step 2) Repository 등록
UI → Settings → Repositories → Connect Repo

| 항목 | 내용 |
|------|------|
| URL | https://github.com/ACS-Check/argocd |
| Username | GitHub ID |
| Password | PAT 또는 Org Token |
| Type | HTTPS |

---

## ✔ Step 3) Application 생성
UI → Applications → NEW APP

- Name: `cc-app`
- Project: `default`
- Repo: 등록한 Repo
- Revision: `HEAD`
- Path: `.`
- Destination: `https://kubernetes.default.svc`
- Namespace: `default`
- Sync Policy: Auto-sync, Self Heal, Prune(optional)

---

## ✔ Step 4) SYNC 실행
UI에서 **SYNC** 클릭 → 자동 배포 진행

---

# 🔧 Troubleshooting

### 🔹 Sync Error
- Token 권한 부족  
- Org 승인 없음  
- Path/Branch 오류  

### 🔹 Ingress 오류
```bash
kubectl describe ingress cc-ingress
```

### 🔹 HPA 오류
```bash
kubectl top pods
kubectl describe hpa cc-nginx
```

### 🔹 ConfigMap 수정 반영
```bash
kubectl rollout restart deploy/cc-nginx
```

---

# 🤝 Contributing
PR / Issue 환영합니다.

---

# 📮 Contact
Maintainer: **ACS-Check**
