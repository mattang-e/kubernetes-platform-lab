# Kubernetes Cluster Bootstrap

## Overview

Node 기본 구성을 완료한 후 `kubeadm`을 사용하여 Kubernetes Cluster를 구축하였다.

첫 번째 Control Plane인 `cp01`에서 Cluster를 초기화하고,
HAProxy와 Keepalived로 구성한 Virtual IP를 Kubernetes API Endpoint로 사용하였다.

이후 `cp02`, `cp03`을 추가 Control Plane으로 Join하고
3개의 Worker Node를 Cluster에 추가하였다.

최종 구성은 다음과 같다.

```text
                  192.168.10.115:6443
                    API Virtual IP
                          │
                    HAProxy / VIP
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
           cp01         cp02         cp03
       Control Plane Control Plane Control Plane
           etcd         etcd         etcd
             │            │            │
             └────────────┼────────────┘
                          │
             ┌────────────┼────────────┐
             ▼            ▼            ▼
         worker01      worker02      worker03
```

---

## 1. kubeadm Configuration 생성

`cp01`에서 kubeadm 기본 설정 파일을 생성하였다.

```bash
kubeadm config print init-defaults > kubeadm-config.yaml
```

생성된 기본 설정을 현재 Cluster 환경에 맞게 수정하였다.

```bash
vi kubeadm-config.yaml
```

주요 설정은 다음과 같다.

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration

localAPIEndpoint:
  advertiseAddress: 192.168.10.101
  bindPort: 6443

nodeRegistration:
  criSocket: unix:///var/run/containerd/containerd.sock
  imagePullPolicy: IfNotPresent
  name: node

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration

kubernetesVersion: 1.34.0

controlPlaneEndpoint: "192.168.10.115:6443"

etcd:
  local:
    dataDir: /var/lib/etcd

networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
```

실제 설정 파일에는 kubeadm에서 생성한 timeout, certificate,
bootstrap token 등의 추가 설정도 포함되어 있다.

---

## 2. 주요 kubeadm 설정

### localAPIEndpoint

```yaml
localAPIEndpoint:
  advertiseAddress: 192.168.10.101
  bindPort: 6443
```

첫 번째 Control Plane인 `cp01`의 kube-apiserver가 사용할
Local API Endpoint를 지정하였다.

```text
cp01
192.168.10.101:6443
```

### controlPlaneEndpoint

```yaml
controlPlaneEndpoint: "192.168.10.115:6443"
```

개별 Control Plane IP가 아닌 HAProxy / Keepalived에서 관리하는
Virtual IP를 Kubernetes Cluster의 공통 API Endpoint로 사용하였다.

```text
kubectl / kubelet
       │
       ▼
192.168.10.115:6443
       │
       ▼
    HAProxy
       │
 ┌─────┼─────┐
 ▼     ▼     ▼
cp01  cp02  cp03
:6443 :6443 :6443
```

이를 통해 Client 및 Worker Node가 특정 Control Plane Node의
IP에 직접 의존하지 않도록 구성하였다.

### Container Runtime

```yaml
criSocket: unix:///var/run/containerd/containerd.sock
```

Container Runtime으로 containerd를 사용하므로 kubelet이
containerd CRI Socket을 사용하도록 설정하였다.

### Kubernetes Network

```yaml
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 10.244.0.0/16
```

각 Network의 역할은 다음과 같다.

| Network | CIDR | Purpose |
|---|---|---|
| Node Network | 192.168.10.0/24 | Physical / VM Node 통신 |
| Pod Network | 10.244.0.0/16 | Pod 간 통신 |
| Service Network | 10.96.0.0/12 | Kubernetes Service ClusterIP |

각 Network 대역이 서로 중복되지 않도록 구성하였다.

---

## 3. Control Plane 초기화

작성한 `kubeadm-config.yaml`을 사용하여 첫 번째 Control Plane을
초기화하였다.

```bash
kubeadm init \
  --config kubeadm-config.yaml \
  --upload-certs
