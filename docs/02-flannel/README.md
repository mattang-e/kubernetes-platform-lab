# Flannel CNI

## Overview

Flannel은 Kubernetes CNI(Container Network Interface) Plugin 중 하나이며 Pod 간 Overlay Network를 제공합니다.

---

## Why Flannel?

kubeadm은 Kubernetes Control Plane만 생성합니다.

Pod 간 네트워크는 구성하지 않으므로 별도의 CNI Plugin이 필요합니다.

```

kubeadm init

↓

Control Plane 생성

↓

Pod Network 없음

↓

Flannel 설치

↓

Node Ready

```

---

# 1. Download Flannel

```bash

curl -LO https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml

```

---

# 2. Modify Pod CIDR

kubeadm init에서 지정한 Pod Network와 동일하게 수정합니다.

```yaml

net-conf.json: |

{

  "Network":"172.17.0.0/16",

  "Backend":{

      "Type":"vxlan"

  }

}

```

---

# 3. Specify Network Interface

```yaml

args:

- --ip-masq

- --kube-subnet-mgr

- --iface=eth0

```

---

# 4. Deploy Flannel

```bash

kubectl apply -f kube-flannel.yml

```

---

# Verification

```bash

kubectl get pods -A

```

```bash

kubectl get nodes

```

Node가 Ready 상태인지 확인합니다.

---

# VXLAN Network

```

Pod

↓

veth

↓

cni0

↓

flannel.1

↓

VXLAN

↓

Remote Node

↓

cni0

↓

veth

↓

Pod

```

---

# Troubleshooting

Flannel Pod 확인

```bash

kubectl get pods -n kube-flannel

```

로그 확인

```bash

kubectl logs -n kube-flannel <POD_NAME>

```
