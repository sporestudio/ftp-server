# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
    config.vm.box = "debian/bookworm64"
    config.ssh.insert_key = false
    config.vm.define "ftp" do |s|
      s.vm.network "private_network", ip: "192.168.57.20"
      s.vm.network "private_network", ip: "192.168.57.30"
      s.vm.hostname="ftp"
      s.vm.provision "ansible" do |ansible|
        ansible.config_file = "./ansible.cfg"  
        ansible.playbook = "ansible/ftp.yml"
        ansible.inventory_path = "ansible/inventory.yml"
      end
    end
    config.vm.define "dns" do |d|
      d.vm.network "private_network", ip: "192.168.57.10"
      d.vm.hostname="dns"
      d.vm.provision "ansible" do |ansible|
        ansible.playbook = "ansible/ns.yml"
        ansible.config_file = "./ansible.cfg"
        ansible.inventory_path = "ansible/inventory.yml"
      end
    end
  end