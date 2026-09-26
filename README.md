# Kubernetes Platform Lab

온프레미스 환경에서 Kubernetes 고가용성 클러스터를 직접 설계하고 구축하는 프로젝트입니다.

## Architecture

                     192.168.10.115:6443
                         Virtual IP
                             │
                 ┌───────────┴───────────┐
                 │                       │
               lb01                    lb02
        HAProxy + Keepalived     HAProxy + Keepalived
                 │                       │
                 └───────────┬───────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
            cp01           cp02           cp03
       192.168.10.101  192.168.10.102  192.168.10.103
       kube-apiserver  kube-apiserver  kube-apiserver
           etcd           etcd           etcd
              │              │              │
              └──────────────┼──────────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
          worker01        worker02        worker03
       192.168.10.111  192.168.10.112  192.168.10.113

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
| etcd | 3 Members |
| API Load Balancer | HAProxy |
| VIP | Keepalived |
| Automation | Ansible |
| Pod CIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/12 |

