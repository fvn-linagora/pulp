# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "centos/7"

  config.vm.network "forwarded_port", guest: 8080, host: 8080 
  # config.vm.network "public_network"

  config.vm.provider "virtualbox" do |vb|
     # Customize the amount of memory on the VM:
     vb.memory = "2500"
  end

  #config.vm.provision "ansible_local" do |ansible|
  #  ansible.playbook = "provisioning/playbook.yml"
  #  ansible.sudo = true
  #  ansible.verbose = true
  #end

  # config.vm.provision "shell", inline: <<-SHELL
  #  apt update
  #  apt-get install -y apache2
  # SHELL
end
