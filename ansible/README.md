# Ansible Automation

Kubernetes Cluster를 구성하는 여러 Node의 반복적인 설정 및 관리를 위해
Ansible을 사용했습니다.

Mac을 Ansible Control Node로 사용하고,
Control Plane 및 Worker Node에 SSH로 접속하여 공통 설정과
반복 작업을 일괄적으로 수행했습니다.

## Architecture

```text
                 Mac
          Ansible Control Node
                  |
                  | SSH
                  |
        +---------+---------+
        |                   |
        v                   v
  control_plane           workers
        |                   |
   +----+----+         +----+----+
   |    |    |         |    |    |
 cp01 cp02 cp03    worker01 02   03
```

## Directory Structure

```text
ansible/
├── README.md
├── inventory.ini
└── playbooks/
    └── node-preparation.yml
```

- `inventory.ini` : Ansible Managed Node 및 Group 정의
- `playbooks/` : 반복적인 Node 설정을 자동화하는 Playbook
- `README.md` : Ansible 구성 및 사용 방법

## Inventory

Kubernetes Node를 역할별로 관리하기 위해
`inventory.ini`를 구성했습니다.

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

Inventory Group 구조는 다음과 같습니다.

```text
k8s
├── control_plane
│   ├── cp01
│   ├── cp02
│   └── cp03
│
└── workers
    ├── worker01
    ├── worker02
    └── worker03
```

`control_plane`과 `workers`를 각각 별도의 Group으로 관리하고,
`k8s:children`을 사용하여 전체 Kubernetes Node를 `k8s` Group으로 묶었습니다.

이를 통해 전체 Node 또는 특정 역할의 Node에 선택적으로
Ansible 작업을 수행할 수 있도록 구성했습니다.

## SSH Key Authentication

Ansible Control Node에서 Managed Node에 Password 입력 없이
접속할 수 있도록 SSH Key 기반 인증을 사용했습니다.

SSH Key 생성:

```bash
ssh-keygen -t ed25519
```

Public Key 배포:

```bash
ssh-copy-id root@192.168.10.101
ssh-copy-id root@192.168.10.102
ssh-copy-id root@192.168.10.103

ssh-copy-id root@192.168.10.111
ssh-copy-id root@192.168.10.112
ssh-copy-id root@192.168.10.113
```

Ansible은 SSH를 통해 Managed Node에 접속하고
Remote Node의 Python을 이용하여 Ansible Module을 실행합니다.

```text
Ansible Control Node
        |
        | SSH
        v
   Managed Node
        |
        | Python
        v
   Ansible Module
```

## Connection Test

Ansible `ping` Module을 사용하여 전체 Kubernetes Node의
연결 상태를 확인할 수 있습니다.

```bash
ansible k8s -i inventory.ini -m ping
```

Control Plane만 확인:

```bash
ansible control_plane -i inventory.ini -m ping
```

Worker Node만 확인:

```bash
ansible workers -i inventory.ini -m ping
```

Ansible의 `ping` Module은 ICMP Ping이 아니라
SSH 접속 및 Remote Python을 통해 Ansible Module이
정상적으로 실행되는지 확인합니다.

정상적인 경우 다음과 같이 `pong`을 반환합니다.

```text
cp01 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

## Ad-hoc Commands

일회성 작업이나 여러 Node의 상태를 빠르게 확인할 때
Ansible Ad-hoc Command를 사용했습니다.

기본 형식:

```bash
ansible <group> -i inventory.ini -m <module> -a "<arguments>"
```

### command

여러 Node에서 동일한 명령을 실행할 때 사용합니다.

Hostname 확인:

```bash
ansible k8s -i inventory.ini -m command \
  -a "hostname"
```

Kernel Version 확인:

```bash
ansible k8s -i inventory.ini -m command \
  -a "uname -r"
```

Kubelet Version 확인:

```bash
ansible k8s -i inventory.ini -m command \
  -a "kubelet --version"
```

### shell

Pipe, redirect 등 Shell 기능이 필요한 명령을 실행할 때 사용합니다.

Kubernetes Repository 확인:

```bash
ansible k8s -i inventory.ini -m shell \
  -a "grep baseurl /etc/yum.repos.d/kubernetes.repo"
