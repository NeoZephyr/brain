```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.require_version ">= 1.6.0"
boxes = [
  {
    :name => "node01",
    :eth1 => "192.168.10.1",
    :mem => "1024",
    :cpu => "1",
    :port => "8888"
  },
  {
    :name => "docker-node2",
    :eth1 => "192.168.10.2",
    :mem => "1024",
    :cpu => "1",
    :port => "9999"
  }
]

Vagrant.configure("2") do |config|

  config.vm.box = "centos/7"

  config.vm.box_check_update = false

  # config.vm.network "forwarded_port", guest: 80, host: 8080
  # config.vm.network "forwarded_port", guest: 80, host: 8080, host_ip: "127.0.0.1"

  # config.vm.network "private_network", ip: "192.168.33.10"

  # config.vm.network "public_network"

  boxes.each do |box|
    config.vm.define box[:name] do |config|
      config.vm.hostname = box[:name]
      # config.vm.network "forwarded_port", guest: 80, host: box[:port]

      config.vm.provider "vmware_fusion" do |v|
        v.vmx["memsize"] = box[:mem]
        v.vmx["numvcpus"] = box[:cpu]
      end
      config.vm.provider "virtualbox" do |v|
        v.customize ["modifyvm", :id, "--memory", box[:mem]]
        v.customize ["modifyvm", :id, "--cpus", box[:cpu]]
      end

      config.vm.network :private_network, ip: box[:eth1]
    end
  end

  # config.vm.synced_folder "./data", "/home/vagrant/data"

  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end

  # config.vm.provision "shell", privileged: true, path: "./setup.sh"
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
end
```