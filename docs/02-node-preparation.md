# Node Preparation

## Overview

Kubernetes 클러스터를 구축하기 전 Control Plane 및 Worker Node의
운영체제와 Container Runtime을 구성하였다.

모든 Kubernetes Node는 Rocky Linux 8.10을 사용하며,
Container Runtime은 containerd를 사용하였다.

노드 간 설정 차이를 최소화하기 위해 공통 OS 설정 및 Kubernetes 패키지 설치는
Ansible을 활용하여 구성하였다.

---

## 1. Node Environment

Kubernetes Cluster는 다음 6개의 Node로 구성하였다.

| Role | Hostname | IP |
|---|---|---|
| Control Plane | cp01 | 192.168.10.101 |
| Control Plane | cp02 | 192.168.10.102 |
| Control Plane | cp03 | 192.168.10.103 |
| Worker | worker01 | 192.168.10.111 |
| Worker | worker02 | 192.168.10.112 |
| Worker | worker03 | 192.168.10.113 |

각 Node는 XCP-ng 위에 VM으로 생성하였다.

### VM Specification

| Role | vCPU | Memory |
|---|---:|---:|
| Control Plane | 2 | 4 GB |
| Worker | 4 | 12~16 GB |

Operating System

```text
Rocky Linux 8.10 Minimal
```

---

## 2. Hostname Configuration

각 Node의 역할을 구분할 수 있도록 hostname을 설정하였다.

예시:

```bash
hostnamectl set-hostname cp01
```

설정 확인:

```bash
hostnamectl
hostname
```

각 Node에 다음과 같이 hostname을 설정하였다.

```text
cp01
cp02
cp03

worker01
worker02
worker03
```

---

## 3. Static Network Configuration

Kubernetes Node 간 통신이 항상 동일한 IP를 사용하도록
각 VM에 Static IP를 구성하였다.

Node Network:

```text
192.168.10.0/24
```

Control Plane:

```text
cp01    192.168.10.101
cp02    192.168.10.102
cp03    192.168.10.103
```

Worker:

```text
worker01    192.168.10.111
worker02    192.168.10.112
worker03    192.168.10.113
```

네트워크 설정 확인:

```bash
ip addr
ip route
```

Node 간 연결 확인:

```bash
ping 192.168.10.102
ping 192.168.10.103
```

---

## 4. Disable Swap

Kubernetes Node에서 swap을 비활성화하였다.

현재 swap 확인:

```bash
swapon --show
```

swap 비활성화:

```bash
swapoff -a
```

재부팅 후에도 swap이 활성화되지 않도록 `/etc/fstab`의
swap 항목을 비활성화하였다.

확인:

```bash
free -h
swapon --show
```

---

## 5. SELinux Configuration

Kubernetes Lab 환경에서 SELinux로 인한 정책 충돌을 줄이기 위해
Permissive Mode로 변경하였다.

```bash
setenforce 0
```

영구 설정:

```bash
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

확인:

```bash
getenforce
```

Expected:

```text
Permissive
```

---

## 6. Kernel Modules

Container Networking에 필요한 Kernel Module을 활성화하였다.

```bash
modprobe overlay
modprobe br_netfilter
```

재부팅 후에도 자동으로 로드되도록 설정하였다.

```bash
cat <<EOF > /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

확인:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

---

## 7. Kernel Network Parameters

Kubernetes Pod Networking을 위해 필요한 sysctl parameter를 설정하였다.

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

설정 적용:

```bash
sysctl --system
```

확인:

```bash
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.ipv4.ip_forward
```

Expected:

```text
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
```

`ip_forward`를 활성화하여 Node가 서로 다른 네트워크 인터페이스 간
패킷을 전달할 수 있도록 하였으며, `bridge-nf-call-iptables`를 활성화하여
Linux Bridge를 통과하는 트래픽이 iptables 정책의 영향을 받을 수 있도록 하였다.

---

## 8. Container Runtime

Kubernetes Container Runtime으로 containerd를 사용하였다.

