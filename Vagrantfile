Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  
  # Global configuration for all VMs
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "1024"
    vb.cpus = 1
    vb.linked_clone = true
  end
  
  config.ssh.insert_key = false
  
  # VM configurations with NEW IP range (192.168.57.x)
  vm_configs = [
    {name: "host1", ip: "192.168.57.10", user: "user1"},
    {name: "host2", ip: "192.168.57.11", user: "user2"},
    {name: "host3", ip: "192.168.57.12", user: "user3"},
    {name: "host4", ip: "192.168.57.13", user: "user4"},
    {name: "host5", ip: "192.168.57.14", user: "user5"},
    {name: "host6", ip: "192.168.57.15", user: "user6"},
    {name: "host7", ip: "192.168.57.16", user: "user7"}
  ]
  
  vm_configs.each do |vm_config|
    config.vm.define vm_config[:name] do |host|
      host.vm.hostname = "ansible-#{vm_config[:name]}"
      host.vm.network "private_network", ip: vm_config[:ip]
      
      host.vm.provider "virtualbox" do |vb|
        vb.name = "ansible-#{vm_config[:name]}"
        if vm_config[:name] == "host5"
          vb.memory = "2048"
          vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"]
        end
      end
      
      host.vm.provision "shell", inline: <<-SHELL
        set -e
        
        echo "🔧 Configuring #{vm_config[:name]}..."
        
        apt-get update
        apt-get upgrade -y
        apt-get install -y curl wget git vim net-tools python3 python3-pip openssh-server sudo
        
        useradd -m -s /bin/bash #{vm_config[:user]}
        echo '#{vm_config[:user]}:password123' | chpasswd
        usermod -aG sudo #{vm_config[:user]}
        
        # Configure passwordless sudo
        echo '#{vm_config[:user]} ALL=(ALL) NOPASSWD:ALL' | tee /etc/sudoers.d/#{vm_config[:user]}
        chmod 0440 /etc/sudoers.d/#{vm_config[:user]}
        
        mkdir -p /home/#{vm_config[:user]}/.ssh
        chown #{vm_config[:user]}:#{vm_config[:user]} /home/#{vm_config[:user]}/.ssh
        chmod 700 /home/#{vm_config[:user]}/.ssh
        
        sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
        sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
        systemctl restart ssh
        
        if [ "#{vm_config[:name]}" = "host5" ]; then
          apt-get install -y virtualbox vagrant
          usermod -aG vboxusers #{vm_config[:user]}
        fi
        
        echo "✅ #{vm_config[:user]} configured on #{vm_config[:name]} (#{vm_config[:ip]})"
      SHELL
    end
  end
end
