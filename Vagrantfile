# OS: Ubuntu 22.04 LTS

Vagrant.configure("2") do |config|
  
  config.vm.define "master" do |master|
    master.vm.hostname = "k8s-master"
    master.vm.box = "generic/ubuntu2204"
    master.vm.network "private_network", ip: "192.168.56.20"
    master.vm.provider :virtualbox do |vb|   
      vb.cpus = 4
      vb.memory = 4096
    end
  end

  config.vm.define "worker" do |worker_01|
    worker_01.vm.hostname = "k8s-worker-001"
    worker_01.vm.network "private_network", ip: "192.168.56.20"
    worker_01.vm.provider :virtualbox do |vb|   
      vb.cpus = 2
      vb.memory = 2048
    end
  end
end