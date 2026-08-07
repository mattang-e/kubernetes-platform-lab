# Kubernetes Cluster 구축 (kubeadm)

## Overview

kubeadm을 이용하여 Kubernetes Control Plane과 Worker Node를 구축하는 과정을 정리한 문서입니다.

---

## Environment

| Component | Version |

|-----------|---------|

| Ubuntu | 24.04 |

| Kubernetes | v1.35.0 |

| kubeadm | 1.35.0 |

| kubelet | 1.35.0 |

| kubectl | 1.35.0 |

| Container Runtime | containerd |

---

# 1. Enable IP Forwarding

## Why?

Kubernetes에서는 Node가 Pod 간 패킷을 전달해야 하므로 Linux Kernel의 IP Forwarding 기능이 필요합니다.

```bash

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf

net.ipv4.ip_forward=1

EOF

```

변경한 sysctl 값을 커널에 즉시 적용합니다.

```bash

sudo sysctl --system

```

적용 여부를 확인합니다.

```bash

sysctl net.ipv4.ip_forward

```

예시

```text

net.ipv4.ip_forward = 1

```

---

# 2. Install Kubernetes Packages

패키지 저장소를 등록하기 위한 기본 패키지를 설치합니다.

```bash

sudo apt update

sudo apt install -y apt-transport-https ca-certificates curl

```

---

# 3. Add Kubernetes Repository

공식 GPG Key 등록

```bash

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \

| sudo gpg --dearmor \

-o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```

Repository 등록

```bash

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \

| sudo tee /etc/apt/sources.list.d/kubernetes.list

```

업데이트

```bash

sudo apt update

```

---

# 4. Install kubeadm

설치 가능한 버전 확인

```bash

apt-cache madison kubeadm

```

설치

```bash

sudo apt install -y \

kubelet=1.35.0-1.1 \

kubeadm=1.35.0-1.1 \

kubectl=1.35.0-1.1

```

버전 고정

```bash

sudo apt-mark hold kubelet kubeadm kubectl

```

---

# 5. Initialize Control Plane

Control Plane Node의 IP를 변수에 저장합니다.

```bash

IP_ADDR=$(ip addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

```

초기화

```bash

sudo kubeadm init \

--apiserver-cert-extra-sans=controlplane \

--apiserver-advertise-address=$IP_ADDR \

--pod-network-cidr=172.17.0.0/16 \

--service-cidr=172.20.0.0/16

```

## Option

| Option | Description |

|---------|-------------|

| apiserver-cert-extra-sans | API Server 인증서 SAN 추가 |

| apiserver-advertise-address | API Server가 사용할 IP |

| pod-network-cidr | Pod Network 대역 |

| service-cidr | ClusterIP 대역 |

---

# 6. Configure kubectl

```bash

mkdir -p ~/.kube

sudo cp -i /etc/kubernetes/admin.conf ~/.kube/config

sudo chown $(id -u):$(id -g) ~/.kube/config

```

---

# 7. Join Worker Node

Join Token 생성

```bash

kubeadm token create --print-join-command

```

Worker에서 실행

```bash

kubeadm join <CONTROL_PLANE_IP>:6443 \

--token <TOKEN> \

--discovery-token-ca-cert-hash sha256:<HASH>

```

---

# Verification

```bash

kubectl get nodes

```

```bash

kubectl get pods -A

```
