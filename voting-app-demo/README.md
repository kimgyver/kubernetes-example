# Minikube로 Voting App 데모 실행하기

이 문서는 현재 폴더의 Kubernetes Voting App을 Minikube로 실행하는 절차를 정리합니다.

## 폴더 구조

- `pods/`: 단일 Pod 기반 매니페스트
- `deployments/`: Deployment 기반 매니페스트

## 사전 준비

- Docker Desktop 실행
- Minikube 설치
- kubectl 설치

도구 확인:

```bash
minikube version
kubectl version --client
```

클러스터 시작(미실행 시):

```bash
minikube start
kubectl get nodes
```

`minikube` 노드가 `Ready` 상태여야 합니다.

현재 컨텍스트 확인(권장):

```bash
kubectl config current-context
```

## 1) Pod 매니페스트로 실행 (`pods/`)

```bash
cd pods
kubectl apply -f .
kubectl get all
```

브라우저에서 서비스 열기:

```bash
minikube service vote
minikube service result
```

## 2) Deployment 매니페스트로 실행 (`deployments/`)

### 기존 리소스 정리(권장)

이전에 `pods/`로 실행했다면 전환 전에 정리하세요:

```bash
kubectl delete -f ../pods --ignore-not-found
kubectl delete -f . --ignore-not-found
```

### Deployment 적용

```bash
cd ../deployments
kubectl apply -f .
kubectl get all
```

정상 서비스 목록:

- `vote` (NodePort)
- `result` (NodePort)
- `redis` (ClusterIP)
- `db` (ClusterIP)

## 중요 체크: deployments에 db 서비스 필요

`db` 서비스가 없으면 `worker`가 DNS 이름 `db`로 Postgres에 연결하지 못해, 결과 화면(`result`)이 투표 집계를 반영하지 못합니다.

필수 파일:

- `deployments/db-service.yaml`

필요 시 수동 적용:

```bash
kubectl apply -f db-service.yaml
kubectl get svc
```

## vote/result 접속

```bash
minikube service vote
minikube service result
```

macOS + Docker 드라이버 환경에서는 Node IP 직접 접근이 실패할 수 있습니다. 이 경우 `minikube service`가 출력하는 `127.0.0.1:<port>` URL을 사용하세요.

## 상태 검증 명령(보강)

배포 후 아래 명령으로 빠르게 상태를 확인하세요:

```bash
kubectl get pods
kubectl get svc
kubectl get deploy
kubectl get endpoints
```

`db`, `redis`, `vote`, `result` 서비스에 엔드포인트가 잡혀 있어야 정상입니다.

## 스케일 테스트

vote 앱 복제 수 증가:

```bash
kubectl scale deployment vote --replicas=5
kubectl get pods
```

## 문제 해결

### 1) 명령어 오타

`kubectl`이 맞습니다. (`kubetcl` 오타 주의)

```bash
kubectl get all
```

### 2) result 화면에 투표가 반영되지 않을 때

아래 순서로 확인:

```bash
kubectl get svc
kubectl get pods
kubectl logs deployment/worker --tail=100
kubectl logs deployment/result --tail=100
```

확인 포인트:

- `service/db` 존재 여부
- `worker` 파드 `Running` 여부
- `db` 또는 `redis` 연결 오류 메시지 유무

### 3) 터널이 끊기는 경우

macOS Docker 드라이버에서는 `minikube service ...`를 실행한 터미널을 계속 열어둬야 합니다.

### 4) 리소스가 안 보일 때

네임스페이스 확인:

```bash
kubectl get all -n default
kubectl config view --minify --output 'jsonpath={..namespace}'
```

## 빠른 초기화(리셋)

현재 디렉토리 기준 전체 재적용:

```bash
kubectl delete -f . --ignore-not-found
kubectl apply -f .
kubectl get all
```

원하는 방식(`pods/` 또는 `deployments/`) 디렉토리에서 실행하면 됩니다.
