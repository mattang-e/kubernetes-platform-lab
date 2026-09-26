# Kubernetes Cluster Troubleshooting

## Overview

Kubernetes HA Cluster 구축 과정에서 발생한 주요 장애와
원인 분석 및 해결 과정을 정리하였다.

```text
Symptom
   ↓
Investigation
   ↓
Root Cause
   ↓
Resolution
   ↓
Verification
```

---

## 1. Kubernetes 1.35 and cgroup v1 Compatibility

### Symptom

Rocky Linux 8.10 환경에서 Kubernetes 1.35 Cluster를 구성하려 했으나
`kubeadm init` 과정에서 cgroup 관련 문제가 발생하였다.

현재 cgroup 상태를 확인하였다.

```bash
stat -fc %T /sys/fs/cgroup/
```

확인 결과:

```text
tmpfs
```

해당 환경은 cgroup v1을 사용하고 있었다.

또한 Rocky Linux 8.10의 기본 Kernel은 4.18 계열이었다.

```bash
uname -r
```

### Investigation

Kubernetes 1.35 환경과 현재 Rocky Linux 8.10의
cgroup 및 Kernel 구성을 함께 검토하였다.

cgroup v2 전환을 위해 Kernel Parameter 변경도 시도하였다.

```text
systemd.unified_cgroup_hierarchy=1
```

그러나 재부팅 후에도 실제 Kernel Command Line에
설정이 정상적으로 적용되지 않았다.

확인:

```bash
cat /proc/cmdline
```

### Root Cause

현재 Lab은 Rocky Linux 8.10, Kernel 4.18 및 cgroup v1 환경으로
구성되어 있었다.

Kubernetes 1.35 적용 과정에서 현재 OS 및 cgroup 환경과의
호환성 제약을 확인하였으며, cgroup v2 전환도 시도했지만
현재 Lab 환경에서는 설정이 정상적으로 적용되지 않았다.

### Resolution

Kubernetes Repository를 v1.35에서 v1.34로 변경하였다.

Ansible을 이용하여 전체 Kubernetes Node의 Repository를 수정하였다.

```bash
ansible k8s -i inventory.ini -m replace \
  -a "path=/etc/yum.repos.d/kubernetes.repo regexp='v1\.35/rpm' replace='v1.34/rpm'"
```

기존 설치 Package도 Kubernetes 1.34 계열로 맞추었다.

```bash
dnf downgrade -y kubeadm kubelet kubectl \
  --disableexcludes=kubernetes
```

실패했던 Control Plane 초기화 상태는 다음 명령으로 정리하였다.

```bash
kubeadm reset -f
```

이후 Kubernetes 1.34 환경에서 Cluster를 다시 구성하였다.

### Verification

```bash
kubectl get nodes
```

Control Plane과 Worker Node가 정상적으로 Cluster에 Join되고
`Ready` 상태로 동작하는 것을 확인하였다.

### Lesson Learned

Kubernetes Version을 결정할 때 Kubernetes 자체 Version뿐 아니라
OS, Kernel, cgroup Version 및 Container Runtime의 호환성을
함께 검토해야 한다는 점을 확인하였다.

---

## 2. Keepalived VIP Conflict

### Symptom

HAProxy와 Keepalived를 이용하여 Kubernetes API용
Virtual IP를 구성하는 과정에서 VIP가 정상적으로 하나의
Load Balancer에만 할당되지 않는 문제가 발생하였다.

구성:

```text
lb01    192.168.10.99
lb02    192.168.10.100

VIP     192.168.10.115
```

정상적인 경우 하나의 Load Balancer가 MASTER가 되어
VIP를 소유해야 한다.

그러나 Keepalived 간 상태 동기화가 정상적으로 이루어지지 않았다.

### Investigation

lb01과 lb02의 Keepalived 상태 및 Network 설정을 점검하였다.

```bash
systemctl status keepalived
```

VIP 상태 확인:

```bash
ip addr
```

Keepalived Service 자체는 동작하고 있었기 때문에
두 Load Balancer 사이의 VRRP 통신을 확인하였다.

