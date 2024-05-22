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

    ```bash
    hostname
    ip link
    sudo cat /sys/class/dmi/id/product_uuid
    ```

2. UFW 방화벽 OFF

    ```bash
    sudo ufw disable
    ```

3. SWAP OFF
    ```bash
    sudo swapoff -a
    sudo sed -i '/swap/ s/^/#/' /etc/fstab # swap 행을 주석처리
    ```

설치

1. Container Runtime 설치 - containerd
containerd와 runc의 관계 이해 필요
[official guide](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#getting-started-with-containerd)

    ```
    1. Install Containerd
    # Download Files
    wget https://github.com/containerd/containerd/releases/download/v2.0.0-rc.2/containerd-2.0.0-rc.2-linux-amd64.tar.gz
    wget https://github.com/containerd/containerd/releases/download/v2.0.0-rc.2/containerd-2.0.0-rc.2-linux-amd64.tar.gz.sha256sum
    
    # Check sha256sum
    cat containerd-2.0.0-rc.2-linux-amd64.tar.gz.sha256sum
    sha256sum containerd-2.0.0-rc.2-linux-amd64.tar.gz
    
    # Install containerd (dir= /usr/local)
    sudo tar Cxzvf /usr/local containerd-2.0.0-rc.2-linux-amd64.tar.gz
    
    # Enroll systemd unit
    wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
    sudo mv containerd.service /etc/systemd/system/
    sudo systemctl daemon-reload
    sudo systemctl enable --now containerd

    2. Install RunC
    # Download Files
    wget https://github.com/opencontainers/runc/releases/download/v1.2.0-rc.1/runc.amd64
    wget https://github.com/opencontainers/runc/releases/download/v1.2.0-rc.1/runc.sha256sum

    # Check sha256sum
    sha256sum runc.amd64
    cat runc.sha256sum

    # Install runc 
    sudo install -m 755 runc.amd64 /usr/local/sbin/runc

    3. Istall CNIlugins
    # Download Files
    wget https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
    wget https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz.sha256

    # Check sha256sum
    sha256sum cni-plugins-linux-amd64-v1.5.0.tgz
    cat cni-plugins-linux-amd64-v1.5.0.tgz.sha256

    # Install CNI Plugin
    sudo mkdir -p /opt/cni/bin
    sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
    ```

2. cgroup driver를 systemd 로 변경

cgroup driver에 대한 필요성과 이해 필요

https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/#containerd

    ```
    sudo mkdir /etc/containerd
    sudo containerd config default > /etc/containerd/config.toml
    - 아래처럼 수정 config.toml
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    ...
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true
    sudo systemctl restart containerd
    ```

3. kubeadm 설치
4. cni 설치
5. kubeadm 으로 클러스터 설치

### 설치문서
https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

### 컨테이너 런타임
https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/

## Ref
- cgroup v2
  
    https://kubernetes.io/docs/concepts/architecture/cgroups/#check-cgroup-version

- vagrant cloud (os search)

    https://app.vagrantup.com/generic/boxes/ubuntu2204

- virtualbox network info

    https://www.nakivo.com/blog/virtualbox-network-setting-guide/