# k8s-deploy-web

PetClinic의 **WEB(React SPA + Nginx)** Helm 차트와 외부 입구 **Envoy Gateway** 매니페스트입니다.
WAS · DB는 [k8s-deploy-was](https://github.com/kdt-6-aws-3/k8s-deploy-was)에 있습니다.

Nginx가 화면(정적 파일)을 주고, `/api/` 요청은 클러스터 안 `was` Service로 넘깁니다.

## 요청 흐름

| | kubeadm (3노드) | EKS (Auto Mode) |
|---|---|---|
| 입구 | `worker1 공인 IP:30090` (NodePort) | NLB (internet-facing, target-type ip) |
| Envoy | 1개, worker1 고정 | 2개, AZ(2a · 2c)에 하나씩 |
| 흐름 | 브라우저 → worker1:30090 → Envoy → web → was | 브라우저 → NLB → Envoy → web → was |
| 매니페스트 | `gateway/gateway.yaml` | `gateway/gateway-eks.yaml` |
| values | `values.yaml` | `values.yaml` + `values-eks.yaml` |

web Service는 두 환경 모두 `ClusterIP`입니다. 외부 요청은 Envoy를 거쳐서만 들어옵니다.

## 구성

```
.
├── container-project-web/      # Nginx Deployment · Service · PDB · SA
│   ├── values.yaml             # 기본값 (kubeadm, GHCR 이미지)
│   └── values-eks.yaml         # EKS 덮어쓰기 (ECR 이미지, AZ 분산)
└── gateway/
    ├── gateway.yaml            # kubeadm: NodePort 30090, worker1
    └── gateway-eks.yaml        # EKS: NLB
```

두 gateway 파일 모두 EnvoyProxy `petclinic-proxy` · GatewayClass `eg` · Gateway `petclinic-gw`(HTTP 80) · HTTPRoute `web`(`/` → web:8080)으로 이루어져 있고, Envoy를 띄우는 방식만 다릅니다.

## 사전 준비

- namespace `petclinic`과 WAS가 먼저 설치되어 있어야 합니다 (k8s-deploy-was 참고).
- Envoy Gateway 컨트롤러 설치:

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm --version v1.10.0 \
  -n envoy-gateway-system --create-namespace
```

## kubeadm 배포

```bash
helm upgrade --install web ./container-project-web -n petclinic
kubectl apply -f gateway/gateway.yaml
```

접속: `http://<worker1 공인 IP>:30090` (보안 그룹에서 30090 허용 필요)

## EKS 배포

```bash
helm upgrade --install web ./container-project-web -n petclinic -f container-project-web/values-eks.yaml
kubectl apply -f gateway/gateway-eks.yaml
```

Auto Mode 내장 컨트롤러(`loadBalancerClass: eks.amazonaws.com/nlb`)가 NLB를 만듭니다. 별도 AWS Load Balancer Controller 설치나 대상 그룹 · ALB 보안 그룹 생성은 필요 없습니다.
NLB는 public 서브넷(`kubernetes.io/role/elb=1`)에 생기고 태그 `Project=k8s-personal`이 붙습니다.

### 확인

```bash
# NLB 주소
kubectl -n petclinic get gateway petclinic-gw

H=<NLB 주소>
curl -s -o /dev/null -w "%{http_code}\n" http://$H/      # 200
curl -s "http://$H/api/owners?size=1"                     # WAS · RDS 응답
```

NLB가 처음 만들어지면 DNS가 퍼지고 대상이 healthy가 될 때까지 몇 분 걸립니다.

## 로그 (EKS: EFS)

이미지의 `/var/log/nginx/access.log` · `error.log`는 stdout · stderr 심볼릭 링크입니다.
EKS(`values-eks.yaml`, `logs.enabled: true`)에서는 그 자리에 EFS PVC `petclinic-shared`의 `web/<Pod 이름>/`를 덮어 마운트해서 nginx가 실제 파일로 씁니다.
PVC는 k8s-deploy-was의 `eks/storage-efs.yaml`로 먼저 만듭니다.

- kubeadm(`values.yaml`)은 `logs.enabled: false`라 지금처럼 stdout으로 나갑니다
- 컨테이너 안에 logrotate가 없어 파일이 계속 커집니다. 오래 운영하면 정리 방법이 필요합니다

## 공통 보안 설정

- `nginx-unprivileged` 이미지(UID 101)로 non-root 실행, 권한 상승 금지, capability 전부 제거 (PSA `restricted` 통과)
- 전용 ServiceAccount, Kubernetes API 토큰 마운트 안 함
- PDB `minAvailable: 1`, `preStop` 10초 대기

## 정리 (비용)

`kubectl delete -f gateway/gateway-eks.yaml`로 Gateway를 지우면 NLB도 함께 지워집니다.
