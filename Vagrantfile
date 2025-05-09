# Configurable worker count
WORKER_COUNT = ENV['WORKER_COUNT'] ? ENV['WORKER_COUNT'].to_i : 2

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"

  # Control Node
  config.vm.define "ctrl" do |ctrl|
    ctrl.vm.hostname = "ctrl"
    ctrl.vm.network "private_network", ip: "192.168.56.100"
    
    ctrl.vm.provider "virtualbox" do |vb|
      # vb.memory = 4096
      vb.memory = 1024
      vb.cpus = 1
      vb.name = "k8s-controller"
    end
  end

  # Worker Nodes
  (1..WORKER_COUNT).each do |i|
    config.vm.define "node-#{i}" do |node|
      node.vm.hostname = "node-#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{i + 100}"
      
      node.vm.provider "virtualbox" do |vb|
        # vb.memory = 6144
        vb.memory = 2048
        vb.cpus = 2
        vb.name = "k8s-worker-#{i}"
      end
    end
  end
end

# use `newgrp docker``