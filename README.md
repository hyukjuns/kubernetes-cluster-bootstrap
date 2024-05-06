# Kubernetes Install by Vagrant and Kubeadm

## vagrant command
- vagrant up / destroy / halt

## To Make Cluster Info
- 1 Master Node
- 2 Worker Node
- Container Network Interface: Calico
- Container Runtime Interface: Containrd
- Container Storage Interface: ???
- Addons -> Prometheus Stack, Ingress Nginx Controller

## Environments & Prerequisite
### HOST PC
- LG Gram Notebook 8 core, 16 GB Memory
- Ubuntu 22.04 LTS
- VirtualBox 6.1.50
- Vagrant 2.4.1
### Virtual Box Setting
- Network
    - NAT : VM을 외부와 통신하게 함 (호스트 <-> VM 간 통신 X)
    - Host-Only : 호스트와 VM 간 통신을 가능하게함 (VM <-> 외부 통신 X)

### Virtual Nodes
- OS: Ubuntu 22.04 LTS (cgroup v2 support)


## Manual Install Step
사전 준비
1. hostname, mac 주소, nic id 고유한지 확인
2. UFW 방화벽 OFF
3. SELINUX OFF
4. SWAP OFF
설치
1. Container Runtime 설치 - containerd
2. cgroup driver를 systemd 로 변경
3. kubeadm 설치
4. cni 설치
5. kubeadm 으로 클러스터 설치

### 설치문서
https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### 컨테이너 런타임
https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/

## Auto Install Scripts

## Ref
- cgroup v2
  
    https://kubernetes.io/docs/concepts/architecture/cgroups/#check-cgroup-version

- vagrant cloud (os search)

    https://app.vagrantup.com/generic/boxes/ubuntu2204

- virtualbox network info

    https://www.nakivo.com/blog/virtualbox-network-setting-guide/