# Kubernetes HA Cluster 구축 (kubeadm)

kubeadm 기반으로 Kubernetes 클러스터를 구축하고,  

Flannel CNI를 적용하여 Pod 간 네트워크 통신이 가능하도록 구성합니다.

## Architecture

```text

Control Plane

    │

    ├── kube-apiserver

    ├── etcd

    ├── kube-scheduler

    └── kube-controller-manager

Worker Node

    │

    ├── kubelet

    ├── kube-proxy

    └── containerd

Pod Network

    │

    └── Flannel (VXLAN)

```

## Environment

- Kubernetes: `v1.35.0`

- kubeadm: `1.35.0-1.1`

- kubelet: `1.35.0-1.1`

- kubectl: `1.35.0-1.1`

- Container Runtime: `containerd`

- CNI: `Flannel`

- Pod CIDR: `172.17.0.0/16`

- Service CIDR: `172.20.0.0/16`

---

## 1. IP Forwarding 활성화

Kubernetes에서는 Pod 간 또는 Node 간 패킷을 전달해야 하므로 Linux의 IP Forwarding 기능을 활성화합니다.

```bash

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf

net.ipv4.ip_forward=1

EOF

```

설정값을 재부팅 없이 즉시 반영합니다.

```bash

sudo sysctl --system

```

설정이 정상적으로 적용되었는지 확인합니다.

```bash

sysctl net.ipv4.ip_forward

```

정상 결과:

```text

net.ipv4.ip_forward = 1

```

---

## 2. Kubernetes 설치에 필요한 패키지 설치

APT 저장소를 HTTPS로 사용할 수 있도록 필요한 패키지를 설치합니다.

```bash

sudo apt-get update

sudo apt-get install -y \

  apt-transport-https \

  ca-certificates \

  curl

```

---

## 3. Kubernetes Repository GPG Key 등록

Kubernetes 공식 패키지의 서명을 검증하기 위한 GPG Key를 등록합니다.

```bash

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.35/deb/Release.key \

  | sudo gpg --dearmor \

  -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

```

---

## 4. Kubernetes APT Repository 등록

Kubernetes `v1.35` 패키지 저장소를 등록합니다.

```bash

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.35/deb/ /' \

  | sudo tee /etc/apt/sources.list.d/kubernetes.list

```

Repository 정보를 갱신합니다.

```bash

sudo apt-get update

```

---

## 5. 설치 가능한 Kubernetes 버전 확인

```bash

sudo apt-cache madison kubeadm

```

예시:

```text

kubeadm | 1.35.0-1.1

```

---

## 6. kubelet, kubeadm, kubectl 설치

Kubernetes 구성 요소의 버전을 동일하게 맞춰 설치합니다.

```bash

sudo apt-get install -y \

  kubelet=1.35.0-1.1 \

  kubeadm=1.35.0-1.1 \

  kubectl=1.35.0-1.1

```

패키지가 자동으로 업그레이드되지 않도록 버전을 고정합니다.

```bash

sudo apt-mark hold kubelet kubeadm kubectl

```

---

## 7. Control Plane 초기화

현재 서버의 `eth0` 인터페이스 IP를 변수로 저장합니다.

```bash

IP_ADDR=$(ip addr show eth0 \

  | grep -oP '(?<=inet\s)\d+(\.\d+){3}')

```

확인:

```bash

echo $IP_ADDR

```

Control Plane을 초기화합니다.

```bash

sudo kubeadm init \

  --kubernetes-version=v1.35.0 \

  --apiserver-cert-extra-sans=controlplane \

  --apiserver-advertise-address=$IP_ADDR \

  --pod-network-cidr=172.17.0.0/16 \

  --service-cidr=172.20.0.0/16

```

### 주요 옵션 설명

`--apiserver-cert-extra-sans=controlplane`

API Server 인증서의 SAN(Subject Alternative Name)에 `controlplane`이라는 DNS 이름을 추가합니다.

```text

https://controlplane:6443

```

형태로 API Server에 접속할 때 TLS 인증서 검증이 가능하도록 합니다.

`--apiserver-advertise-address`

API Server가 자신의 주소로 사용할 Control Plane Node의 IP입니다.

```text

$IP_ADDR:6443

```

`--pod-network-cidr`

Pod에 할당할 IP 대역입니다.

```text

172.17.0.0/16

```

`--service-cidr`

ClusterIP Service에 할당할 IP 대역입니다.

```text

172.20.0.0/16

```

---

## 8. kubectl 설정

`kubeadm init` 완료 후 일반 사용자 계정에서 `kubectl`을 사용하기 위해 kubeconfig를 설정합니다.

