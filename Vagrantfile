# OS: Ubuntu 22.04 LTS

Vagrant.configure("2") do |config|
  
  # Master 
  config.vm.define "master" do |master|
    master.vm.hostname = "k8s-master"
    master.vm.box = "generic/ubuntu2204"
    master.vm.network "private_network", ip: "192.168.56.10"
    master.vm.provider :virtualbox do |vb|   
      vb.cpus = 4
      vb.memory = 4096
    end
  end

  # Worker 01
  config.vm.define "worker_01" do |worker_01|
    worker_01.vm.hostname = "k8s-worker-001"
    worker_01.vm.box = "generic/ubuntu2204"
    worker_01.vm.network "private_network", ip: "192.168.56.20"
    worker_01.vm.provider :virtualbox do |vb|   
      vb.cpus = 2
      vb.memory = 2048
    end
  end

  # Worker 02
  config.vm.define "worker_02" do |worker_02|
    worker_02.vm.hostname = "k8s-worker-002"
    worker_02.vm.box = "generic/ubuntu2204"
    worker_02.vm.network "private_network", ip: "192.168.56.21"
    worker_02.vm.provider :virtualbox do |vb|   
      vb.cpus = 2
      vb.memory = 2048
    end
  end
end