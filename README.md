# vagrant-centos7-docker
```bash
centos7
docker17.09.1
```

## make a box based on centos7.box
```bash
vagrant login
vagrant init nuaays/centos7-docker

vagrant up
vagrant halt
vagrant package ( vagrant package --output=centos7-docker.box )

mv package.box centos7-docker.box
vagrant box add --name centos7-docker centos7-docker.box
```

## Vagrantfile
```ruby
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
```


## init script
```bash
sudo cp /usr/share/zoneinfo/Asia/Shanghai  /etc/localtime
sudo curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
sudo curl -o /etc/yum.repos.d/docker-ce.repo  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config

#### Install Tools
sudo yum update
sudo yum install -y vim wget curl ntp screen git etcd ebtables flannel socat net-tools.x86_64 iperf bridge-utils.x86_64 grub2.x86_64 yum-utils device-mapper-persistent-data lvm2 libdevmapper* grub2.x86_64
sudo yum install -y libtool-ltdl  selinux-policy  policycoreutils policycoreutils-python libseccomp

####
systemctl start ntpd
systemctl enable ntpd
systemctl disable firewalld.service
systemctl stop firewalld.service
systemctl disable iptables.service
systemctl stop iptables.service
iptables -P INPUT ACCEPT
iptables -P OUTPUT ACCEPT
iptables -P FORWARD ACCEPT

#### Setting for K8S
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
net.ipv4.ip_forward = 1
vm.swappiness=0
EOF
sudo sysctl --system

##### Upgrade Linux Kernel
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
rpm -Uvh https://mirrors.tuna.tsinghua.edu.cn/elrepo/kernel/el7/x86_64/RPMS/elrepo-release-7.0-3.el7.elrepo.noarch.rpm
yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
yum --enablerepo=elrepo-kernel install -y kernel-ml kernel-ml-devel #yum --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel
awk -F\' '$1=="menuentry " {print i++ " @ " $2}' /etc/grub2.cfg
grub2-set-default 0
rpm -qa | grep kernel

##### Install Docker
wget https://mirrors.aliyun.com/docker-ce/linux/centos/7/x86_64/stable/Packages/docker-ce-17.09.1.ce-1.el7.centos.x86_64.rpm
yum localinstall -y docker-ce-17.09.1.ce-1.el7.centos.x86_64.rpm
sudo usermod -aG docker vagrant

#### Setting Docker
cat <<EOF >/etc/sysconfig/docker
OPTIONS="-H unix:///var/run/docker.sock -H tcp://0.0.0.0:2375 --storage-driver=overlay --exec-opt native.cgroupdriver=cgroupfs --graph=/localdisk/docker/graph --insecure-registry=gcr.io --insecure-registry=quay.io --insecure-registry=registry.cn-hangzhou.aliyuncs.com --insecure-registry=senyang.com:5000 --insecure-registry=192.168.56.101:5000 --registry-mirror=https://ip2y4bft.mirror.aliyuncs.com --registry-mirror=http://138f94c6.m.daocloud.io"
EOF

mkdir /etc/docker/
cat <<EOF > /etc/docker/daemon.json
{
    "hosts": [
        "tcp://0.0.0.0:2375",
        "unix:///var/run/docker.sock"
     ],
    "debug": true,
    "log-driver": "json-file",
    "log-level": "debug",
    "experimental": true,
    "metrics-addr": "0.0.0.0:1337",
    "selinux-enabled": false,
    "registry-mirrors": [
        "http://138f94c6.m.daocloud.io",
        "https://olzwzeg2.mirror.aliyuncs.com",
        "https://ip2y4bft.mirror.aliyuncs.com"
    ],
    "insecure-registries":[
        "gcr.io",
        "quay.io",
        "registry.cn-hangzhou.aliyuncs.com",
        "senyang.com:5000"
    ],
    "exec-opts": [
        "native.cgroupdriver=cgroupfs"
    ],
    "graph": "/localdisk/docker/graph",
    "storage-driver": "overlay2",
    "storage-opts": [ "overlay2.override_kernel_check=true" ],
    "live-restore": true
}
EOF

sudo systemctl enable docker
reboot
```
