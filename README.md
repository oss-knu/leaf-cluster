# Leaf Cluster (k3s + Teleport Kube Agent)

이 저장소는 **망분리 Kubernetes 환경의 내부망 클러스터(Leaf)** 구성을 제공합니다. 
Leaf는 **Public IP 없이 사설망**에서 동작하며, **Root Cluster의 Proxy**를 통해서만 접근됩니다.  
Kubernetes API는 외부에 직접 노출되지 않고, **mTLS Reverse Tunnel**을 통해 Root와만 통신합니다.

---

## 개요
- 내부망 **k3s** 단일 노드(필요 시 확장 가능)
- **Teleport Kube Agent(Helm 차트)** 로 Root와 mTLS 역방향 터널 연결
- Zone 기반 네임스페이스 분리(`app/db/mgmt/dmz`)
- **Kubernetes RBAC (Role/RoleBinding)** 으로 최소 권한 제어
- 감사/모니터링은 Root의 **Audit → Logstash → OpenSearch**로 중앙 집계

---

## 전체 아키텍처에서의 Leaf Cluster 역할

Leaf는 **업무 워크로드가 실제로 실행되는 내부망 영역**이며, 다음을 보장합니다.

1. **비공개 K8s API**  
   - 외부 인바운드 트래픽을 차단하고, Root를 통해서만 접근 허용  
   - mTLS 터널의 종단점으로서 **Teleport Kube Agent**가 동작

2. **권한 최소화**  
   - Root에서 발급된 `kubernetes_groups`에 한해서만 RoleBinding으로 권한 부여  
   - 네임스페이스(Zone) 단위로 접근 범위 제한

3. **운영 단순화**  
   - Leaf 자체에는 에이전트와 RBAC만 유지  
   - 인증·인가·감사·시각화는 **모두 Root**에서 중앙 관리

> **즉, Leaf = “워크로드를 실행하지만, 접근/권한/감사는 Root를 통해서만 일어나는 내부망 실행 영역”**입니다.

---

## 아키텍처 (Leaf 관점)

```
[User] → Root Proxy/Auth ↔ (mTLS) ↔ Leaf: Teleport Kube Agent → K8s API
```

- **Teleport Kube Agent**: Root와 mTLS 연결, K8s API 프록시 역할  
- **Namespaces (Zones)**: `app`, `db`, `mgmt`, `dmz`

---

## 요구사항
- **OS**: (예시) Ubuntu 22.04/24.04 LTS on 홈 서버(내부망)  
- **Kubernetes**: k3s v1.32.x (싱글 노드; 멀티 노드로 확장 가능)  
- **Helm**: v3.18.x  
- **Teleport Kube Agent**: OSS v17.5.6 (공식 차트)  
- **네트워크**: 외부 인바운드 차단, **Root로의 아웃바운드 HTTPS만 허용**

---

## 주요 설정 파일
- `teleport/chart/values.yaml`  
- `role/*.yaml`  
  - 네임스페이스별 **Role** 정의 (예: `db-exec-role`)  
- `role-binding/*.yaml`  
  - Root의 `kubernetes_groups` 를 **RoleBinding**으로 연결

---

## 설치 및 실행

### 0) 선행: k3s 설치 (예시)
```bash
curl -sfL https://get.k3s.io | sh -
# 확인
kubectl get nodes
```

### 1) Zone 네임스페이스 생성
```bash
kubectl create ns app || true
kubectl create ns db || true
kubectl create ns mgmt || true
kubectl create ns dmz || true
```

### 2) RBAC 적용 (Role → RoleBinding)

#### Role 예시
```bash
kubectl apply -f k8s-role/app-role.yaml
kubectl apply -f k8s-role/db-role.yaml
kubectl apply -f k8s-role/db-exec-role.yaml
kubectl apply -f k8s-role/dmz-role.yaml
kubectl apply -f k8s-role/mgmt-role.yaml
```

#### RoleBinding 예시
```bash
kubectl apply -f role-binding/rolebinding-app.yaml
kubectl apply -f role-binding/rolebinding-db.yaml
kubectl apply -f role-binding/roleBinding-db-exec.yaml
kubectl apply -f role-binding/rolebinding-dmz.yaml
kubectl apply -f role-binding/rolebinding-mgmt.yaml
```

> **중요**: `subjects[].name` 값은 Root의 **Teleport Role**에서 내려주는  
> `kubernetes_groups` 와 반드시 **일치**해야 합니다. (예: `app-group`, `db-group` 등)

### 3) Teleport Kube Agent 설치 (Helm)
```bash
helm repo add teleport https://charts.releases.teleport.dev
helm repo update

helm upgrade --install teleport-agent teleport/teleport-kube-agent   -n mgmt   -f teleport/chart/values.yaml
```

**values.yaml 핵심 키(예시)**  
- `authServer`: Root Proxy 주소 (예: `root.example.com:443`)  
- `joinParams.token`: Root에서 발급한 **join token**  
- `kubeClusterName`: Leaf 클러스터 식별자 (`tsh kube ls`에 표시)  
- `resources/affinity/tolerations`: 환경에 맞게 조정

### 4) 연결 확인 (Root에서)
```bash
# Root bastion에서
tsh login --proxy=https://<ROOT_DNS_OR_IP>:443
tsh kube ls     # Leaf 클러스터가 보여야 함
tsh kube login <leaf-cluster-name>
kubectl get ns  # Leaf의 네임스페이스 목록 확인
```

---

## RBAC 예시

### Role (`role/db-exec-role.yaml`)
```yaml
# db-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: db-exec
  namespace: db
rules:
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create", "get"]
```

### RoleBinding (`role-binding/rolebinding-db-exec.yaml`)
```yaml
# rolebinding-db.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: bind-db-exec-group
  namespace: db
subjects:
- kind: Group
  name: db-exec-group           # ← teleport role에서 내려오는 kubernetes_groups
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: db-exec
  apiGroup: rbac.authorization.k8s.io
```


