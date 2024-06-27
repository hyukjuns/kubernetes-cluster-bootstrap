# Kubernetes 1.30.1 Installation by Vagrant and Kubeadm

# 설치 환경 및 사전 준비
## 설치 환경 (호스트 PC)
- Ubuntu Desktop 22.04 LTS (8core, 16GB Mem)
- VirtualBox 6.1.50
- Vagrant 2.4.1
- Vbox Network Adapter
    - NAT Adapter: VM 을 외부와 통신하게 함 (호스트PC와 VM간 통신은 불가)
    - Host-Only Adapter: 호스트PC와 VM 사이의 통신을 가능하게함 (VM과 외부 통신은 불가)
- Vbox Image
    - Ubuntu 22.04 LTS (cgroupV2 를 기본값으로 지원)
## 설치할 클러스터 정보
- Kubernetes Version: 1.30.1
- 1 Master Node, 1 Worker Node
- Container Network Interface: Weave
- Container Runtime Interface: Containrd
- Container Storage Interface: None

## 사전준비

### 1. Vagrant로 VM 생성

[Vagrantfile](./Vagrantfile) 을 사용해서 마스터 1대, 워커 1대 생성

```bash
vagrant up
```
### 2. 생성된 VM의 시스템 고유 정보 확인

hostname, mac 주소, nic id 고유한지 확인

```bash
hostname
ip link
sudo cat /sys/class/dmi/id/product_uuid
```

# 설치 순서

## 1. UFW 방화벽 OFF

```bash
sudo ufw disable
```

## 2. SWAP 메모리 해제

```bash
sudo swapoff -a
sudo sed -i '/swap/ s/^/#/' /etc/fstab # swap 행을 주석처리
```

## 3. Container Runtime 설치 (containerd)
    