Host Firewall인 `firewalld`가 활성화되어 있는 것도 확인하였다.

### Root Cause

`firewalld`에 의해 Keepalived가 사용하는
VRRP 통신이 차단되고 있었다.

VRRP는 TCP/UDP Port가 아니라 IP Protocol 112를 사용한다.

두 Load Balancer가 서로의 VRRP Advertisement를 정상적으로
수신하지 못하면서 VIP 상태가 정상적으로 결정되지 않았다.

### Resolution

문제 원인을 확인하기 위해 `firewalld`를 일시적으로 중지하여
VRRP 통신 여부를 검증하였다.

```bash
systemctl stop firewalld
```

이후 Keepalived가 정상적으로 MASTER/BACKUP 상태를 구성하였다.

### Verification

VIP 확인:

```bash
ip addr
```

Kubernetes API Endpoint 확인:

```bash
kubectl cluster-info
```

API Server가 다음 VIP를 통해 정상적으로 접근되는 것을 확인하였다.

```text
https://192.168.10.115:6443
```

### Lesson Learned

Keepalived 구성에서는 Service 상태만 확인하는 것이 아니라
VRRP와 같은 Network Protocol이 Host Firewall을 통과할 수 있는지
함께 확인해야 한다.

---

## 3. Calico Network Failure with firewalld

### Symptom

Calico 설치 후 일부 Node에서 `calico-node`가
정상적인 Ready 상태가 되지 않았다.

```text
calico-node    0/1    Running
```

또한 Node에는 다음 Taint가 존재하였다.

```text
node.kubernetes.io/network-unavailable
```

Control Plane Node를 drain하는 과정에서도
Calico 관련 Pod를 정상적으로 Eviction하지 못하는 문제가 발생하였다.

```text
Cannot evict pod ...
would violate pod's disruption budget
```

### Investigation

Calico Pod 상태를 확인하였다.

```bash
kubectl get pods -n calico-system -o wide
```

문제가 발생한 `calico-node`를 상세 확인하였다.

```bash
kubectl describe pod -n calico-system <CALICO_NODE_POD>
```

Startup Probe에서 다음 오류를 확인하였다.

```text
BIRD is not ready
error querying BIRD
unable to connect to BIRDv4 socket
```

Calico Log에서도 Network Component가 정상적으로 준비되지 않는
상태를 확인하였다.

Kubernetes Node의 Host Firewall이 활성화되어 있었기 때문에
Firewall 정책이 Calico Node 간 통신에 영향을 주는지 확인하였다.

### Root Cause

Lab 환경에서 `firewalld`를 중지한 후 Calico가 정상화되었기 때문에
Host Firewall 정책이 Calico Network 통신에 영향을 주고 있음을 확인하였다.

다만 장애 당시 정확히 어떤 Calico Protocol 또는 Port가
직접적인 원인이었는지는 특정하지 않았기 때문에,
특정 Port를 원인으로 단정하지 않았다.

### Resolution

문제 범위를 확인하기 위해 Ansible을 사용하여
Kubernetes Node의 `firewalld`를 일시적으로 중지하였다.

```bash
ansible k8s -i inventory.ini -m service \
  -a "name=firewalld state=stopped"
```

이후 Calico Component가 정상 상태로 복구되었다.

### Verification

```bash
kubectl get pods -n calico-system -o wide
```

`calico-node`가 다음과 같이 정상화된 것을 확인하였다.

```text
calico-node    1/1    Running
```

Node 상태:

```bash
kubectl get nodes
```

모든 Kubernetes Node가 `Ready` 상태인 것을 확인하였다.

### Remaining Work

현재 구성은 Lab 환경에서 문제 원인을 확인하기 위한 조치이다.

향후 Calico의 실제 Encapsulation/BGP 설정과 필요한 Network Traffic을
확인하여 최소한의 Firewall Rule만 허용한 후
`firewalld`를 다시 활성화할 예정이다.

### Lesson Learned

Kubernetes Network 장애 발생 시 CNI Pod 상태뿐 아니라
Node Taint, Startup Probe, CNI Log 및 Host Firewall까지
함께 확인해야 한다.

