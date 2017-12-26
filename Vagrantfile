# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "centos7"
  config.vm.box_check_update = false
  config.ssh.forward_agent = true
  config.vm.network "public_network"

  config.vm.hostname = "centos7-docker"

  config.vm.provider :virtualbox do |vb|
    vb.customize ["modifyvm", :id, "--vram", "16"]
    vb.customize ["modifyvm", :id, "--cpuexecutioncap", "50", "--memory", "1024", "--cpus", "1" ]
    vb.name = "centos7-docker"
  end

  config.push.define "atlas" do |push|
     push.app = "nuaays/centos7-docker"
  end

  config.vm.provision "shell", inline: <<-SHELL
        sudo cp /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
  SHELL

  config.vm.provision "shell", privileged: true, path: "init.sh"
  #config.vm.provision "file", source: "./daemon.json", destination: "/etc/docker/daemon.json"

end
