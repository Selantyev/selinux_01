# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos/7"

#  config.vm.provision "ansible" do |ansible|
    #ansible.verbose = "vvv"
    #ansible.playbook = "provisioning/playbook.yml"
    #ansible.become = "true"
#  end

  config.vm.provider "virtualbox" do |v|
	  v.memory = 512
  end

  config.vm.define "ns01" do |ns01|
    ns01.vm.network "private_network", ip: "192.168.50.10", virtualbox__intnet: "dns"
    ns01.vm.hostname = "ns01"
  end

  config.vm.define "client" do |client|
    client.vm.network "private_network", ip: "192.168.50.15", virtualbox__intnet: "dns"
    client.vm.hostname = "client"
  end

  config.vm.define "ansible" do |ansible|
    ansible.vm.network "private_network", ip: "192.168.50.20", virtualbox__intnet: "dns"
    ansible.vm.hostname = "ansible"
  end
  
end
