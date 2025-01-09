# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"
  config.vm.define "fpt-server" do |s|
    s.vm.network "private_network", ip: "192.168.57.20"
    s.vm.network "private_network", ip: "192.168.57.30"
    s.vm.hostname="server"
  end
  config.vm.define "dns" do |d|
    d.vm.network "private_network", ip: "192.168.57.10"
    d.vm.hostname="dns"
  end
end
