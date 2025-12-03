🚀 ACS-Check / argocd

Kubernetes + ArgoCD GitOps 실습 예제

Nginx ↔ Tomcat 기반 예제 애플리케이션을 Kubernetes에 배포하고,
ArgoCD를 사용하여 GitOps 방식으로 자동 배포 환경을 구성하는 저장소입니다.

🧰 0. 시작 전 환경 세팅

이 저장소를 실행하기 전에 아래 환경이 준비되어 있어야 합니다.

✔ 1) Kubernetes 클러스터

아래 모든 환경에서 적용 가능:

kubeadm 기반 Kubernetes(예: VMware/Hyper-V 온프레미스)

Minikube / Kind

Docker Desktop Kubernetes

AWS EKS / NCP Kubernetes 등

상태 확인:

kubectl get nodes
kubectl cluster-info

✔ 2) kubectl 설치

확인:

kubectl version --client


설치:

Ubuntu: sudo snap install kubectl --classic

Mac: brew install kubectl

Windows: Chocolatey → choco install kubernetes-cli

✔ 3) kubeconfig 설정

기본 경로:

~/.kube/config


불러오기:

scp root@<master-ip>:/etc/kubernetes/admin.conf ~/.kube/config
export KUBECONFIG=~/.kube/config

✔ 4) 필수 Add-on
(1) Metrics Server (HPA 필요)
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

(2) Ingress Controller

(이 Repository에서 Ingress 사용 시 필수)

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml

✔ 5) Git 설치
git --version

✔ 6) GitHub 토큰 준비(중요)

Public Repo여도 ArgoCD는 HTTPS 기반 Repo 인증 때문에 Token을 요구할 수 있습니다.

준비해야 할 토큰 종류

Personal Access Token(PAT)

Organization Repo라면 Organization Fine-grained Token 승인 필요

자세한 내용은 아래 "GitHub Token 설정" 섹션 참고.

📦 Repository 구성
📁 manifests/
 ├─ cc-nginx-deploy.yaml
 ├─ cc-nginx-svc.yaml
 ├─ cc-nginx-conf.yaml
 ├─ cc-nginx-hpa.yaml
 ├─ cc-tomcat-deploy.yaml
 ├─ cc-tomcat-svc.yaml
 ├─ cc-tomcat-hpa.yaml
 └─ cc-ingress.yaml    # host/TLS 수정 필수

⚡ Quick Start (kubectl)
전체 배포
kubectl apply -f .

상태 확인
kubectl get pods,svc,deploy,hpa -o wide

전체 삭제
kubectl delete -f .

🎯 ArgoCD GitOps 구성
✔ 1) ArgoCD 설치
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl -n argocd port-forward svc/argocd-server 8080:443


접속:

https://localhost:8080


초기 admin 비밀번호 확인:

kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

🔐 GitHub Token 설정 (PAT + Org Repo 인증)
✔ 1) Personal Access Token(Fine-grained)

GitHub → Settings → Developer settings → Fine-grained token

권한(추천):

Repository: Read-only

Metadata: Read

Contents: Read

✔ 2) Organization Repo Token 승인(중요)

Organization에서 다음이 설정되어 있어야 ArgoCD가 Repo를 읽을 수 있습니다:

Owner가 해야 하는 설정

Organization → Settings → Security → Personal Access Tokens

✔ Allow fine-grained tokens

✔ Allow PAT for members

Organization → People

Repo 읽기(Read) 권한 부여

Organization → Settings → Requests

Token 승인(Approve)

🎨 ArgoCD UI 기반 GitOps Workflow (핵심)

GitOps 전체 과정을 CLI 없이 UI 중심으로 진행하는 방식을 정리했습니다.

✔ Step 1) ArgoCD 접속
kubectl -n argocd port-forward svc/argocd-server 8080:443


브라우저 접속:

https://localhost:8080

✔ Step 2) ArgoCD UI에 Repository 등록

ArgoCD UI → Settings → Repositories → Connect Repo

입력값:

항목	값
Repository URL	https://github.com/ACS-Check/argocd
Username	GitHub ID
Password	PAT 또는 Org Token
Connection Type	HTTPS

💡 주의

Public Repo라도 Token 입력 필수적인 경우가 많습니다.

Private Repo라면 반드시 Token 필요.

등록 후 “Test” 버튼 → Successful 확인.

✔ Step 3) ArgoCD UI에서 Application 생성

UI → Applications → NEW APP

① General

Application Name: cc-app

Project: default

② Source

Repository URL: (방금 등록한 Repo)

Revision: HEAD

Path: .

③ Destination

Cluster: https://kubernetes.default.svc

Namespace: default (없으면 자동 생성되도록 설정 가능)

④ Sync Policy

자동화 원하면:

✔ Auto-Sync

✔ Self Heal

✔ Prune resources

운영 환경에서는 Prune 사용 시 주의 필요.

“Create” 클릭 → Application 생성됨.

✔ Step 4) Sync (배포 실행)

Application 상세 화면 → 상단의 SYNC 버튼 클릭

ArgoCD가 다음을 수행함:

Git에서 매니페스트 Pull (Token 필요)

K8s에 Apply

Deployment / Service / Ingress 생성

Healthy 체크

UI에서 각 리소스의 상태가 시각적으로 표시됨.

✔ Step 5) 배포 상태 확인

UI에서 “Pod / Service / Deployment / Ingress” 리소스를 클릭하면
상태와 이벤트, 로그까지 GUI로 확인 가능.

kubectl CLI를 사용할 수도 있음:

kubectl get pods -o wide
kubectl get svc -o wide
kubectl describe deploy cc-nginx

🔧 운영 Troubleshooting
🔹 앱이 Sync 안됨

Repo URL / Branch / Path 확인

Token 권한 오류 확인

Org Repo 승인 여부 확인

🔹 Ingress 동작 안함
kubectl get pods -n ingress-nginx
kubectl describe ingress cc-ingress

🔹 HPA 미동작
kubectl top pods
kubectl describe hpa cc-nginx

🔹 ConfigMap 변경 반영 안됨
kubectl rollout restart deploy/cc-nginx

🧭 Recommended GitOps Flow

개발자가 브랜치 생성

YAML 수정 후 PR

리뷰 후 메인 브랜치 Merge

ArgoCD 자동 Sync

UI에서 상태 모니터링

문제 시 Rollback