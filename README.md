# Kubernetes Platform Lab

온프레미스 환경에서 Kubernetes 고가용성 클러스터를 직접 설계하고 구축한 프로젝트입니다.

XCP-ng 기반 가상화 환경에 3개의 Control Plane과 3개의 Worker Node를 구성하고,
HAProxy와 Keepalived를 사용하여 Kubernetes API Server의 고가용성을 구성했습니다.

Container Runtime은 containerd를 사용하며,
Pod Network는 Calico CNI를 구성했습니다.

Node의 공통 시스템 설정과 Kubernetes 구축에 필요한 사전 작업은
Ansible을 사용하여 자동화했습니다.

## Architecture

```text
                     Client
                       |
                       v
              192.168.10.115:6443
                 Kubernetes API VIP
                       |
            +----------+----------+
            |                     |
          lb01                  lb02
   HAProxy + Keepalived   HAProxy + Keepalived
   192.168.10.99          192.168.10.100
            |                     |
            +----------+----------+
                       |
          +------------+------------+
          |            |            |
        cp01         cp02         cp03
   192.168.10.101  192.168.10.102  192.168.10.103
   kube-apiserver   kube-apiserver   kube-apiserver
        etcd             etcd             etcd
          |                |                |
          +----------------+----------------+
                           |
          +----------------+----------------+
          |                |                |
      worker01          worker02          worker03
   192.168.10.111    192.168.10.112    192.168.10.113
```

## Environment

| Component | Configuration |
|---|---|
| Hypervisor | XCP-ng |
| OS | Rocky Linux 8.10 |
| Kubernetes | v1.34.12 |
| Container Runtime | containerd |
| CNI | Calico v3.32.2 |
| Control Plane | 3 Nodes |
| Worker | 3 Nodes |
| etcd | 3 Members (Stacked etcd) |
| API Load Balancer | HAProxy |
| Virtual IP | Keepalived |
| Automation | Ansible |
| Pod CIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/12 |

## Node Configuration

| Role | Hostname | IP Address |
|---|---|---|
| Load Balancer | lb01 | 192.168.10.99 |
| Load Balancer | lb02 | 192.168.10.100 |
| Control Plane | cp01 | 192.168.10.101 |
| Control Plane | cp02 | 192.168.10.102 |
| Control Plane | cp03 | 192.168.10.103 |
| Worker | worker01 | 192.168.10.111 |
| Worker | worker02 | 192.168.10.112 |
| Worker | worker03 | 192.168.10.113 |

Kubernetes API Server Virtual IP:

```text
192.168.10.115:6443
```

## Control Plane High Availability

Kubernetes API Server는 3개의 Control Plane Node에서 동작합니다.

Client와 Kubernetes Node는 개별 Control Plane에 직접 접근하지 않고
Keepalived가 관리하는 Virtual IP `192.168.10.115`를 통해 API Server에 접근합니다.

```text
kubectl / kubelet
       |
       v
192.168.10.115:6443
       |
       v
   Keepalived
       |
       v
    HAProxy
       |
       +---- cp01:6443
       +---- cp02:6443
       +---- cp03:6443
```

HAProxy는 3개의 Control Plane API Server로 트래픽을 전달하고,
Keepalived는 `lb01`, `lb02` 사이에서 Kubernetes API VIP를 관리합니다.

이를 통해 하나의 Load Balancer 또는 Control Plane Node에 장애가 발생하더라도
Kubernetes API에 지속적으로 접근할 수 있도록 구성했습니다.

## etcd

각 Control Plane Node에 etcd Member를 구성하여
3 Member Stacked etcd Cluster를 구성했습니다.

```text
cp01 ─┐
cp02 ─┼── etcd Cluster
cp03 ─┘
```

3개의 etcd Member를 사용하여 하나의 Member에 장애가 발생하더라도
quorum을 유지할 수 있도록 구성했습니다.

## Network

Pod Network는 Calico CNI를 사용합니다.

```text
Pod CIDR     : 10.244.0.0/16
Service CIDR : 10.96.0.0/12
```

Calico를 통해 Node 간 Pod Network를 구성하고
Kubernetes NetworkPolicy를 사용할 수 있도록 구성했습니다.

## Automation

Kubernetes Node의 반복적인 설정 및 관리를 위해 Ansible을 사용했습니다.
Control Plane과 Worker Node를 Inventory Group으로 관리하고,
Ad-hoc Command와 Playbook을 사용하여 공통 시스템 설정 및
반복 작업을 자동화했습니다.

주요 구성:
- Ansible Inventory 기반 Node Group 관리
- SSH Key 기반 Node 접근
- `ping`, `command`, `shell`을 이용한 상태 확인
- `copy`, `replace`, `service` Module을 이용한 일괄 설정
- Kubernetes Node 공통 설정 Playbook
자세한 Ansible 구성 및 사용 방법은 다음 문서에서 확인할 수 있습니다.
[Ansible Automation →](ansible/README.md)

## Cluster Build Flow

Kubernetes Cluster는 다음 순서로 구성했습니다.

```text
Node Preparation
       |
       v
Container Runtime
(containerd)
       |
       v
Kubernetes Packages
       |
       v
HAProxy / Keepalived
       |
       v
kubeadm init
       |
       v
Control Plane Join
       |
       v
Worker Join
       |
       v
Calico CNI
       |
       v
Cluster Validation
```

## Repository Structure

```text
kubernetes-platform-lab/
├── README.md
├── ansible/
│   ├── README.md
│   ├── inventory.ini
│   └── playbooks/
│       └── node-preparation.yml
└── docs/
    ├── 01-node-preparation.md
    ├── 02-kubeadm-cluster.md
    ├── 03-control-plane-ha.md
    ├── 04-calico-network.md
    ├── 05-cluster-validation.md
    └── troubleshooting.md
```

## Documentation

Kubernetes Cluster 구축 과정은 단계별 문서로 정리했습니다.

| Step | Document | Description |
|---|---|---|
| 01 | [Node Preparation](docs/01-node-preparation.md) | Kubernetes Node 사전 설정 |
| 02 | [kubeadm Cluster](docs/02-kubeadm-cluster.md) | kubeadm 기반 Kubernetes Cluster 구성 |
| 03 | [Control Plane HA](docs/03-control-plane-ha.md) | HAProxy / Keepalived 기반 Kubernetes API HA 구성 |
| 04 | [Calico Network](docs/04-calico-network.md) | Calico CNI 및 Pod Network 구성 |
| 05 | [Cluster Validation](docs/05-cluster-validation.md) | Cluster 및 HA 구성 최종 검증 |

구축 과정에서 발생한 주요 문제와 해결 과정은
[Troubleshooting](docs/troubleshooting.md)에 정리했습니다.

## Related Repositories

### Server Inventory API

[server-inventory-api](https://github.com/mattang-e/server-inventory-api)

FastAPI와 PostgreSQL을 사용하는 Server Inventory REST API의
Application Source 및 Container Image Build 구성을 관리합니다.

### Server Inventory Kubernetes

[server-inventory-k8s](https://github.com/mattang-e/server-inventory-k8s)

본 Kubernetes Platform 위에 Server Inventory API를 배포하고 운영하기 위한
Application Infrastructure 구성을 관리합니다.

Private Harbor Registry, Gateway API, Envoy Gateway, MetalLB,
PostgreSQL StatefulSet 및 NetApp NFS Storage를 사용하여 구성했습니다.

## Getting Started

실제 Kubernetes Cluster 구축 과정은 다음 문서부터 확인할 수 있습니다.

[01. Node Preparation →](docs/01-node-preparation.md)