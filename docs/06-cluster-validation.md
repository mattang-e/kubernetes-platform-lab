# Cluster Validation

## Overview

Kubernetes HA Cluster 구축 완료 후 실제 Workload를 배포하여
Cluster의 주요 기능이 정상적으로 동작하는지 검증하였다.

검증 항목은 다음과 같다.

- Node 상태
- Control Plane 상태
- Pod Scheduling
- Container Runtime
- Pod Network
- Service Network
- CoreDNS
- Kubernetes API Endpoint

---

## 1. Node Status

Cluster에 Join된 모든 Node의 상태를 확인하였다.

```bash
kubectl get nodes
```

실행 결과:

```text
NAME       STATUS   ROLES           VERSION
cp01       Ready    control-plane   v1.34.12
cp02       Ready    control-plane   v1.34.12
cp03       Ready    control-plane   v1.34.12
worker01   Ready    <none>          v1.34.12
worker02   Ready    <none>          v1.34.12
worker03   Ready    <none>          v1.34.12
```

3개의 Control Plane과 3개의 Worker Node가 모두
`Ready` 상태인 것을 확인하였다.

---

## 2. Control Plane Status

Control Plane Component 상태를 확인하였다.

```bash
kubectl get pods -n kube-system -o wide
```
```bash
NAME                           READY   STATUS    RESTARTS   AGE   IP               NODE       NOMINATED NODE   READINESS GATES
coredns-66bc5c9577-492bz       1/1     Running   0          13h   10.244.5.2       worker01   <none>           <none>
coredns-66bc5c9577-dmj6v       1/1     Running   0          13h   10.244.30.66     worker02   <none>           <none>
etcd-cp01                      1/1     Running   0          13h   192.168.10.101   cp01       <none>           <none>
etcd-cp02                      1/1     Running   0          13h   192.168.10.102   cp02       <none>           <none>
etcd-cp03                      1/1     Running   0          13h   192.168.10.103   cp03       <none>           <none>
kube-apiserver-cp01            1/1     Running   0          13h   192.168.10.101   cp01       <none>           <none>
kube-apiserver-cp02            1/1     Running   0          13h   192.168.10.102   cp02       <none>           <none>
kube-apiserver-cp03            1/1     Running   0          13h   192.168.10.103   cp03       <none>           <none>
kube-controller-manager-cp01   1/1     Running   0          13h   192.168.10.101   cp01       <none>           <none>
kube-controller-manager-cp02   1/1     Running   0          13h   192.168.10.102   cp02       <none>           <none>
kube-controller-manager-cp03   1/1     Running   0          13h   192.168.10.103   cp03       <none>           <none>
kube-proxy-7wlcc               1/1     Running   0          13h   192.168.10.101   cp01       <none>           <none>
kube-proxy-lldm2               1/1     Running   0          14h   192.168.10.113   worker03   <none>           <none>
kube-proxy-mm57r               1/1     Running   0          14h   192.168.10.111   worker01   <none>           <none>
kube-proxy-n4clt               1/1     Running   0          13h   192.168.10.102   cp02       <none>           <none>
kube-proxy-rczlh               1/1     Running   0          14h   192.168.10.112   worker02   <none>           <none>
kube-proxy-zpb56               1/1     Running   0          13h   192.168.10.103   cp03       <none>           <none>
kube-scheduler-cp01            1/1     Running   0          13h   192.168.10.101   cp01       <none>           <none>
kube-scheduler-cp02            1/1     Running   0          13h   192.168.10.102   cp02       <none>           <none>
kube-scheduler-cp03            1/1     Running   0          13h   192.168.10.103   cp03       <none>           <none>
```

각 Control Plane Node에서 다음 Component가 동작한다.

```text
kube-apiserver
kube-controller-manager
kube-scheduler
etcd
```

Control Plane은 3개의 Node로 구성되어 있으며
etcd 또한 3-member Cluster로 구성하였다.

```text
cp01 ─ etcd
cp02 ─ etcd
cp03 ─ etcd
```

---

## 3. etcd Cluster

etcd member 상태를 확인하였다.

```bash
kubectl exec -n kube-system etcd-cp02 -c etcd -- \
etcdctl member list \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
--key=/etc/kubernetes/pki/etcd/healthcheck-client.key
```

3개의 Control Plane Node가 etcd member로 등록되어 있는 것을 확인하였다.

```text
f7073d50389b7d8c, started, cp01, https://192.168.10.101:2380, https://192.168.10.101:2379, false
1b319b64b31833f2, started, cp02, https://192.168.10.102:2380, https://192.168.10.102:2379, false
4abb4d712c4d1d81, started, cp03, https://192.168.10.103:2380, https://192.168.10.103:2379, false
```

이를 통해 3-member stacked etcd Cluster 구성을 확인하였다.

---

## 4. Test Deployment

