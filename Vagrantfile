# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "amazonlinux2"

  # sync a folder on the host machine to the guest machine
  config.vm.synced_folder ".", "/vagrant"

  # time sync from host machine
  config.vm.provider :virtualbox do |vb|
    vb.customize ["setextradata", :id, "VBoxInternal/Devices/VMMDev/0/Config/GetHostTimeDisabled", 0]
  end

  # https://www.vagrantup.com/docs/provisioning/ansible_local.html
  config.vm.provision "ansible_local" do |ansible|
    ansible.install        = false
    ansible.playbook       = 'ansible/provision.yml'
    ansible.inventory_path = 'ansible/inventory'
    ansible.limit          = 'all'
    ansible.verbose        = true
  end

  # vagrant-vbguest plugin not support amazonlinux2
  #   The guest's platform ("amazon") is currently not supported, will try generic Linux method...
  #   This system is currently not set up to build kernel modules.
  #   Please install the Linux kernel "header" files matching the current kernel
  #   for adding new hardware support to the system.
  config.vbguest.installer = VagrantVbguest::Installers::RedHat

  config.vm.define "docker01", primary: true do |c|
    c.vm.hostname = 'docker01.maepachi.local'
    config.vm.network :public_network,
                      :bridge => "en0: Wi-Fi (AirPort)",
                      :ip => "172.16.100.10",
                      :netmask => "255.255.0.0"
  end
end
