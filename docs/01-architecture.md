# Cluster Architecture

## Overview

단일 Control Plane 구성이 가지는 장애 지점을 줄이고
Kubernetes Control Plane의 고가용성 구조를 학습하기 위해
3개의 Control Plane Node로 클러스터를 구성하였다.

## Architecture
```text

                     Client
                       │
                       ▼
              192.168.10.115:6443
                     VIP
                       │
            ┌──────────┴──────────┐
            │                     │
          lb01                  lb02
     HAProxy/Keepalived    HAProxy/Keepalived
            │                     │
            └──────────┬──────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
        cp01         cp02         cp03
     API Server   API Server   API Server
       etcd          etcd          etcd
          │            │            │
          └────────────┼────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
      worker01      worker02      worker03
```

## Node Configuration

| Role | Hostname | IP |
|---|---|---|
| Load Balancer | lb01 | 192.168.10.99 |
| Load Balancer | lb02 | 192.168.10.100 |
| Control Plane | cp01 | 192.168.10.101 |
| Control Plane | cp02 | 192.168.10.102 |
| Control Plane | cp03 | 192.168.10.103 |
| Worker | worker01 | 192.168.10.111 |
| Worker | worker02 | 192.168.10.112 |
| Worker | worker03 | 192.168.10.113 |

API Virtual IP
```shell
    192.168.10.115
```
Pod CIDR
```shell
    10.244.0.0/16
```
Service CIDR
```shell
    10.96.0.0/12
```

## Control Plane HA

Kubernetes API Server는 3개의 Control Plane Node에서 동작한다.

클라이언트는 개별 Control Plane Node에 직접 접근하지 않고
Keepalived에서 관리하는 Virtual IP를 통해 API Server에 접근한다.
```text
    kubectl
       │
       ▼
    VIP :6443
       │
       ▼
    HAProxy
       │
       ├── cp01:6443
       ├── cp02:6443
       └── cp03:6443
```

## etcd

각 Control Plane Node에 etcd member를 구성하여
3-member stacked etcd cluster를 구성하였다.
```text
    cp01 ─┐
    cp02 ─┼── etcd cluster
    cp03 ─┘
```
3개의 member를 사용함으로써 하나의 etcd member 장애 상황에서도
quorum을 유지할 수 있도록 구성하였다.


## Network

Pod Network는 Calico를 사용하였다.
```shell
    Pod CIDR : 10.244.0.0/16
```
각 Node의 Pod 간 통신 및 Kubernetes NetworkPolicy를
지원할 수 있도록 Calico CNI를 적용하였다.