실제 Workload Scheduling을 검증하기 위해
nginx Deployment를 생성하였다.

```bash
kubectl create deployment nginx-test \
  --image=nginx:latest \
  --replicas=3
```

Deployment 상태 확인:

```bash
kubectl get deployment nginx-test
```
```bash
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
nginx-test   3/3     3            3           45s
```

Pod 상태 및 배치 Node 확인:

```bash
kubectl get pods -o wide
```
```bash
NAME                          READY   STATUS    RESTARTS   AGE   IP             NODE       NOMINATED NODE   READINESS GATES
nginx-test-786f7585db-nxbns   1/1     Running   0          80s   10.244.5.5     worker01   <none>           <none>
nginx-test-786f7585db-bqs6w   1/1     Running   0          80s   10.244.30.71   worker02   <none>           <none>
nginx-test-786f7585db-hck25   1/1     Running   0          80s   10.244.19.69   worker03   <none>           <none>
```

3개의 nginx Pod가 정상적으로 생성되고
Worker Node에서 실행되는 것을 확인하였다.

이 테스트를 통해 다음 기능을 확인하였다.

---

## 5. ClusterIP Service

Deployment에 접근하기 위한 ClusterIP Service를 생성하였다.

```bash
kubectl expose deployment nginx-test \
  --name=nginx-test-svc \
  --port=80 \
  --target-port=80 \
  --type=ClusterIP
```

Service 확인:

```bash
kubectl get svc nginx-test-svc
```
```bash
NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
nginx-test-svc   ClusterIP   10.96.235.37   <none>        80/TCP    19s
```

Endpoint 확인:

```bash
kubectl get endpoints nginx-test-svc
```
```bash
NAME             ENDPOINTS                                       AGE
nginx-test-svc   10.244.19.69:80,10.244.30.71:80,10.244.5.5:80   53s
```

Service가 nginx Pod를 Endpoint로 정상적으로 연결하고 있는지 확인하였다.

---

## 6. CoreDNS and Service Communication

Cluster 내부 DNS와 Service Network를 동시에 검증하였다.

테스트용 Pod를 생성하여 Service Name으로 nginx에 접근하였다.

```bash
kubectl run curl-test \
  --image=curlimages/curl \
  --restart=Never \
  --rm -it \
  -- curl http://nginx-test-svc
```
```text
<h1>Welcome to nginx!</h1>
```
nginx Welcome Page가 정상적으로 반환되는 것을 확인하였다.

요청 흐름:

```text
curl-test Pod
      │
      │ http://nginx-test-svc
      ▼
    CoreDNS
      │
      ▼
ClusterIP Service
      │
      ▼
 nginx Pod
```

이를 통해 다음 기능이 정상적으로 동작하는 것을 확인하였다.

```text
CoreDNS
Service Discovery
ClusterIP
Pod Network
Service → Pod Routing
```

---

## 7. Kubernetes API Endpoint

Kubernetes API가 Virtual IP를 통해 정상적으로 접근되는지 확인하였다.

```bash
kubectl cluster-info
```

Kubernetes API Endpoint:

```text
Kubernetes control plane is running at https://192.168.10.115:6443
```

Client는 개별 Control Plane Node가 아닌
HAProxy / Keepalived로 구성된 VIP를 통해 API Server에 접근한다.

---

## 8. Calico Status
Calico Node 상태를 확인하였다.

```bash
kubectl get pods -n calico-system \
  -l k8s-app=calico-node
```

모든 Kubernetes Node의 `calico-node`가 `1/1 Running` 상태인 것을 확인하였다.

상세한 Calico 구성 및 검증은
[Calico Network](05-calico-network.md)를 참고한다.

---

## 9. Final Validation

최종적으로 다음 항목이 정상적으로 동작하는 것을 확인하였다.

| Component | Status |
|---|---|
| Control Plane | 정상 |
| Worker Node | 정상 |
| etcd 3-member | 정상 |
| API Virtual IP | 정상 |
| HAProxy | 정상 |
| Keepalived | 정상 |
| Calico | 정상 |
| Pod Scheduling | 정상 |
| Container Runtime | 정상 |
| CoreDNS | 정상 |
| ClusterIP Service | 정상 |
| Pod-to-Pod Network | 정상 |

Cluster 기본 구성 및 Network 검증을 완료하였다.

---

## 10. Test Resource Cleanup

검증에 사용한 Resource를 삭제하였다.

```bash
kubectl delete deployment nginx-test
kubectl delete service nginx-test-svc
```

테스트 Resource가 삭제되었는지 확인하였다.

```bash
kubectl get deployment nginx-test
kubectl get service nginx-test-svc
```

---

## Next Step

Cluster 구축 및 기본 기능 검증 과정에서 발생한 문제와
원인 분석 및 해결 과정을 별도의 Troubleshooting 문서에 정리하였다.

→ [Troubleshooting](troubleshooting.md)
