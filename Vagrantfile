Vagrant.configure("2") do |config|
  config.vm.define "master" do |master|
    master.vm.box = "generic/centos7"
    master.vm.box_version = "3.6.10" # centos 7.9 based
    master.vm.network "public_network", :bridge => 'en0'
    master.vm.provider :virtualbox do |vb|   
      vb.cpus = 2
      vb.memory = 3096
    end
  end

  config.vm.define "worker" do |worker|
    worker.vm.box = "generic/centos7"
    worker.vm.box_version = "3.6.10" # centos 7.9 based
    worker.vm.network "public_network", :bridge => 'en0'
    worker.vm.provider :virtualbox do |vb|   
      vb.cpus = 2
      vb.memory = 2048
    end
  end
end