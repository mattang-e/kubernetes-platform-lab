# Helm

## 1. Helm이란?

Helm은 Kubernetes의 Package Manager로, 여러 개의 Kubernetes 리소스를 Chart라는 형태로 패키징하여 설치, 업그레이드, 롤백 및 삭제를 쉽게 관리할 수 있도록 해주는 도구입니다.

Linux의 apt, yum과 같은 역할을 Kubernetes에서 수행합니다.

대표적으로 다음과 같은 오픈소스를 Helm으로 설치합니다.

- Prometheus

- Grafana

- Argo CD

- Jenkins

- Harbor

- Ingress NGINX

- cert-manager

---

## 2. Helm Architecture

```

                Artifact Hub

                      │

                      ▼

                 Repository

                      │

                 Download Chart

                      │

                      ▼

              Chart + values.yaml

                      │

                 Template Rendering

                      │

                      ▼

            Kubernetes Manifest(YAML)

                      │

                      ▼

                 kube-apiserver

                      │

                      ▼

      Deployment / Service / ConfigMap / Pod

          (Release 정보는 Secret에 저장)

```

### Component

| Component | Description |

|----------|-------------|

| Artifact Hub | Chart 검색 사이트 |

| Repository | Chart 저장소 |

| Chart | Kubernetes Resource Package |

| Release | 설치된 애플리케이션 |

| Revision | Release 변경 이력 |

---

## 3. Helm 내부 동작

Helm은 Chart 자체를 Kubernetes에 저장하지 않습니다.

Chart와 values.yaml을 이용하여 Manifest를 생성(Rendering)한 후 API Server로 전달합니다.

```

Chart

+

values.yaml

↓

Template Rendering

↓

Manifest

↓

API Server

↓

Deployment

Service

Ingress

ConfigMap

Secret

```

Helm은 설치 시 Release 정보를 Kubernetes Secret(ConfigMap)에 저장합니다.

이를 이용하여 Upgrade와 Rollback을 수행합니다.

---

## 4. Helm 설치

Control Plane Node에서 설치

```bash

curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

chmod 700 get_helm.sh

./get_helm.sh

```

확인

```bash

helm version

```

---

## 5. Debug Mode

환경변수 확인

```bash

echo $HELM_DEBUG

```

Debug 실행

```bash

helm install myapp bitnami/nginx --debug

```

Debug Mode에서는

- Chart Loading

- values.yaml 적용 결과

- Rendered Manifest

- Kubernetes API 요청

- Error Stack

등을 확인할 수 있습니다.

---

## 6. Artifact Hub와 Repository

### Artifact Hub

Chart를 검색하는 사이트입니다.

예)

```

apache

↓

bitnami/apache

```

### Repository

실제 Chart가 저장되어 있는 저장소입니다.

등록

```bash

helm repo add bitnami https://charts.bitnami.com/bitnami

```

확인

```bash

helm repo list

```

업데이트

```bash

helm repo update

```

검색

```bash

helm search repo apache

```

Chart 정보

```bash

helm show chart bitnami/apache

```

기본 values 확인

```bash

helm show values bitnami/apache

```

---

## 7. Chart 설치

```bash

helm install amaze-surf bitnami/apache

```

```

amaze-surf

```

↓

Release Name

```

bitnami/apache

```

↓

Repository / Chart

Release 이름은 사용자가 자유롭게 지정할 수 있습니다.

설치 확인

```bash

helm list

```

---

## 8. values.yaml

Chart에는 기본 설정이 존재합니다.

예)

```yaml

replicaCount: 1

service:

  type: ClusterIP

image:

  tag: 1.24

```

values.yaml을 수정하여 원하는 설정으로 설치할 수 있습니다.

```bash

helm install nginx bitnami/nginx \

-f values.yaml

```

현재 적용된 설정

```bash

helm get values RELEASE_NAME

```

---

## 9. Upgrade

Repository 최신 정보 가져오기

```bash

helm repo update

```

Chart 정보 확인

```bash

helm show chart bitnami/nginx

```

Upgrade

```bash

helm upgrade dazzling-web bitnami/nginx --version 18.3.6

```

### Chart Version과 App Version

```

Chart Version

18.3.6

```

↓

Helm Chart 버전

```

App Version

1.24

```

↓

실제 Container Image 버전

Chart Version이 변경되어도 App Version이 동일하면 이미지 버전은 변경되지 않을 수 있습니다.

사용자가 values.yaml에서 image.tag를 지정한 경우 해당 버전이 우선 적용됩니다.

---

## 10. Rollback

Release 이력 확인

```bash

helm history dazzling-web

```

예)

```

REVISION

1

2

3

```

Revision 2로 복구

```bash

helm rollback dazzling-web 2

```

Rollback 후에는 Revision이 감소하는 것이 아니라 새로운 Revision이 생성됩니다.

```

Revision 1

↓

Revision 2

↓

Revision 3

↓

Revision 4 (Revision 2 상태)

```

상태 확인

```bash

helm status dazzling-web

```

---

## 11. Release

Release는 Helm이 설치한 애플리케이션 인스턴스입니다.

```

Repository

↓

Chart

↓

Release

↓

Revision

```

동일 Namespace에서는 Release 이름을 중복하여 사용할 수 없습니다.

```

helm install web bitnami/apache

```

다시

```

helm install web bitnami/nginx

```

를 실행하면

```

cannot re-use a name that is still in use

```

오류가 발생합니다.

---

## 12. Frequently Used Commands

```bash

helm version

helm env

helm repo list

helm repo update

helm search repo nginx

helm show chart bitnami/nginx

helm show values bitnami/nginx

helm list

helm status RELEASE

helm history RELEASE

helm get values RELEASE

helm install RELEASE CHART

helm upgrade RELEASE CHART

helm rollback RELEASE REVISION

helm uninstall RELEASE

```

---

## 13. Summary

- Helm은 Kubernetes Package Manager이다.

- Chart를 이용하여 여러 Kubernetes Resource를 한 번에 배포한다.

- Chart를 Rendering하여 Manifest를 생성한 후 API Server로 전달한다.

- Release 정보는 Kubernetes Secret에 저장된다.

- Upgrade 시 Revision이 증가한다.

- Rollback은