Docker Engine을 별도로 설치하지 않고 Kubernetes가 CRI를 통해
containerd와 직접 통신하도록 구성하였다.

```text
kubelet
   │
   │ CRI
   ▼
containerd
   │
   ▼
  runc
   │
   ▼
Linux Kernel
```

containerd 설치 후 기본 설정 파일을 생성하였다.

```bash
mkdir -p /etc/containerd

containerd config default > /etc/containerd/config.toml
```

Kubernetes의 systemd 기반 cgroup 관리와 일치하도록
다음 옵션을 활성화하였다.

```toml
SystemdCgroup = true
```

containerd 재시작 및 자동 시작 설정:

```bash
systemctl restart containerd
systemctl enable containerd
```

상태 확인:

```bash
systemctl status containerd
```

---

## 9. Kubernetes Repository

Kubernetes 공식 RPM Repository를 구성하였다.

프로젝트에서는 Kubernetes `v1.34` Repository를 사용하였다.

```ini
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.34/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
```

Repository 확인:

```bash
dnf repolist
```

---

## 10. Kubernetes Package Installation

모든 Kubernetes Node에 다음 패키지를 설치하였다.

```text
kubeadm
kubelet
kubectl
```

설치:

```bash
dnf install -y kubelet kubeadm kubectl \
  --disableexcludes=kubernetes
```

kubelet 자동 시작 설정:

```bash
systemctl enable kubelet
```

> Cluster 초기화 또는 Join 전에는 kubelet이 정상적으로 동작하지 않을 수 있다.
> kubeadm을 통해 Node 설정이 완료되면 kubelet configuration이 생성된다.

설치 버전 확인:

```bash
kubeadm version -o short
kubelet --version
kubectl version --client
```

본 프로젝트에서는 Kubernetes `v1.34.x` 버전을 사용하였다.

---

## 11. Ansible Automation

6개의 Kubernetes Node에 동일한 설정을 반복 적용하기 위해
Mac을 Ansible Control Node로 사용하였다.

```text
Mac
 │
 │ Ansible / SSH
 │
 ├── cp01
 ├── cp02
 ├── cp03
 ├── worker01
 ├── worker02
 └── worker03
```

Inventory는 Control Plane과 Worker를 각각 그룹화하였다.

```ini
[control_plane]
cp01 ansible_host=192.168.10.101 ansible_user=root
cp02 ansible_host=192.168.10.102 ansible_user=root
cp03 ansible_host=192.168.10.103 ansible_user=root

[workers]
worker01 ansible_host=192.168.10.111 ansible_user=root
worker02 ansible_host=192.168.10.112 ansible_user=root
worker03 ansible_host=192.168.10.113 ansible_user=root

[k8s:children]
control_plane
workers

[k8s:vars]
ansible_python_interpreter=/usr/bin/python3.12
```

연결 확인:

```bash
ansible k8s -i inventory.ini -m ping
```

Ansible을 이용하여 Kubernetes Node의 공통 설정과
패키지 관리를 동일하게 적용할 수 있도록 구성하였다.

---

## 12. Node Preparation Verification

Kubernetes Cluster를 초기화하기 전 각 Node에서 다음 항목을 확인하였다.

```bash
hostname

swapon --show

getenforce

lsmod | grep br_netfilter

sysctl net.ipv4.ip_forward

systemctl status containerd

kubeadm version -o short

kubelet --version
```

최종적으로 모든 Node에서 다음 조건을 만족하는 것을 확인하였다.

```text
Swap              Disabled
SELinux           Permissive
Container Runtime containerd
SystemdCgroup     true
IP Forwarding     Enabled
br_netfilter      Loaded
Kubernetes        v1.34.x
```

---

## Next Step

Node 기본 구성이 완료된 후 다음 단계에서 kubeadm을 이용하여
첫 번째 Control Plane을 초기화하고 추가 Control Plane 및 Worker Node를
Cluster에 Join하였다.

→ [Kubernetes Cluster Bootstrap](03-kubeadm-cluster.md)
