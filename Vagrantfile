# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/focal64"
  config.vm.box_check_update = false

  # Common configuration for all VMs
  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = "1024"
    vb.cpus = 1
  end

  # Site1 hosts (4 hosts)
  (1..4).each do |i|
    config.vm.define "site1-host#{i}" do |node|
      node.vm.hostname = "site1-host#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{10+i}"
      
      # Create user for each host
      node.vm.provision "shell", inline: <<-SHELL
        useradd -m -s /bin/bash user#{i}
        echo "user#{i}:password#{i}" | chpasswd
        usermod -aG sudo user#{i}
        
        # Setup SSH key for user
        mkdir -p /home/user#{i}/.ssh
        cp /home/vagrant/.ssh/authorized_keys /home/user#{i}/.ssh/
        chown -R user#{i}:user#{i} /home/user#{i}/.ssh
        chmod 700 /home/user#{i}/.ssh
        chmod 600 /home/user#{i}/.ssh/authorized_keys
        
        # Allow passwordless sudo
        echo "user#{i} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
      SHELL
    end
  end

  # Site2 hosts (3 hosts)
  (5..7).each do |i|
    config.vm.define "site2-host#{i}" do |node|
      node.vm.hostname = "site2-host#{i}"
      node.vm.network "private_network", ip: "192.168.56.#{10+i}"
      
      # Special configuration for first site2 host (for nested VM)
      if i == 5
        node.vm.provider "virtualbox" do |vb|
          vb.memory = "2048"
          vb.cpus = 2
          # Enable nested virtualization
          vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
        end
      end
      
      # Create user for each host
      node.vm.provision "shell", inline: <<-SHELL
        useradd -m -s /bin/bash user#{i}
        echo "user#{i}:password#{i}" | chpasswd
        usermod -aG sudo user#{i}
        
        # Setup SSH key for user
        mkdir -p /home/user#{i}/.ssh
        cp /home/vagrant/.ssh/authorized_keys /home/user#{i}/.ssh/
        chown -R user#{i}:user#{i} /home/user#{i}/.ssh
        chmod 700 /home/user#{i}/.ssh
        chmod 600 /home/user#{i}/.ssh/authorized_keys
        
        # Allow passwordless sudo
        echo "user#{i} ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
      SHELL
    end
  end
end
