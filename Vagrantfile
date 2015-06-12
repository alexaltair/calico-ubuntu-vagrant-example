# -*- mode: ruby -*-
# vi: set ft=ruby :

$num_instances = 2
$instance_name_prefix = "ubuntu"

# All Vagrant configuration is done below. The "2" in Vagrant.configure
# configures the configuration version (we support older styles for
# backwards compatibility). Please don't change it unless you know what
# you're doing.
Vagrant.configure(2) do |config|
  # The most common configuration options are documented and commented below.
  # For a complete reference, please see the online documentation at
  # https://docs.vagrantup.com.

  # Every Vagrant development environment requires a box. You can search for
  # boxes at https://atlas.hashicorp.com/search.
  config.vm.box = "phusion/ubuntu-14.04-amd64"

  # Disable automatic box update checking. If you disable this, then
  # boxes will only be checked for updates when the user runs
  # `vagrant box outdated`. This is not recommended.
  # config.vm.box_check_update = false

  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 80 on the guest machine.
  # config.vm.network "forwarded_port", guest: 80, host: 8080

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"

  # Provider-specific configuration so you can fine-tune various
  # backing providers for Vagrant. These expose provider-specific options.
  # Example for VirtualBox:
  #
  # config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
  #   vb.memory = "1024"
  # end
  #
  # View the documentation for the provider you are using for more
  # information on available options.

  # Enable provisioning with a shell script. Additional provisioners such as
  # Puppet, Chef, Ansible, Salt, and Docker are also available. Please see the
  # documentation for more information about their specific syntax and use.
  # config.vm.provision "shell", inline: <<-SHELL
  #   sudo apt-get update
  #   sudo apt-get install -y apache2
  # SHELL

  (1..$num_instances).each do |i|
    vm_name = "%s-%02d" % [$instance_name_prefix, i]
    config.vm.define vm_name do |host|
      host.vm.hostname = vm_name

      ip = "172.17.8.#{i+100}"
      host.vm.network :private_network, ip: ip

      # download calico and preload the docker images.
      host.vm.provision :shell, inline: "wget -q https://circle-artifacts.com/gh/Metaswitch/calico-docker/718/artifacts/0"\
                                        "/home/ubuntu/calico-docker/dist/calicoctl"
      host.vm.provision :shell, inline: "chmod +x calicoctl"
      host.vm.provision :shell, inline: "sudo modprobe ip6_tables"
      host.vm.provision :shell, inline: "sudo modprobe xt_set"
      host.vm.provision :shell, inline: "sudo sysctl -w net.ipv6.conf.all.forwarding=1"
      host.vm.provision :docker, images: [
          "busybox:latest",
          "calico/node:libnetwork",
          "quay.io/coreos/etcd:v2.0.11",
      ]

      # Replace docker with an unreleased version.
      filename = "docker"
      host.vm.provision :shell, inline: "wget https://transfer.sh/PzCye/#{filename} > /dev/null 2>&1"
      # host.vm.provision :shell, inline: "wget https://master.dockerproject.org/linux/amd64/docker-1.7.0-dev > /dev/null"
      host.vm.provision :shell, inline: "chmod +x #{filename}"
      host.vm.provision :shell, inline: "sudo stop docker"
      host.vm.provision :shell, inline: "sudo cp #{filename} $(which docker)"
      host.vm.provision :shell, inline: "sudo start docker"

      if i == 1
        host.vm.provision :docker do |d|
          d.run "quay.io/coreos/etcd",
            args: "-p 2379:2379 -p 2380:2380",
            cmd: "--name calico "\
             "--advertise-client-urls http://#{ip}:2379 "\
             "--listen-client-urls http://0.0.0.0:2379 "\
             "--initial-advertise-peer-urls http://#{ip}:2380 "\
             "--listen-peer-urls http://0.0.0.0:2380 "\
             "--initial-cluster-token etcd-cluster-2 "\
             "--initial-cluster calico=http://#{ip}:2380 "\
             "--initial-cluster-state new"
        end
      end
    end
  end
end