---

## 4. Control Plane Node Name Correction

### Symptom

첫 번째 Control Plane을 `kubeadm init`으로 구성한 후
OS Hostname은 `cp01`이지만 Kubernetes에서는 Node 이름이
`node`로 등록된 것을 확인하였다.

```bash
kubectl get nodes
```

```text
NAME
node
cp02
cp03
worker01
worker02
worker03
```

### Investigation

초기 Cluster 구성에 사용한 `kubeadm-config.yaml`을 확인하였다.

설정에 다음 항목이 존재하였다.

```yaml
nodeRegistration:
  name: node
```

`kubeadm config print init-defaults`를 기반으로 설정 파일을 작성하면서
기본값인 `node`가 그대로 남아 있었다.

### Root Cause

OS Hostname 문제는 아니었다.

`kubeadm-config.yaml`의 다음 설정에 의해:

```yaml
nodeRegistration:
  name: node
```

첫 번째 Control Plane이 Kubernetes에 `node`라는 이름으로
등록된 것이 원인이었다.

### Resolution

이미 3개의 Control Plane이 구성된 상태였기 때문에
Cluster 전체를 다시 구축하지 않고 해당 Control Plane만
안전하게 제거 후 재가입하였다.

먼저 Node를 drain하였다.

```bash
kubectl drain node \
  --ignore-daemonsets \
  --delete-emptydir-data
```

cp01에서 kubeadm 상태를 초기화하였다.

```bash
kubeadm reset -f
```

기존 Kubernetes Node Object를 삭제하였다.

```bash
kubectl delete node node
```

기존 cp01의 etcd Member가 제거되고
cp02와 cp03의 etcd Member가 정상적으로 유지되는 것을 확인하였다.
이후 새로운 Certificate Key와 Join Token을 생성하였다.

```bash
kubeadm init phase upload-certs --upload-certs
```

```bash
kubeadm token create --print-join-command
```

cp01을 다시 Control Plane으로 Join하였다.

```bash
kubeadm join 192.168.10.115:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY>
```

### Verification

Node 이름 확인:

```bash
kubectl get nodes
```

```text
cp01       Ready    control-plane
cp02       Ready    control-plane
cp03       Ready    control-plane
worker01   Ready
worker02   Ready
worker03   Ready
```

etcd Cluster도 다시 확인하였다.

```bash
kubectl exec -n kube-system etcd-cp02 -c etcd -- \
etcdctl member list \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
--key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

최종적으로 다음 3개의 etcd Member가 정상적으로 구성된 것을 확인하였다.

```text
cp01
cp02
cp03
```

### Lesson Learned

`kubeadm config print init-defaults`로 생성한 설정 파일은
그대로 사용하지 않고 실제 환경에 맞게 모든 값을 검토해야 한다.

또한 HA Control Plane 환경에서는 하나의 Control Plane을 제거하더라도
나머지 Control Plane과 etcd Quorum 상태를 확인하면서 작업하면
Cluster 전체를 재구축하지 않고 Node를 교체할 수 있다.

---

## Summary

이번 Cluster 구축 과정에서 다음 문제를 직접 분석하고 해결하였다.

| Issue | Cause | Resolution |
|---|---|---|
| Kubernetes 1.35 적용 문제 | Rocky 8.10 / Kernel 4.18 / cgroup v1 환경과의 호환성 제약 | Lab 환경을 유지하고 Kubernetes 1.34로 구성 |
| Keepalived VIP 문제 | Firewall의 VRRP 통신 차단 | Firewall 격리 테스트 후 VIP 정상화 |
| Calico Network 장애 | Host Firewall 영향 | Firewall 격리 테스트 및 Network 복구 |
| cp01 Node 이름 오류 | `nodeRegistration.name` 설정 | Control Plane 제거 후 재가입 |

이 과정을 통해 Kubernetes Cluster 운영 시
단순 Resource 상태뿐 아니라 OS, Network, Firewall, etcd,
CNI 및 Control Plane 구성까지 함께 분석해야 한다는 점을 확인하였다.
