# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.network "private_network", ip: "192.168.57.10"
  config.vm.provision "shell", inline: <<-SHELL
    # Update package list and install bind9 and bind9-utils
    apt-get update && apt-get upgrade -y
    apt-get install -y bind9 bind9-utils

    # Copy all documents from /vagrant to /home/vagrant
    cp -v /vagrant/named /etc/default/named
    cp -v /vagrant/named.conf.local /etc/bind/named.conf.local
    cp -v /vagrant/named.conf.options /etc/bind/named.conf.options
    cp -v /vagrant/db.sri.ies /var/lib/bind/
    cp -v /vagrant/rev.sri.ies /var/lib/bind/
    
    # Restart bind9 service
    systemctl restart bind9
  SHELL
end
