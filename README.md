# kubernetes-jason

쿠버네티스 핵심 개념을 이 저장소 YAML과 함께 익히기 위한 학습용 레포입니다.

이 문서만 읽어도 아래를 이해할 수 있도록 구성했습니다.

- 쿠버네티스가 왜 필요한지
- API Server, Controller, Scheduler, 워커 노드가 각각 무엇을 하는지
- Pod, ReplicaSet, Deployment, Service의 역할
- 한 종류의 서버에 네 객체를 전부 직접 만들어야 하는지
- 그 개념이 이 레포 파일과 어떻게 연결되는지

단일 nginx에서 시작해 ReplicaSet, Deployment, Service를 거친 뒤, Docker Example Voting App을 Minikube에 배포합니다.

## 쿠버네티스가 하는 일

쿠버네티스(Kubernetes, k8s)는 컨테이너를 여러 서버에 올리고, 죽으면 다시 살리고, 트래픽을 나눠 주는 오케스트레이션 시스템입니다.

혼자 Docker로 `docker run nginx`를 하면 그 컨테이너가 죽으면 끝입니다. 쿠버네티스는 원하는 상태(desired state)를 YAML로 선언하면, 클러스터가 그 상태를 계속 맞춰 줍니다.

핵심 계층:

| 계층 | 역할 |
|------|------|
| **클러스터** | 컨트롤 플레인(마스터) + 워커 노드 묶음 |
| **노드** | 실제 컨테이너가 돌아가는 서버 |
| **Pod** | 쿠버네티스가 다루는 최소 배포 단위. 컨테이너 1개 이상 |
| **ReplicaSet** | 같은 Pod를 N개 유지 |
| **Deployment** | ReplicaSet을 관리하고 롤링 업데이트까지 담당 |
| **Service** | Pod IP가 바뀌어도 고정된 이름으로 접근하게 함 |

이 저장소 폴더 구조가 바로 그 학습 순서입니다.

```text
kubernetes-jason/
├── pods/                 # 1. 단일 Pod
├── replicasets/          # 2. 복제본 유지
├── deployments/          # 3. 배포와 스케일
├── services/             # 4. 네트워크 진입점
└── voting-app-demo/      # 5. 실제 멀티서비스 앱
    ├── pods/
    └── deployments/
```

## 이 레포와 클러스터의 관계

이 저장소에는 API Server, Controller, Scheduler, 워커 노드 **구현이 없습니다**. 있는 것은 그 컴포넌트들에게 보내는 YAML뿐입니다.

비유하면, **이 레포는 편지(YAML)이고, API Server / Controller / Scheduler / 워커 노드는 그 편지를 처리하는 우체국(클러스터)** 입니다.

```text
YAML (이 레포)
  → kubectl apply
  → API Server
  → Controllers
  → Scheduler
  → 워커 노드 kubelet
  → nginx / vote / redis 컨테이너 실행
```

| 다이어그램 박스 | 이 레포에 있나 | 실제로 어디에 있나 |
|----------------|----------------|-------------------|
| API Server | 없음 | `kube-apiserver` |
| Controllers | 없음 | `kube-controller-manager` |
| Scheduler | 없음 | `kube-scheduler` |
| 워커 노드 | 없음 | `kubelet` + `kube-proxy` + 컨테이너 런타임 |
| 앱 선언 | 있음 | `pods/`, `deployments/`, `voting-app-demo/` YAML |

`minikube start`를 하면 위 네 가지가 로컬 클러스터로 뜹니다.

```bash
kubectl get pods -n kube-system
```

대략 이런 것들이 보입니다.

- `kube-apiserver-minikube`
- `kube-controller-manager-minikube`
- `kube-scheduler-minikube`
- `kube-proxy-...`

kubelet은 Pod가 아니라 노드 프로세스입니다.

### YAML이 거치는 경로

예를 들어 `deployments/deployment.yaml`을 적용하면:

```bash
kubectl apply -f deployments/deployment.yaml
```

1. **kubectl**이 YAML을 API Server에 보냅니다.
2. **API Server**가 `Deployment` 객체를 etcd에 저장합니다.
3. **Deployment Controller**가 보고 ReplicaSet을 만듭니다.  
   `replicasets/replicaset.yaml`을 직접 적용하면 이 단계를 건너뛰고 ReplicaSet만 만듭니다.
4. **ReplicaSet Controller**가 `template`을 보고 Pod를 3개 만듭니다.  
   `pods/nginx.yaml`을 직접 적용하면 Controller 없이 Pod만 생깁니다.
