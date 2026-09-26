# Control Plane High Availability

## Overview

Kubernetes Control Plane의 단일 장애 지점 을 줄이기 위해
3개의 Control Plane Node와 2개의 Load Balancer를 구성하였다.

Kubernetes API Server의 단일 진입점은 HAProxy와 Keepalived를 이용하여
Virtual IP(VIP)로 구성하였다.

```text
                        Client
                          │
                          ▼
                 192.168.10.115:6443
                     Virtual IP
                          │
             ┌────────────┴────────────┐
             │                         │
           lb01                      lb02
      192.168.10.99             192.168.10.100
   HAProxy + Keepalived      HAProxy + Keepalived
             │                         │
             └────────────┬────────────┘
                          │
              ┌───────────┼───────────┐
              ▼           ▼           ▼
            cp01        cp02        cp03
       192.168.10.101  .102        .103
          API :6443   API :6443   API :6443
```

---

## 1. Load Balancer 구성

Kubernetes API Server의 고가용성을 위해 2개의 Load Balancer VM을 구성하였다.

| Role | Hostname | IP |
|---|---|---|
| Load Balancer | lb01 | 192.168.10.99 |
| Load Balancer | lb02 | 192.168.10.100 |
| Virtual IP | VIP | 192.168.10.115 |

각 Load Balancer에는 다음 Component를 설치하였다.

```text
HAProxy
Keepalived
```

HAProxy는 Kubernetes API Server의 Load Balancing을 담당하고,
Keepalived는 두 Load Balancer 사이에서 Virtual IP를 관리한다.

---

## 2. HAProxy

HAProxy는 TCP 6443 Port로 들어오는 Kubernetes API 요청을
3개의 Control Plane Node로 전달하도록 구성하였다.

```text
                   HAProxy :6443
                         │
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         cp01:6443  cp02:6443  cp03:6443
```

HAProxy 설정 파일:

```bash
/etc/haproxy/haproxy.cfg
```

구성 예시는 다음과 같다.

```haproxy
frontend kubernetes-api
    bind *:6443
    mode tcp
    default_backend kubernetes-control-plane

backend kubernetes-control-plane
    mode tcp
    balance roundrobin

    server cp01 192.168.10.101:6443 check
    server cp02 192.168.10.102:6443 check
    server cp03 192.168.10.103:6443 check
```

> 위 설정은 프로젝트의 HAProxy 구성 구조를 설명하기 위한 예시이다.
> 실제 환경의 `/etc/haproxy/haproxy.cfg`와 옵션이 일부 다를 수 있다.

HAProxy 활성화:

```bash
systemctl enable --now haproxy
```

상태 확인:

```bash
systemctl status haproxy
```

Listening Port 확인:

```bash
ss -lntp | grep 6443
```

---

## 3. Keepalived

HAProxy 자체가 단일 장애 지점이 되지 않도록
`lb01`, `lb02`에 Keepalived를 구성하였다.

Keepalived는 VRRP를 이용하여 두 Load Balancer 중 하나가
Virtual IP를 소유하도록 구성한다.

```text
               192.168.10.115
                     VIP
                      │
             ┌────────┴────────┐
             │                 │
           lb01              lb02
          MASTER             BACKUP
             │                 │
             └────── VRRP ─────┘
```

정상 상태에서는 MASTER가 VIP를 소유하고,
MASTER 장애 시 BACKUP이 VIP를 인계할 수 있는 구조로 구성하였다.

Keepalived 설정 파일:

```bash
/etc/keepalived/keepalived.conf
```

구성 형태는 다음과 같다.

```conf
vrrp_instance VI_1 {
    state MASTER
    interface <NETWORK_INTERFACE>

    virtual_router_id 51
    priority <PRIORITY>

    virtual_ipaddress {
        192.168.10.115
    }
}
```

`lb02`는 동일한 `virtual_router_id`를 사용하되
BACKUP 역할과 더 낮은 priority를 사용하도록 구성하였다.

> Interface 이름과 priority는 실제 Load Balancer의 설정에 맞게 지정한다.

Keepalived 활성화:

```bash
systemctl enable --now keepalived
```

상태 확인:

```bash
systemctl status keepalived
```

---

## 4. Virtual IP 확인

현재 VIP를 소유한 Load Balancer에서 다음 명령으로 확인하였다.

```bash
ip addr
```

또는:

```bash
ip addr | grep 192.168.10.115
```

MASTER Load Balancer에서 다음 VIP가 확인된다.

```text
192.168.10.115
```

BACKUP Load Balancer에서는 정상 상태일 경우 동일한 VIP를
동시에 소유하지 않는다.

---

## 5. Kubernetes API Endpoint

