# Calico Network

## Overview

Kubernetes Cluster의 Pod Network를 구성하기 위해
CNI(Container Network Interface)로 Calico를 사용하였다.

Cluster의 Pod Network CIDR은 kubeadm 초기화 단계에서 다음과 같이 설정하였다.

```yaml
networking:
  podSubnet: 10.244.0.0/16
```

Calico는 각 Kubernetes Node에 Network Component를 배포하여
서로 다른 Node에 위치한 Pod 간 통신이 가능하도록 구성한다.

```text
worker01                         worker02
┌─────────────────┐            ┌─────────────────┐
│                 │            │                 │
│ Pod             │            │ Pod             │
│ 10.244.x.x      │            │ 10.244.x.x      │
│       │         │            │       │         │
│       ▼         │            │       ▼         │
│ calico-node     │◀──────────▶│ calico-node     │
│                 │            │                 │
└─────────────────┘            └─────────────────┘
```

---

## 1. Calico Installation

Control Plane 초기화 후 CNI가 설치되지 않은 상태에서는
Node가 `NotReady` 상태로 표시될 수 있다.

```bash
kubectl get nodes
```

Calico 설치 Manifest를 적용하여 Pod Network를 구성하였다.

```bash
kubectl apply -f <CALICO_INSTALL_MANIFEST>
```

본 프로젝트에서는 Tigera Operator 기반으로 Calico를 구성하였다.

Calico 관련 Resource 확인:

```bash
kubectl get pods -n calico-system
```

```bash
kubectl get pods -n tigera-operator
```

---

## 2. Calico Components

Calico 설치 후 Cluster에는 Pod Networking을 담당하는
여러 Component가 구성된다.

주요 Component는 다음과 같다.

| Component | Role |
|---|---|
| calico-node | 각 Node에서 Pod Network 구성 |
| calico-kube-controllers | Calico Network Resource 관리 |
| calico-typha | Calico Node와 Kubernetes Datastore 사이 연결 최적화 |
| csi-node-driver | Calico 관련 CSI 기능 제공 |

`calico-node`는 DaemonSet으로 배포되므로
각 Kubernetes Node마다 하나씩 실행된다.

확인:

```bash
kubectl get daemonset -n calico-system
```

```bash
kubectl get pods -n calico-system -o wide
```

---

## 3. Node Network Status

Calico가 정상적으로 구성되면 Kubernetes Node가
`Ready` 상태로 변경된다.

```bash
kubectl get nodes
```

최종적으로 3개의 Control Plane과 3개의 Worker Node가
모두 `Ready` 상태인 것을 확인하였다.

```text
NAME       STATUS   ROLES
cp01       Ready    control-plane
cp02       Ready    control-plane
cp03       Ready    control-plane
worker01   Ready    <none>
worker02   Ready    <none>
worker03   Ready    <none>
```

---

## 4. Pod Network

Pod는 Node Network와 별도의 IP 대역을 사용하도록 구성하였다.

```text
Node Network       192.168.10.0/24
Pod Network        10.244.0.0/16
Service Network    10.96.0.0/12
```

Pod가 어느 Node에 배치되었는지와 할당된 IP를 다음 명령으로 확인하였다.

```bash
kubectl get pods -A -o wide
```

Pod IP는 `10.244.0.0/16` 범위에서 할당된다.

---

## 5. Pod-to-Pod Communication

Calico Network가 정상적으로 동작하는지 확인하기 위해
서로 다른 Node에 배치된 Pod 간 Network 통신을 확인하였다.

```text
Pod A
10.244.x.x
   │
   │ Calico Network
   ▼
Pod B
10.244.x.x
```

Pod 위치 확인:

```bash
kubectl get pods -o wide
```

필요한 경우 Pod 내부에서 다른 Pod IP로 직접 통신하여
Network 연결 상태를 확인할 수 있다.

```bash
kubectl exec -it <POD_NAME> -- ping <POD_IP>
```

---

## 6. Service Network Test

Pod Network뿐 아니라 Kubernetes Service를 통한
통신도 함께 검증하였다.

테스트용 nginx Deployment 생성:

```bash
kubectl create deployment nginx-test \
  --image=nginx:latest \
  --replicas=3
```

ClusterIP Service 생성:

```bash
kubectl expose deployment nginx-test \
  --name=nginx-test-svc \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

Service 확인:

```bash
kubectl get svc
```

Cluster 내부에서 Service DNS를 이용하여 접근하였다.

```bash
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  --rm -it \
  -- curl http://nginx-test-svc
```

nginx 응답이 정상적으로 반환되는 것을 확인하였다.

이를 통해 다음 경로가 정상적으로 동작하는 것을 검증하였다.

```text
Test Pod
   │
   ▼
CoreDNS
   │
   ▼
ClusterIP Service
   │
   ▼
Backend Pod
```

---

## 7. Troubleshooting Experience

Cluster 구축 과정에서 일부 Node의 `calico-node`가
정상적인 Ready 상태로 변경되지 않는 문제가 발생하였다.

```text
calico-node    0/1    Running
```

Node에는 다음 Taint가 적용된 상태였다.

```text
node.kubernetes.io/network-unavailable
```

`calico-node`의 Startup Probe를 확인한 결과
BIRD가 정상적으로 준비되지 않은 상태를 확인하였다.

```text
BIRD is not ready
unable to connect to BIRDv4 socket
```

문제 범위를 확인하기 위해 Node의 Host Firewall 정책을 점검하였으며,
Lab 환경에서 `firewalld`를 일시적으로 중지한 후
Calico가 정상 상태로 변경되는 것을 확인하였다.

```bash
ansible k8s -i inventory.ini -m service \
  -a "name=firewalld state=stopped"
```

이후:

```text
calico-node    1/1    Running
```

상태로 변경되고 모든 Kubernetes Node가 `Ready` 상태가 되었다.

이 테스트를 통해 Host Firewall 정책이 Calico Node 간 통신에
영향을 주고 있음을 확인하였다.

정확한 Calico Encapsulation/BGP 설정과 필요한 Network Traffic을
기준으로 Firewall 정책을 구성하는 작업은 추가 Hardening 항목으로 남겼다.

상세한 장애 분석 과정은 Troubleshooting 문서에서 별도로 정리한다.

→ [Troubleshooting](troubleshooting.md)

---

## 8. Verification

Calico Component:

```bash
kubectl get pods -n calico-system -o wide
```

Node 상태:

```bash
kubectl get nodes
```

Pod IP:

```bash
kubectl get pods -A -o wide
```

Service:

```bash
kubectl get svc
```

최종적으로 다음 항목을 확인하였다.

```text
Calico Node       Running
Kubernetes Node   Ready
Pod IP Allocation 정상
Pod Network       정상
Service Network   정상
CoreDNS           정상
```

---

## Next Step

Kubernetes Control Plane HA와 Calico Pod Network 구성이 완료된 후
실제 Deployment와 Service를 생성하여 Cluster 동작을 검증하였다.

→ [Cluster Validation](06-cluster-validation.md)