```

`--upload-certs` 옵션을 사용하여 추가 Control Plane Node가 Join할 때
필요한 Control Plane 인증서를 공유할 수 있도록 구성하였다.

초기화 과정에서 kubeadm은 다음과 같은 Kubernetes Control Plane
Component를 구성한다.

```text
kube-apiserver
kube-controller-manager
kube-scheduler
etcd
```

Control Plane Component는 Static Pod 형태로 구성된다.

확인:

```bash
ls /etc/kubernetes/manifests/
```

```text
etcd.yaml
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
```

---

## 4. kubectl Configuration

Cluster 초기화 후 `kubectl`이 생성된 Cluster를 관리할 수 있도록
admin kubeconfig를 설정하였다.

```bash
mkdir -p $HOME/.kube

cp -i /etc/kubernetes/admin.conf \
  $HOME/.kube/config

chown $(id -u):$(id -g) \
  $HOME/.kube/config
```

Cluster 상태 확인:

```bash
kubectl get nodes
```

CNI가 아직 설치되지 않은 상태에서는 Node가 `NotReady`로 표시될 수 있다.

---

## 5. CNI Installation

Pod Network 구성을 위해 Calico를 CNI로 사용하였다.

Calico 설치 후 각 Node에 `calico-node`가 배포되고
Pod Network가 구성된다.

확인:

```bash
kubectl get pods -n calico-system
```

```bash
kubectl get nodes
```

Calico가 정상적으로 구성되면 Node가 `Ready` 상태로 변경된다.

Calico의 상세 구성 및 Network Troubleshooting은 별도 문서에서 다룬다.

→ [Calico Network](05-calico-network.md)

---

## 6. Additional Control Plane Join

`cp02`, `cp03`을 추가 Control Plane으로 구성하기 위해
Join Token과 Certificate Key를 사용하였다.

Join Token 생성:

```bash
kubeadm token create --print-join-command
```

필요한 경우 Control Plane Certificate를 다시 Upload하였다.

```bash
kubeadm init phase upload-certs --upload-certs
```

추가 Control Plane은 다음과 같은 형태로 Join하였다.

```bash
kubeadm join 192.168.10.115:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY>
```

`--control-plane` 옵션을 사용하여 일반 Worker가 아닌
Control Plane Node로 추가하였다.

`cp02`, `cp03` Join 후 각 Node에서 다음 Component가 구성된다.

```text
kube-apiserver
kube-controller-manager
kube-scheduler
etcd
```

---

## 7. Worker Node Join

Worker Node에서는 Control Plane Join과 달리
`--control-plane`, `--certificate-key` 옵션을 사용하지 않는다.

```bash
kubeadm join 192.168.10.115:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

다음 Node를 Cluster에 추가하였다.

```text
worker01
worker02
worker03
```

---

## 8. Cluster Verification

모든 Node Join 후 Cluster 상태를 확인하였다.

```bash
kubectl get nodes
```

최종 구성:

```text
NAME       STATUS   ROLES           VERSION
cp01       Ready    control-plane   v1.34.12
cp02       Ready    control-plane   v1.34.12
cp03       Ready    control-plane   v1.34.12
worker01   Ready    <none>          v1.34.12
worker02   Ready    <none>          v1.34.12
worker03   Ready    <none>          v1.34.12
```

Control Plane Pod 확인:

```bash
kubectl get pods -n kube-system -o wide
```

etcd는 각 Control Plane Node에 하나씩 구성하여
3-member stacked etcd Cluster로 구성하였다.

```text
cp01 ─ etcd
cp02 ─ etcd
cp03 ─ etcd
```

---

## 9. Node Name Issue

초기 `kubeadm-config.yaml` 생성 시 다음 기본 설정이 포함되어 있었다.

```yaml
nodeRegistration:
  name: node
```

이 값을 실제 hostname인 `cp01`로 변경하지 않고 초기화를 진행하여
첫 번째 Control Plane이 Kubernetes에서 `node`라는 이름으로 등록되었다.

```text
OS Hostname        Kubernetes Node Name

cp01        →      node
```

이후 Control Plane을 안전하게 제거한 뒤 `cp01`이라는 이름으로
다시 Cluster에 Join하여 Node 이름을 일치시켰다.

최종 상태:

```text
cp01
cp02
cp03
worker01
worker02
worker03
```

상세한 문제 분석 및 복구 과정은 Troubleshooting 문서에서 다룬다.

→ [Troubleshooting](troubleshooting.md)

---

## Next Step

Kubernetes Cluster Bootstrap 이후 HAProxy와 Keepalived를 이용하여
Kubernetes API Server의 고가용성 Endpoint를 구성하였다.

→ [Control Plane HA](04-control-plane-ha.md)
