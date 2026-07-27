Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/jammy64" # Ubuntu 22.04 LTS
  
    config.vm.hostname = "cloud1"
  
    # Private network so Ansible can reach it via a fixed IP
    config.vm.network "private_network", ip: "192.168.56.10"
  
    # Forward ports so you can hit the site from your Mac's browser too
    config.vm.network "forwarded_port", guest: 80, host: 8080
    config.vm.network "forwarded_port", guest: 443, host: 8443
  
    config.vm.provider "virtualbox" do |vb|
      vb.name = "cloud1-dev"
      vb.memory = 2048
      vb.cpus = 2
    end
  
    # Make sure python3 is present for Ansible (subject requires this anyway)
    config.vm.provision "shell", inline: <<-SHELL
      apt-get update
      apt-get install -y python3
    SHELL
  end