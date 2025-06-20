# Configurable worker count
WORKER_COUNT = ENV['WORKER_COUNT'] ? ENV['WORKER_COUNT'].to_i : 1

Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-24.04"

  # Common provision for all nodes
  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "ansible/general.yaml"
    ansible.compatibility_mode = "2.0"
    ansible.extra_vars = {
      worker_count: WORKER_COUNT
    }
  end

  # Control Node
  config.vm.define "ctrl" do |ctrl|
    ctrl.vm.hostname = "ctrl"
    ctrl.vm.network "private_network", ip: "192.168.56.100"
    
    # Control node basic setup
    ctrl.vm.provider "virtualbox" do |vb|
      # VM memory settings
      vb.memory = 2048
      vb.cpus = 2
      vb.name = "k8s-controller"
      # Uncomment if encounter startup issues
      vb.customize ["modifyvm", :id, "--usb", "off"]
      vb.customize ["modifyvm", :id, "--usbehci", "off"]
    end

    # Control node ansible provision
    ctrl.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/ctrl.yaml"
        ansible.compatibility_mode = "2.0"
    end
  end

#   # Worker Nodes
  (1..WORKER_COUNT).each do |i|
    config.vm.define "node-#{i}" do |node|
      node.vm.hostname = "node-#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{i + 100}"

      # Worker node basic setup
      node.vm.provider "virtualbox" do |vb|
        # VM memory settings
        vb.memory = 4096
        vb.cpus = 4
        vb.name = "k8s-worker-#{i}"
        # Uncomment if encounter startup issues
        vb.customize ["modifyvm", :id, "--usb", "off"]
        vb.customize ["modifyvm", :id, "--usbehci", "off"]
      end

      # Worker node ansible provision
      node.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/node.yaml"
        ansible.compatibility_mode = "2.0"
      end
    end
  end
  # Generate ansible inventory after all VMs are up
  config.trigger.after :up do |trigger|
    trigger.name = "Generate ansible inventory config file"
    trigger.ruby do
      File.open('inventory.cfg', 'w') do |f|

        # active controller node
        f.puts "[ctrl]"
        f.puts "ctrl=92.168.56.100"

        # active worker nodes (loop through each one)
        f.puts "\n[nodes]"
        (1..WORKER_COUNT).each do |i|
          f.puts "node-#{i}=192.168.56.#{i + 100}"
        end
      end
    end
  end
end