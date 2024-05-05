# Kubernetes Install by Vagrant and Kubeadm

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

## Auto Install Scripts

## Ref
- cgroup v2
  
    https://kubernetes.io/docs/concepts/architecture/cgroups/#check-cgroup-version

- vagrant cloud (os search)

    https://app.vagrantup.com/generic/boxes/ubuntu2204

- virtualbox network info

    https://www.nakivo.com/blog/virtualbox-network-setting-guide/