```

Container Runtime 상태 확인:

```bash
ansible k8s -i inventory.ini -m shell \
  -a "systemctl is-active containerd"
```

### copy

동일한 설정 파일을 여러 Node에 배포할 때 사용합니다.

예를 들어 Kubernetes Network 관련 sysctl 설정 파일을
전체 Kubernetes Node에 배포할 수 있습니다.

```bash
ansible k8s -i inventory.ini -m copy \
  -a "src=files/k8s.conf dest=/etc/sysctl.d/k8s.conf owner=root group=root mode=0644"
```

배포 구조:

```text
Ansible Control Node
      |
      | k8s.conf
      |
      +-------------------------+
      |                         |
      v                         v
Control Plane                Workers
      |                         |
      v                         v
/etc/sysctl.d/k8s.conf    /etc/sysctl.d/k8s.conf
```

### replace

여러 Node의 기존 설정 파일에서 특정 값을 일괄적으로
변경할 때 사용합니다.

Kubernetes 1.35 Repository를 1.34 Repository로 변경하는 과정에서
다음 명령을 사용했습니다.

```bash
ansible k8s -i inventory.ini -m replace \
  -a "path=/etc/yum.repos.d/kubernetes.repo regexp='v1\.35/rpm' replace='v1.34/rpm'"
```

각 Node에 직접 접속하여 Repository 파일을 수정하지 않고
전체 Kubernetes Node의 설정을 한 번에 변경할 수 있습니다.

### service

여러 Node의 Service 상태를 일괄적으로 관리할 때 사용합니다.

Calico Network 문제를 확인하는 과정에서
Host Firewall의 영향을 확인하기 위해 다음 명령을 사용했습니다.

```bash
ansible k8s -i inventory.ini -m service \
  -a "name=firewalld state=stopped"
```

Service 상태 확인:

```bash
ansible k8s -i inventory.ini -m shell \
  -a "systemctl is-active firewalld"
```

이 과정에서 Host Firewall이 Calico Network에 영향을 주고 있음을
확인할 수 있었습니다.

## Ad-hoc Command vs Playbook

단순 상태 확인이나 일회성 작업은 Ad-hoc Command를 사용하고,
반복적으로 적용해야 하는 여러 설정은 Playbook으로 관리합니다.

```text
Ad-hoc Command
       |
       +-- ping
       +-- command
       +-- shell
       +-- copy
       +-- replace
       +-- service

Playbook
       |
       +-- 여러 Task 정의
       +-- 반복 가능한 설정
       +-- Node 설정 표준화
       +-- 자동화 코드 관리
```

## Node Preparation Playbook

Kubernetes Node의 공통 사전 설정은
`playbooks/node-preparation.yml`에 정의합니다.

주요 자동화 대상은 다음과 같습니다.

- Swap 비활성화
- SELinux 설정
- Kernel Module 설정
- Kubernetes Network sysctl 설정
- containerd 설정 및 Service 관리
- kubelet Service 관리

Playbook:

[Node Preparation Playbook](playbooks/node-preparation.yml)

실행:

```bash
ansible-playbook \
  -i inventory.ini \
  playbooks/node-preparation.yml
```

## Troubleshooting Automation

Ansible은 초기 구축뿐 아니라 Cluster 문제를 분석하는 과정에서도
사용했습니다.

대표적으로 다음 작업을 여러 Node에 동시에 수행했습니다.

```text
Kubernetes Repository Version 변경
        |
        +-- replace module

Host Firewall 일시 중지
        |
        +-- service module

Node 상태 확인
        |
        +-- command / shell module

Node 연결 확인
        |
        +-- ping module
```

이를 통해 여러 Kubernetes Node에 직접 SSH 접속하여
동일한 작업을 반복하는 대신 Ansible Control Node에서
일괄적으로 작업할 수 있도록 구성했습니다.

## Next Step

Kubernetes Node 공통 설정을 자동화하는 Playbook은
다음 파일에서 확인할 수 있습니다.

[Node Preparation Playbook →](playbooks/node-preparation.yml)