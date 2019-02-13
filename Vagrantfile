# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "swarm_manager", primary: true do |swarm_manager|
    swarm_manager.vm.box = "debian/contrib-stretch64"
    swarm_manager.vm.hostname = 'swarm-manager'
    swarm_manager.vm.box_check_update = false
    swarm_manager.vm.synced_folder ".", "/vagrant_data"

    swarm_manager.vm.network :private_network, ip: "192.168.56.101"

    swarm_manager.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--memory", 512]
      v.customize ["modifyvm", :id, "--name", "swarm_manager"]
    end

    swarm_manager.vm.provision "shell", inline: <<-SHELL
        # curl
        apt install -y curl

        # docker
        apt-get update
        apt-get install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
        curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
        add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
        apt-get update
        apt-get install -y docker-ce=18.06.1~ce~3-0~debian
        usermod -aG docker vagrant

        # docker-compose
        curl -L https://github.com/docker/compose/releases/download/1.23.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
        chmod +x /usr/local/bin/docker-compose
    SHELL
  end

  config.vm.define "swarm_worker" do |swarm_worker|
    swarm_worker.vm.box = "debian/contrib-stretch64"
    swarm_worker.vm.hostname = 'swarm-worker'
    swarm_worker.vm.box_check_update = false
    swarm_worker.vm.synced_folder ".", "/vagrant_data"

    swarm_worker.vm.network :private_network, ip: "192.168.56.102"

    swarm_worker.vm.provider :virtualbox do |v|
      v.customize ["modifyvm", :id, "--memory", 512]
      v.customize ["modifyvm", :id, "--name", "swarm_worker"]
    end

    swarm_worker.vm.provision "shell", inline: <<-SHELL
        # curl
        apt install -y curl

        # docker
        apt-get update
        apt-get install -y apt-transport-https ca-certificates curl gnupg2 software-properties-common
        curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
        add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/debian $(lsb_release -cs) stable"
        apt-get update
        apt-get install -y docker-ce=18.06.1~ce~3-0~debian
        usermod -aG docker vagrant

        # docker-compose
        curl -L https://github.com/docker/compose/releases/download/1.23.1/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
        chmod +x /usr/local/bin/docker-compose
    SHELL
  end
end
