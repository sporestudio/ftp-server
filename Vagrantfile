# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
    config.vm.box = "debian/bookworm64"
    config.vm.define "ftp-server" do |s|
      s.vm.network "private_network", ip: "192.168.57.20"
      s.vm.network "private_network", ip: "192.168.57.30"
      s.vm.hostname="server"
      s.vm.provision "ansible" do |ansible|
        ansible.config_file = "ansible/ansible.cfg"  
        ansible.playbook = "ansible/ftp.yml"
        ansible.inventory_path = "ansible/inventory.yml"
      end
    end
    config.vm.define "dns" do |d|
      d.vm.network "private_network", ip: "192.168.57.10"
      d.vm.hostname="dns"
      d.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/ns.yml" # playbooks may change after I actually get to see them...
        ansible.config_file = "ansible/ansible.cfg"
        ansible.inventory_path = "ansible/inventory.yml"
      end
    end
  end