5. **Scheduler**가 아직 노드가 없는 Pod를 워커 노드에 붙입니다.
6. **워커 노드의 kubelet**이 그 Pod spec을 읽고 `image:`의 컨테이너를 실행합니다.

워커 노드에서 실행되는 결과물은 YAML의 `image:` 필드입니다.

- 기초 예제: `image: nginx`
- 투표 앱: `dockersamples/examplevotingapp_vote`, `redis:alpine`, `postgres:15-alpine` 등

## 사전 준비

- Docker Desktop
- Minikube
- kubectl

```bash
minikube version
kubectl version --client
minikube start
kubectl get nodes
kubectl config current-context
```

노드가 `Ready`여야 합니다.

## Pod — 실행 단위

쿠버네티스가 스케줄하고 실행하는 가장 작은 단위입니다. 컨테이너 1개 이상을 묶고, 그 묶음이 하나의 IP와 볼륨을 공유합니다.

하는 일:

- `image:`에 적힌 컨테이너를 노드에서 실행
- 앱 프로세스의 실제 집
- 죽으면 그 Pod는 끝. 스스로 다시 안 만듦

예: `pods/nginx.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-2
  labels:
    app: myapp
spec:
  containers:
    - name: nginx
      image: nginx
```

의미:

- `kind: Pod` — 컨테이너 묶음 하나 만들어 달라
- `metadata.name` — 객체 이름
- `labels` — 나중에 Service / ReplicaSet이 Pod를 찾을 때 쓰는 태그
- `image: nginx` — Docker Hub의 nginx 이미지를 실행

Pod IP는 재생성마다 바뀝니다. 그래서 프로덕션에서는 Pod를 직접 오래 두지 않고, 보통 Deployment가 Pod를 만듭니다.

```bash
kubectl apply -f pods/nginx.yaml
kubectl get pods
```

## ReplicaSet — 개수 유지

“`app: myapp` Pod를 항상 3개 둬라”를 지키는 컨트롤러입니다.

하는 일:

- `selector`로 대상 Pod를 찾음
- `replicas`보다 적으면 `template`으로 새로 만듦
- 많으면 삭제
- Pod가 크래시해도 다시 채움

예: `replicasets/replicaset.yaml`

```yaml
kind: ReplicaSet
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
```

여기서 중요한 개념이 **셀렉터**입니다. ReplicaSet은 `app: myapp` 라벨이 붙은 Pod를 찾습니다. 하나를 수동으로 지워도 곧 다시 생깁니다.

```text
ReplicaSet (replicas=3)
  ├── Pod app=myapp
  ├── Pod app=myapp
  └── Pod app=myapp
```

ReplicaSet만 있으면 복구와 스케일은 됩니다. 다만 이미지 버전을 바꾸는 롤링 업데이트, 롤백은 Deployment 역할입니다. 실무에서는 ReplicaSet YAML을 직접 잘 안 만들고, Deployment가 내부적으로 만듭니다.

## Deployment — 배포 관리

ReplicaSet 위의 운영 레이어입니다. 사용자가 보통 만지는 객체입니다.

하는 일:

- 내부적으로 ReplicaSet을 생성/교체
- 복제 수 변경 (`kubectl scale`)
- 이미지 교체 시 새 ReplicaSet을 만들고 점진적으로 전환 (롤링 업데이트)
- 문제 생기면 이전 ReplicaSet으로 롤백

예: `deployments/deployment.yaml`

```text
Deployment
  └── ReplicaSet
        ├── Pod
        ├── Pod
        └── Pod
```

관계 정리:

- Deployment가 없으면 ReplicaSet을 직접 관리해야 함
- ReplicaSet이 없으면 Pod 개수가 보장되지 않음
- Pod가 없으면 컨테이너가 안 돌아감

Voting App에서는 `kubectl scale deployment vote --replicas=5`로 이 개념을 실제로 써 봅니다. `pods/`로 올린 단일 Pod는 이렇게 늘릴 수 없습니다.

## Service — 안정적인 네트워크 입구

Pod IP는 재시작마다 바뀝니다. Service는 라벨로 Pod를 골라 고정된 가상 IP / DNS 이름을 줍니다.

하는 일:

- `selector`와 맞는 Pod를 엔드포인트로 묶음
- 클러스터 안에서 `vote`, `redis`, `db` 같은 이름으로 접속
- 여러 Pod가 있으면 로드밸런싱
- `NodePort`면 클러스터 밖에서도 접근