HAProxy와 Keepalived로 구성한 VIP를 Kubernetes의
`controlPlaneEndpoint`로 사용하였다.

`kubeadm-config.yaml`:

```yaml
controlPlaneEndpoint: "192.168.10.115:6443"
```

따라서 Kubernetes Component와 Client는 개별 Control Plane IP가 아니라
다음 Endpoint를 통해 Kubernetes API에 접근한다.

```text
https://192.168.10.115:6443
```

전체 요청 흐름:

```text
kubectl
   │
   ▼
192.168.10.115:6443
   │
   ▼
Keepalived VIP
   │
   ▼
HAProxy
   │
   ├── 192.168.10.101:6443
   ├── 192.168.10.102:6443
   └── 192.168.10.103:6443
          │
          ▼
    kube-apiserver
```

이 구조를 통해 Client는 특정 Control Plane Node의 IP를
직접 지정할 필요가 없다.

---

## 6. HA 동작 구조

Control Plane Node 3대의 kube-apiserver는 모두 요청을 처리할 수 있다.

```text
cp01   kube-apiserver
cp02   kube-apiserver
cp03   kube-apiserver
```

HAProxy는 정상 상태인 API Server로 요청을 전달한다.

반면 `kube-scheduler`와 `kube-controller-manager`는
각 Control Plane에 실행되지만 Leader Election을 통해
하나의 Instance가 Active 역할을 수행한다.

etcd는 3-member Cluster로 구성되어 quorum 기반으로 동작한다.

```text
Control Plane

cp01
 ├─ kube-apiserver
 ├─ kube-scheduler
 ├─ kube-controller-manager
 └─ etcd

cp02
 ├─ kube-apiserver
 ├─ kube-scheduler
 ├─ kube-controller-manager
 └─ etcd

cp03
 ├─ kube-apiserver
 ├─ kube-scheduler
 ├─ kube-controller-manager
 └─ etcd
```

---

## 7. HA Verification

Kubernetes API가 VIP를 통해 정상적으로 동작하는지 확인하였다.

```bash
kubectl cluster-info
```
```text
Kubernetes control plane is running at https://192.168.10.115:6443
```

Node 상태:

```bash
kubectl get nodes
```

```text
NAME       STATUS   ROLES           VERSION
cp01       Ready    control-plane   v1.34.12
cp02       Ready    control-plane   v1.34.12
cp03       Ready    control-plane   v1.34.12
worker01   Ready    <none>          v1.34.12
worker02   Ready    <none>          v1.34.12
worker03   Ready    <none>          v1.34.12
```

HAProxy 상태:

```bash
systemctl status haproxy
```

Keepalived 상태:

```bash
systemctl status keepalived
```

VIP 확인:

```bash
ip addr | grep 192.168.10.115
```

---

## 8. Troubleshooting Experience

초기 Keepalived 구성 과정에서 `lb01`, `lb02`가 동시에
Virtual IP를 소유하는 문제가 발생하였다.

확인 결과 Host Firewall 정책으로 인해 Load Balancer 간
VRRP 통신이 정상적으로 이루어지지 않았다.

VRRP는 일반적인 TCP/UDP Port 기반 통신이 아니라
IP Protocol 112를 사용하므로 이에 맞는 Firewall 정책이 필요하였다.

Firewall 정책을 수정한 후 MASTER/BACKUP 상태가 정상적으로 구성되고
하나의 Load Balancer만 VIP를 소유하는 것을 확인하였다.

```text
Before

lb01 ─ VIP 192.168.10.115
lb02 ─ VIP 192.168.10.115
          X


After

lb01 (MASTER) ─ VIP 192.168.10.115
lb02 (BACKUP) ─ No VIP
          ✓
```

상세한 장애 분석 과정은 Troubleshooting 문서에서 별도로 정리한다.

→ [Troubleshooting](troubleshooting.md)

---

## 9. Lab Environment Limitation

프로젝트의 Kubernetes Node 및 Load Balancer VM은
단일 XCP-ng Host 위에서 실행되는 Lab 환경으로 구성하였다.

따라서 Kubernetes Control Plane과 API Endpoint는 논리적으로
고가용성 구조를 구성하였지만, 물리적인 XCP-ng Host 자체의 장애까지
보호하는 구조는 아니다.

Production 환경에서는 Control Plane과 Load Balancer를
서로 다른 Physical Host 또는 Availability Zone에 분산하여
Host Level의 단일 장애 지점도 제거해야 한다.

---

## Next Step

Control Plane HA 구성과 Kubernetes Cluster Bootstrap 이후
Calico를 이용하여 Pod Network를 구성하였다.

→ [Calico Network](05-calico-network.md)