```bash

mkdir -p $HOME/.kube

```

```bash

sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

```

```bash

sudo chown $(id -u):$(id -g) $HOME/.kube/config

```

정상 동작 확인:

```bash

kubectl get nodes

```

CNI를 설치하기 전에는 Node가 다음처럼 `NotReady` 상태일 수 있습니다.

```text

NAME           STATUS

controlplane   NotReady

```

---

## 9. Worker Node Join 명령 생성

Control Plane에서 다음 명령을 실행합니다.

```bash

kubeadm token create --print-join-command

```

예시 출력:

```bash

kubeadm join <CONTROL-PLANE-IP>:6443 \

  --token <TOKEN> \

  --discovery-token-ca-cert-hash sha256:<HASH>

```

이 명령을 Worker Node에서 실행합니다.

```bash

sudo kubeadm join <CONTROL-PLANE-IP>:6443 \

  --token <TOKEN> \

  --discovery-token-ca-cert-hash sha256:<HASH>

```

> 실제 Token과 CA Hash 값은 환경마다 다르므로 README에는 직접 기록하지 않습니다.

---

## 10. Flannel CNI 다운로드

Flannel Manifest를 다운로드합니다.

```bash

curl -LO \

https://raw.githubusercontent.com/flannel-io/flannel/v0.20.2/Documentation/kube-flannel.yml

```

---

## 11. Flannel Pod CIDR 수정

이번 클러스터에서는 Pod CIDR을

```text

172.17.0.0/16

```

으로 설정했기 때문에 Flannel Manifest의 기본 Network 값을 동일하게 변경합니다.

`kube-flannel.yml`에서 다음 부분을 수정합니다.

```yaml

net-conf.json: |

  {

    "Network": "172.17.0.0/16",

    "Backend": {

      "Type": "vxlan"

    }

  }

```

필요한 경우 사용할 NIC도 지정합니다.

```yaml

args:

  - --ip-masq

  - --kube-subnet-mgr

  - --iface=eth0

```

### VXLAN

Flannel은 Node 간 Pod 트래픽을 전달하기 위해 VXLAN Overlay Network를 구성합니다.

```text

Node1 Pod

   │

   ▼

flannel.1

   │

 VXLAN

   │

   ▼

flannel.1

   │

   ▼

Node2 Pod

```

---

## 12. Flannel 적용

```bash

kubectl apply -f kube-flannel.yml

```

---

## 13. Pod 상태 확인

```bash

watch kubectl get pods -A

```

Flannel과 CoreDNS가 정상적으로 `Running` 상태가 되는지 확인합니다.

예시:

```text

NAMESPACE      NAME                          STATUS

kube-system    coredns-xxxxx                 Running

kube-system    kube-apiserver-controlplane   Running

kube-system    kube-controller-manager       Running

kube-system    kube-scheduler-controlplane   Running

kube-flannel   kube-flannel-ds-xxxxx         Running

```

Node 상태도 확인합니다.

```bash

kubectl get nodes -o wide

```

정상적인 경우:

```text

NAME           STATUS   ROLES           VERSION

controlplane   Ready    control-plane   v1.35.0

worker01       Ready    <none>          v1.35.0

worker02       Ready    <none>          v1.35.0

```

---

## Network Flow

```text

Pod A

  │

  ▼

veth

  │

  ▼

Node Network

  │

  ▼

Flannel VXLAN

  │

  ▼

Remote Node

  │

  ▼

veth

  │

  ▼

Pod B

```

---

## Verification Commands

```bash

kubectl get nodes -o wide

```

```bash

kubectl get pods -A

```

```bash

kubectl get svc -A

```

```bash

kubectl cluster-info

```

---

## Troubleshooting

### Node가 NotReady 상태인 경우

CNI Pod 상태를 확인합니다.

```bash

kubectl get pods -A

```

Flannel Log를 확인합니다.

```bash

kubectl logs -n kube-flannel <flannel-pod-name>

```

### kubeadm init 재구성 시 기존 설정 충돌

기존 클러스터를 초기화합니다.

```bash

sudo kubeadm reset -f

```

필요한 경우 기존 CNI 설정도 제거합니다.

```bash

sudo rm -rf /etc/cni/net.d

```

---

## Future Improvements

향후 아래 구성 요소를 추가하여 실제 운영 환경과 유사한 Kubernetes Platform으로 확장할 예정입니다.

- Kubernetes Control Plane HA

- HAProxy + Keepalived

- Calico / Cilium

- Ingress NGINX

- Helm

- Argo CD

- Jenkins CI/CD

- Prometheus

- Grafana

- Alertmanager

- NetworkPolicy

- HPA

- Persistent Volume