예: `services/service-definition.yaml`

```yaml
kind: Service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30004
  selector:
    app: myapp
```

포트 세 가지:

| 필드 | 의미 |
|------|------|
| `port` | 클러스터 안에서 Service로 들어오는 포트 |
| `targetPort` | 실제 Pod / 컨테이너가 듣는 포트 |
| `nodePort` | 노드 바깥에서 접근하는 포트 (30000–32767) |

Service 종류:

| 타입 | 이 저장소에서 | 용도 |
|------|----------------|------|
| **ClusterIP** (기본) | `redis`, `db` | 클러스터 내부 전용. 외부에 안 열림 |
| **NodePort** | `vote`, `result`, `myapp-service` | 노드 IP:포트로 외부 접근 |
| **LoadBalancer** | 없음 | 클라우드 LB를 붙일 때 |

클러스터 안에서는 DNS 이름 `myapp-service`로 붙습니다. Voting App에서 worker가 호스트 이름 `db`, `redis`로 붙는 이유가 바로 이것입니다. 서비스 이름이 DNS가 됩니다.

## 한 종류의 서버에 네 가지가 다 있어야 하나?

아니요. 한 종류의 서버마다 Pod, ReplicaSet, Deployment, Service YAML을 전부 만들 필요는 없습니다.

실무에서는 보통 이것만 만듭니다.

- **Deployment 1개** — 그 서버의 Pod를 만들고 개수/업데이트를 관리
- **Service 1개** — 그 서버를 다른 앱이나 밖에서 찾아야 할 때

ReplicaSet과 Pod는 Deployment가 알아서 만듭니다.

```text
직접 만드는 것
  ├── Deployment
  └── Service (필요할 때만)

자동 생성
  ├── ReplicaSet
  └── Pod × replicas
```

Voting App의 `vote`가 그 패턴입니다.

| 직접 작성 | 자동 생성 |
|-----------|-----------|
| `vote-deployment.yaml` | ReplicaSet, Pod |
| `vote-service.yaml` | — |

`kubectl apply -f vote-deployment.yaml`만 해도:

1. Deployment `vote` 생성
2. ReplicaSet `vote-xxxxx` 생성
3. Pod `vote-xxxxx-yyyyy` 생성

`kubectl get all`에 ReplicaSet / Pod가 보이는 이유입니다. YAML로 따로 안 적어도 됩니다.

언제 무엇이 필요한가:

| 만드는 것 | 언제 |
|-----------|------|
| Deployment만 | 배치 워커처럼 남이 네트워크로 안 붙는 서버. 이 레포의 `worker` |
| Deployment + Service | 웹앱, API, DB, Redis처럼 접속 대상이 있는 서버. `vote`, `result`, `redis`, `db` |
| Pod만 직접 | 디버그, 일회성 실험. 죽으면 복구 없음. 학습용 `pods/` |
| ReplicaSet을 직접 | 거의 안 씀. Deployment가 대신 관리 |

“한 종류의 서버”는 같은 앱의 복제 그룹입니다. vote를 5대로 늘려도 Deployment 1 + Service 1입니다. Pod가 5개로 늘어날 뿐입니다.

```text
vote 서버 종류
  ├── Deployment vote     (1개, 직접 작성)
  ├── ReplicaSet          (1개, 자동)
  ├── Pod × replicas      (자동)
  └── Service vote        (붙어야 하면 1개, 직접 작성)
```

redis, db, result는 각각 자기 Deployment(+ 필요하면 Service)를 가집니다. 클러스터 전체에 Pod / Deployment / ReplicaSet / Service가 하나씩만 있는 구조가 아닙니다.

넷의 분업:

1. **Pod** — 프로세스를 돌린다
2. **ReplicaSet** — 그 프로세스를 N개 유지한다
3. **Deployment** — 그 유지를 배포/업데이트 단위로 관리한다
4. **Service** — 그 Pod들을 이름 하나로 찾아가게 한다

Pod만 있으면 앱은 뜨지만 죽으면 끝입니다. ReplicaSet까지 있으면 살아남습니다. Deployment까지 있으면 버전을 바꿀 수 있습니다. Service까지 있으면 다른 앱과 브라우저가 안정적으로 붙습니다.

## 기초 예제 실행

객체 하나씩 올려 보려면:

```bash
kubectl apply -f pods/nginx.yaml
kubectl apply -f replicasets/replicaset.yaml
kubectl apply -f deployments/deployment.yaml
kubectl apply -f services/service-definition.yaml
kubectl get all
```

`deployments/deployment.yaml`만 적용해도 ReplicaSet과 Pod가 생깁니다. `myapp-service`는 `app: myapp` 라벨 Pod로 트래픽을 보냅니다.

정리:

```bash
kubectl delete -f services/service-definition.yaml --ignore-not-found
kubectl delete -f deployments/deployment.yaml --ignore-not-found
kubectl delete -f replicasets/replicaset.yaml --ignore-not-found
kubectl delete -f pods/nginx.yaml --ignore-not-found
kubectl delete -f pods/pod.yaml --ignore-not-found
```

## Voting App

[Docker Example Voting App](https://github.com/dockersamples/example-voting-app)을 쿠버네티스 YAML로 옮긴 데모입니다. 위에서 배운 네 객체가 실제로 맞물리는 예제입니다.

```text
사용자 → vote → redis → worker → postgres(db) → result → 사용자
```

| 이름 | 역할 | 이미지 | 외부 공개 |
|------|------|--------|-----------|
| `vote` | 투표 UI | `dockersamples/examplevotingapp_vote` | NodePort `31000` |
| `redis` | 투표를 잠시 담는 큐 | `redis:alpine` | ClusterIP |
| `worker` | redis에서 꺼내 DB에 저장 | `dockersamples/examplevotingapp_worker` | 없음 (백그라운드) |
| `db` | 집계 저장소 Postgres | `postgres:15-alpine` | ClusterIP |
| `result` | 실시간 결과 화면 | `dockersamples/examplevotingapp_result` | NodePort `31001` |

내부 통신이 되는 이유는 서비스 이름과 라벨이 맞춰져 있기 때문입니다.

- `vote` 앱은 호스트 `redis`로 접속 → `service/redis`가 `app: redis` Pod로 전달
- `worker`는 `redis`와 `db`로 접속
- `result`는 `db`로 접속

`db` Service가 없으면 worker가 DNS 이름 `db`를 못 찾고, result 화면에 투표가 안 나옵니다.

같은 앱을 두 가지 방식으로 올릴 수 있습니다.

- `voting-app-demo/pods/` — Pod를 하나씩 직접 생성. 학습용. 죽으면 자동 복구 없고 스케일도 어려움
- `voting-app-demo/deployments/` — 권장. 각 앱을 Deployment로 올림. vote를 5개로 늘려도 Service가 라벨 `app: vote`로 트래픽을 나눔

권장 실행:

```bash
cd voting-app-demo/deployments
kubectl apply -f .
kubectl get all
minikube service vote
minikube service result
```

macOS + Docker 드라이버에서는 노드 IP 직접 접근이 안 될 수 있습니다. `minikube service`가 알려 주는 `127.0.0.1:<port>`를 쓰고, 그 터미널은 열어 둬야 터널이 유지됩니다.

스케일 테스트:

```bash
kubectl scale deployment vote --replicas=5
kubectl get pods
```

vote Pod가 5개가 되고, `vote` Service가 그 5개로 요청을 분산합니다.

상세 절차, 점검 명령, 문제 해결은 [voting-app-demo/README.md](voting-app-demo/README.md)를 보세요.

## 이 저장소에 없는 것

입문 예제라 아래는 아직 없습니다.

- **Ingress** — 도메인/경로 기반 HTTP 라우팅
- **ConfigMap / Secret** — 설정과 비밀번호 분리. 지금은 `db` 비밀번호가 YAML에 그대로 있음
- **PersistentVolume** — DB 데이터가 Pod 재시작 후에도 남게
- **Health check** — liveness / readiness probe
- **Namespace, HPA, Helm**

이 레포의 위치는 “쿠버네티스 객체 4종(Pod, ReplicaSet, Deployment, Service)을 YAML로 읽고, 멀티서비스 앱이 라벨·DNS로 연결되는 모습을 보는 입문 코스”입니다.

한 줄 정리:

**Pod는 실행 단위, ReplicaSet은 개수 유지, Deployment는 배포 관리, Service는 안정적인 네트워크 이름이고, Voting App은 이 네 가지가 실제로 맞물리는 예제입니다. 그 네 객체를 실행하는 엔진은 이 레포가 아니라 Minikube 클러스터의 API Server, Controller, Scheduler, kubelet입니다.**
