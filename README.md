# myapp — Helm Chart 실습

간단한 WAS를 Helm 차트로 패키징한 실습 레포지토리.

`helm install` 한 번으로 Pod 기동 · 로드밸런싱 대상 등록 · 인그레스 경로 개방까지 수행된다.

환경은 minikube 이다.

## 차트 구조

```
<asbg-01>/
├── app.py
├── Dockerfile
└── charts/myapp/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-dev.yaml
    └── templates/
        ├── deployment.yaml
        ├── service.yaml
        └── ingress.yaml
```

| 파일 | 리소스 |
| --- | --- |
| `deployment.yaml` | Deployment — Pod 개수·이미지·`/healthz` |
| `service.yaml` | Service (ClusterIP) — selector로 로드밸런싱 대상 관리 |
| `ingress.yaml` | Ingress — `ingressClassName: nginx`, 호스트/경로 라우팅 |

리소스 이름은 모두 `{{ .Release.Name }}`으로 통일.

### values

| 키 | 설명 | 기본값 |
| --- | --- | --- |
| `replicaCount` | Pod 개수 | `2` (dev: `3`) |
| `image.repository` / `image.tag` | 이미지 | `myapp` / `v1` |
| `image.pullPolicy` | 로컬 이미지 사용 | `IfNotPresent` |
| `service.port` | 컨테이너 포트 | `8080` |
| `ingress.host` / `ingress.path` | 외부 경로 | `myapp.local` / `/` |

환경 의존적인 값은 템플릿에 하드코딩하지 않고 values로 주입.

## 설치 방법

### 0. 클러스터 · 인프라 준비

```bash
minikube start --cpus=2 --memory=4096
minikube addons enable ingress
kubectl -n ingress-nginx get pods -w
kubectl get ingressclass          # nginx (default) 확인
```

### 1. 이미지 빌드

```bash
minikube image build -t myapp:v1 .
minikube image ls | grep myapp
```

또는

```bash
docker build -t myapp:v1 .
minikube image load myapp:v1
```

### 2. 설치

```bash
helm install myapp ./charts/myapp -f values-dev.yaml
```

### 3. 확인

```bash
kubectl get pods
kubectl get endpointslices
curl -H "Host: myapp.local" http://$(minikube ip)/
```

### 업그레이드 / 롤백 / 삭제

```bash
helm upgrade myapp ./charts/myapp --set replicaCount=5 -f values-dev.yaml
helm history myapp
helm rollback myapp 1
helm uninstall myapp
```

## 무엇을 하나의 차트로 묶었는가

**Deployment + Service + Ingress** — 요청이 앱에 도달하기 위한 최소 단위 전부.

기준은 생명주기를 함께하는가.

세 리소스는 이 앱과 함께 생기고 함께 사라지며, 하나라도 빠지면 목표가 성립하지 않는다.
Deployment만 있으면 접근 지점이 없고, Service까지면 클러스터 내부에서만 닿는다.

**제외한 것**

| 제외 | 이유 |
| --- | --- |
| NGINX Ingress Controller | 여러 앱이 공유하는 인프라 계층 |
| IngressClass | 클러스터 단위 리소스 |
| 컨테이너 이미지 | Helm 릴리스의 관리 대상이 아님 |
| Namespace | 앱이 소유하는 게 아니라 앱이 배치되는 환경 |

차트는 `ingressClassName: nginx`로 기존 컨트롤러를 참조만 하고 설치하지 않는다.
그래서 `helm uninstall` 시 Deployment·Service·Ingress만 삭제되고, 컨트롤러·IngressClass·이미지는 남는다.

## 선언한 것과 수렴된 것

| 리소스 | 선언 | 수렴 대상 |
| --- | --- | --- |
| Deployment | Pod N개 | 실행 중인 Pod 집합 |
| Service | `app=myapp` selector | EndpointSlice(대상 IP 목록) |
| Ingress | `myapp.local` → Service | 컨트롤러의 nginx 설정 |

`helm upgrade`는 Deployment의 `spec.replicas`만 갱신하고, 부족한 Pod는 컨트롤러가 생성한다(reconciliation).
계층은 다르지만 선언 형식과 운영 모델은 동일하다.

## 트러블슈팅

| 증상 | 해결 |
| --- | --- |
| `minikube docker-env` → `MK_USAGE` | containerd 런타임. `minikube image build/load` 사용 |
| `ImagePullBackOff` | `imagePullPolicy: IfNotPresent` 확인 |

