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
### 사전 준비

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
4. iptables 세팅
리눅스 커널 기능인 iptables, netfilter 기능을 통해 k8s service의 네트워크 기능이 이루어짐,
kube-proxy 컴포넌트는 service로 들어온 요청을 처리하기 위해 리눅스 커널의 iptables로 네트워크 경로를 관리하고, netfilter는 커널단의 부하분산 장치로 역할을 하며 iptables를 참고하여 실질적인 패킷 전달을 하며 부하분산을 수행한다.
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 필요한 sysctl 파라미터를 설정하면, 재부팅 후에도 값이 유지된다.
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 재부팅하지 않고 sysctl 파라미터 적용하기
sudo sysctl --system
```

### 설치

**Container Runtime Interface와 cgroup 이해하기**

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

    3. Istall CNI Plugins
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

2. 컨테이너 런타임(containerd)의 cgroup driver를 systemd 로 변경
cgroup: 프로세스에게 할당하는 리소스를 제한함 (cpu/mem/disk/net) 
시스템의 안정성을 위해 host node / kubelet / container runtime 모두 같은 cgroup 드라이버를 사용해야함
host <-> kubelet/cri의 cgroup 드라이버가 다르다면, 시스템에는 2개의 cgroup 관리자가 있는 것이기 때문에, cgorup을 사용한 자원 관리에 혼동이 발생할 수 있고, 이로인해 시스템에 자원 압박이 발생할 위험이 존재한다.

또한, host 시스템에서 cgroup v2를 사용한다면, cgroup driver는 systemd를 사용하는것이 권장사항이다.

결국 지금 사용하는 노드의 OS는 Ubuntu22.04로 cgorup version이 cgroupv2 이며, init 시스템도 systemd를 사용하기 때문에, host, kubelet, container rumtime의 cgroup driver를 systemd 로 설정하기로 하였다.

host에서 어떤 cgroup 버전을 사용하는지 아래 명령어로 확인 가능하다.
```
# Identify the cgroup version on Linux Nodes 
stat -fc %T /sys/fs/cgroup/
```
cgroup v2를 사용하기 위한 requirements는 다음과 같고, 현재 구축할 시스템은 요건을 충족한다.
```
OS enabled cgroup v2
-> ubuntu 22.04 는 default로 cgroup v2를 사용함

Linux kernel version is 5.8 or later
-> 5.15 이상의 커널 버전 확인
root@k8s-master:/home/vagrant# uname -a
Linux k8s-master 5.15.0-91-generic #101-Ubuntu SMP Tue Nov 14 13:30:08 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux

container rumtime supports cgorup v2
(containerd v1.4 and later)
-> containerd의 버전은 v2.0.0 으로 조건 충족

kubelet and container runtime are configured to use the systemd cgroup driver
-> 설치 과정에서 kubelet과 container runtime의 cgroup dirver를 systemd로 맞춰줄 예정이다.
```

https://kubernetes.io/ko/docs/setup/production-environment/container-runtimes/#containerd
https://tech.kakao.com/posts/394

    ```bash
    # 설정파일 생성
    sudo mkdir /etc/containerd
    sudo containerd config default | sudo tee /etc/containerd/config.toml
    
    # 아래처럼 수정 config.toml
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    ...
    [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
        SystemdCgroup = true
    
    # 재시작
    sudo systemctl restart containerd
    ```

**CNI 이해하기**
-> containerd 설치할때 함께 설치하는 cni plugin과 weave, calico 같은 cni는 무슨 관계인지?
-> kubelet이 관리할 시절에는 /opt/cni/bin (바이너리 경로), /etc/cni/net.d/ (설정 경로)를 통해서 cni관리를 했고, (kubelet option --cni-bin-dir 통해서) 특정 버전 이후부터 그 방식이 아닌지?
https://kubernetes.io/ko/docs/concepts/cluster-administration/networking/
https://github.com/containernetworking/cni
https://kubernetes.io/ko/docs/concepts/services-networking/

3. kubeadm / kubelet / kubectl 설치
4. kubelet의 cgroup driver를 systemd로 변경
https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/configure-cgroup-driver/
5. kubeadm 으로 클러스터 부트스트랩
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/
6. CNI Add on 설치
https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network

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