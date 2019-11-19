# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "cloudcore" do |cloudcore|
      cloudcore.vm.box = "ubuntu/bionic64"
      cloudcore.vm.hostname = "cloudcore"
      cloudcore.vm.synced_folder "manifests/", "/home/vagrant/manifests"
      cloudcore.vm.network "private_network", ip: "172.2.0.2"
  end

  config.vm.define "edgenode" do |edgenode|
      edgenode.vm.box = "ubuntu/bionic64"
      edgenode.vm.hostname = "edgenode"
      edgenode.vm.synced_folder "manifests/edgecore", "/home/vagrant/manifests/edgecore"
      edgenode.vm.network "private_network", ip: "172.2.0.3"
  end

  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   apt-get update
  #   apt-get install -y apache2
  # SHELL
end
