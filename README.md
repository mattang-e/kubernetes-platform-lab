# Kubernetes Platform Lab

kubeadm 기반으로 Kubernetes Cluster를 구축하고, 네트워크(CNI), Ingress, 모니터링, GitOps 등 실제 운영 환경에서 사용하는 기술을 단계적으로 구성하는 프로젝트입니다.

---

## Project Goals

- Kubernetes Cluster 구축

- Container Runtime 구성

- CNI 구성

- Ingress 구축

- Service Load Balancing

- Monitoring 구축

- GitOps 구축

- CI/CD 구축

---

## Technology Stack

| Category | Technology |

|----------|------------|

| OS | Ubuntu 24.04 |

| Container Runtime | containerd |

| Kubernetes | kubeadm v1.35 |

| CNI | Flannel |

| Package Manager | Helm |

| Ingress | NGINX Ingress |

| Load Balancer | MetalLB |

| Monitoring | Prometheus + Grafana |

| GitOps | Argo CD |

| CI/CD | Jenkins |

---

## Architecture

> (추후 Architecture Diagram 추가 예정)

```text

                    Internet

                        │

                Load Balancer

                        │

             Ingress Controller

                        │

                Service (ClusterIP)

                        │

                  kube-proxy

                        │

                       Pod

```

---

## Documents

| No | Topic | Status |

|----|-------|--------|

| 01 | Kubernetes Cluster 구축 (kubeadm) | ✅ |

| 02 | Flannel CNI | ✅ |

| 03 | Helm | 🚧 |

| 04 | NGINX Ingress | 🚧 |

| 05 | MetalLB | 🚧 |

| 06 | Prometheus | 🚧 |

| 07 | Grafana | 🚧 |

| 08 | Argo CD | 🚧 |

| 09 | Jenkins | 🚧 |

| 10 | NetworkPolicy | 🚧 |

---

## Project Structure

```text

kubernetes-platform-lab/

README.md

docs/

├── 01-kubeadm-install/

├── 02-flannel/

├── 03-helm/

├── 04-ingress-nginx/

├── 05-metallb/

├── 06-prometheus/

├── 07-grafana/

├── 08-argocd/

└── 09-jenkins/

```

---

## Learning Roadmap

- [x] Kubernetes Cluster 구축

- [x] Flannel CNI

- [ ] Helm

- [ ] Ingress NGINX

- [ ] MetalLB

- [ ] Prometheus

- [ ] Grafana

- [ ] Argo CD

- [ ] Jenkins

- [ ] HPA

- [ ] NetworkPolicy