[K8s 공식 가이드 문서](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

[containerd 공식 설치 문서](https://github.com/containerd/containerd/blob/main/docs/getting-started.md#getting-started-with-containerd)

> containerd와 runc의 관계
리눅스 커널 기능인 iptables, netfilter 기능을 통해 k8s service의 네트워크 기능이 이루어짐,
kube-proxy 컴포넌트는 service로 들어온 요청을 처리하기 위해 리눅스 커널의 iptables로 네트워크 경로를 관리하고, netfilter는 커널단의 부하분산 장치로 역할을 하며 iptables를 참고하여 실질적인 패킷 전달을 하며 부하분산을 수행한다.
    
```bash

## 1. sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system

# Verify
sysctl net.ipv4.ip_forward

## 2.  Install Containerd
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

## 3. Install RunC
# Download Files
wget https://github.com/opencontainers/runc/releases/download/v1.2.0-rc.1/runc.amd64
wget https://github.com/opencontainers/runc/releases/download/v1.2.0-rc.1/runc.sha256sum

# Check sha256sum
sha256sum runc.amd64
cat runc.sha256sum

# Install runc 
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

## 4. Istall CNI Plugins
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

## 4. 컨테이너 런타임(containerd)의 cgroup driver를 systemd 로 변경
설치 과정에서 container runtime의 cgroup dirver를 systemd로 맞춰줄 예정이다. kubelet은 v1.22 or later 부터 cgroupdriver가 기본적으로 systemd 이다.

> cgroup 이해
cgroup: 프로세스에게 할당하는 리소스를 제한함 (cpu/mem/disk/net) 
시스템의 안정성을 위해 host node / kubelet / container runtime 모두 같은 cgroup 드라이버를 사용해야함
host <-> kubelet/cri의 cgroup 드라이버가 다르다면, 시스템에는 2개의 cgroup 관리자가 있는 것이기 때문에, cgorup을 사용한 자원 관리에 혼동이 발생할 수 있고, 이로인해 시스템에 자원 압박이 발생할 위험이 존재한다.
또한, host 시스템에서 cgroup v2를 사용한다면, cgroup driver는 systemd를 사용하는것이 권장사항이다.
결국 지금 사용하는 노드의 OS는 Ubuntu22.04로 cgorup version이 cgroupv2 이며, init 시스템도 systemd를 사용하기 때문에, host, kubelet, container rumtime의 cgroup driver를 systemd 로 설정하기로 하였다.
host에서 어떤 cgroup 버전을 사용하는지 아래 명령어로 확인 가능하다.

```bash
# Identify the cgroup version on Linux Nodes 
stat -fc %T /sys/fs/cgroup/
```
cgroupv2를 사용하는 시스템은 다음 링크 참고

[Linux Distribution cgroup v2 support](https://kubernetes.io/docs/concepts/architecture/cgroups/#linux-distribution-cgroup-v2-support)

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


## 5. kubeadm / kubelet / kubectl 설치

https://kubernetes.io/ko/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

```
# Debian based distro
sudo apt-get update
# apt-transport-https may be a dummy package; if so, you can skip that package
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

# If the directory `/etc/apt/keyrings` does not exist, it should be created before the curl command, read the note below.
# sudo mkdir -p -m 755 /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
Note:

# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

#Install
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

#Enable kubelet
sudo systemctl enable --now kubelet

```
*kubelet의 cgroup driver를 systemd로 변경할 필요 없음, kubelet의 default cgroup driver는 systemd 이다 (v1.22 or later)

## 6. kubeadm 으로 클러스터 부트스트랩

[공식문서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

```bash

## 마스터 노드 부트스트랩
kubeadm init --apiserver-advertise-address 192.168.56.10 --pod-network-cidr=192.168.0.0/16

# 옵션 설명
--apiserver-advertise-address 는 마스터 노드 1개일 경우 사용해도 되지만 마스터 노드가 여러개일 경우 --control-plane-endpoint 을 대신 사용해서 공용 엔드포인트로 사용할 수 있다. 이는 마스터 노드 앞단에 부하분산 장치를 사용한 HA 클러스터 구성에 사용 가능하다.


## 출력 저장
# kubeconfig 저장
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# 워커노드 조인에 사용할 커맨드 저장
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>

## 조인 커맨드에 사용할 token이 24시간 지나서 없어졋거나 잊어버렸을 경우
# 토큰 확인
kubeadm token list
# 토큰 재생성
kubeadm token create
# 토큰 해시값 확인
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'

```

## 7. CNI Add-On 설치 (Weave)
>CNI 이해하기
containerd 설치할때 함께 설치하는 cni plugin과 weave, calico 같은 cni는 무슨 관계인지?
kubelet이 관리할 시절에는 /opt/cni/bin (바이너리 경로), /etc/cni/net.d/ (설정 경로)를 통해서 cni관리를 했고, (kubelet option --cni-bin-dir 통해서) 특정 버전 이후부터 그 방식이 아닌지?

[공식문서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network)

[Weave 설치](https://github.com/rajch/weave#using-weave-on-kubernetes)

```bash
kubectl apply -f https://reweave.azurewebsites.net/k8s/v1.29/net.yaml
```


## 8. 워커 노드 조인
1~5번 수행 후 조인 커맨드로 클러스터 조인
```
# 커맨드 잊어버렸을 경우 6번 참고하여 커맨드 찾기
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

# Ref

k8s 문서는 영어로 봐야 정확하다.


## 설치 순서
[부트스트랩 관련 문서 목차](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/)

[1. 설치시작문서](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

[2. 컨테이너 런타임 설치](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

[3. 클러스터 생성](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)

## 기타 참고 문서
### containerd
- https://github.com/containerd/containerd/blob/main/docs/getting-started.md#getting-started-with-containerd
### cgroup
- https://kubernetes.io/docs/concepts/architecture/cgroups/#check-cgroup-version
- https://tech.kakao.com/posts/394
### CNI
- https://kubernetes.io/ko/docs/concepts/cluster-administration/networking/
- https://github.com/containernetworking/cni
- https://kubernetes.io/ko/docs/concepts/services-networking/
### vagrant 
- https://app.vagrantup.com/generic/boxes/ubuntu2204

### virtualbox
- https://www.nakivo.com/blog/virtualbox-network-setting-guide/