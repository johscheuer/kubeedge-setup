# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.define "cloudcore" do |cloudcore|
      cloudcore.vm.box = "ubuntu/bionic64"
      cloudcore.vm.hostname = "cloudcore"
      cloudcore.vm.synced_folder "manifests/", "/home/vagrant/manifests"
      cloudcore.vm.network "private_network", ip: "172.2.0.2"

      cloudcore.vm.provider "virtualbox" do |v|
        v.memory = 2042
      end
  end

  config.vm.define "edgenode" do |edgenode|
      edgenode.vm.box = "ubuntu/bionic64"
      edgenode.vm.hostname = "edgenode"
      edgenode.vm.synced_folder "manifests/edgecore", "/home/vagrant/manifests/edgecore"
      edgenode.vm.network "private_network", ip: "172.2.0.3"
  end
end
