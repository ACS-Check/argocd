# 🚀 ACS-Check / argocd  
**Kubernetes + ArgoCD GitOps 실습 예제**

Nginx ↔ Tomcat 기반 예제 애플리케이션을 Kubernetes에 배포하고,  
ArgoCD를 사용하여 GitOps 방식으로 자동 배포 환경을 구성하는 저장소입니다.

---

# 🧰 0. 시작 전 환경 세팅  
(⚠️ Kubernetes 클러스터는 이미 설치되어 있다고 가정)

이 저장소는 **이미 Kubernetes 클러스터가 Ready 상태**라는 전제 하에 동작합니다.  
추가적인 클러스터 설치 과정 없이 아래 요소만 준비되면 바로 실행할 수 있습니다.

---

## ✔ 1) kubectl 연결 확인

```bash
kubectl get nodes
kubectl get pods -A
```

정상 출력 예:

```
NAME           STATUS   ROLES           AGE   VERSION
master-node    Ready    control-plane   20d   v1.29.x
worker-node1   Ready    <none>          20d   v1.29.x
```

---

## ✔ 2) kubeconfig는 이미 설정되어 있다고 가정

kubectl 명령이 정상적으로 실행된다면 추가 설정은 필요 없습니다.

---

## ✔ 3) 필요한 Add-on 두 가지만 확인

### (1) Metrics Server (HPA 사용 시 필수)

설치 여부 확인:

```bash
kubectl top nodes
```

미설치 시:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

### (2) Ingress Controller (Ingress 사용 시 필수)

설치 여부 확인:

```bash
kubectl get pods -n ingress-nginx
```

미설치 시:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

---

## ✔ 4) GitHub Token 준비

ArgoCD에서 GitHub Repo를 읽어오기 위해 필요합니다.

- Personal Access Token (Fine-grained)
- Organization Repo일 경우 Org 승인 필요

자세한 내용은 아래 Token 설정 섹션 참고.

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

초기 admin 비밀번호:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret   -o jsonpath="{.data.password}" | base64 -d
```

---

# 🔐 GitHub Token 설정 (PAT + Organization Repo 인증)

## ✔ Personal Access Token(Fine-grained) 생성

필요 권한:
- Repository: Read-only  
- Contents: Read  
- Metadata: Read  

---

## ✔ Organization Repo Token 승인 (중요)

Organization Owner가 해야 하는 작업:

1. Settings → Security → Personal Access Tokens  
   - ✔ Allow fine-grained tokens  
   - ✔ Allow PAT usage  
2. People → 해당 사용자 Repo Read 권한 부여  
3. Settings → Requests → Token 승인  

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

브라우저:  
```
https://localhost:8080
```

---

## ✔ Step 2) Repository 등록

UI → **Settings → Repositories → Connect Repo**

| 항목 | 내용 |
|------|------|
| URL | https://github.com/ACS-Check/argocd |
| Username | GitHub ID |
| Password | PAT 또는 Org Token |
| Type | HTTPS |

---

## ✔ Step 3) Application 생성

UI → **Applications → NEW APP**

- Name: `cc-app`
- Project: `default`
- Repo: 등록한 Repo
- Revision: `HEAD`
- Path: `.`
- Cluster: `https://kubernetes.default.svc`
- Namespace: `default`
- Sync Policy:  
  - Auto-sync  
  - Self Heal  
  - Prune(optional)

---

## ✔ Step 4) Sync (배포 실행)

UI에서 **SYNC** 클릭 → 자동 배포 진행

---

# 🔧 Troubleshooting

### 🔹 Sync Error
- Token 권한 부족  
- Organization 승인 누락  
- Repo 주소 또는 Path 오류  

### 🔹 Ingress 문제
```bash
kubectl describe ingress cc-ingress
```

### 🔹 HPA 문제
```bash
kubectl top pods
kubectl describe hpa cc-nginx
```

### 🔹 ConfigMap 변경 반영
```bash
kubectl rollout restart deploy/cc-nginx
```

---

# 🤝 Contributing

PR / Issue 환영합니다.

---

# 📮 Contact
Maintainer: **ACS-Check**
