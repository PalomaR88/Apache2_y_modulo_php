# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define :serapache do |serapache|
    serapache.vm.box = "debian/buster64"
    serapache.vm.hostname = "servidor"
    serapache.vm.network :public_network,:bridge=>"enp2s0"
  end
end
