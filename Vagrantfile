# -*- mode: ruby -*-
# # # vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

server_ip="10.20.90.60"
server_vnc_port=5910

workers = [
  { :hostname => "worker1", :ip => "10.20.90.61", :vnc_port => 5911 },
  { :hostname => "worker2", :ip => "10.20.90.62", :vnc_port => 5912 },
  { :hostname => "worker3", :ip => "10.20.90.63", :vnc_port => 5913 }
]

$write_server_config=<<SCRIPT
cat <<EOF > /etc/rancher/rke2/config.yaml
token: helloworld
EOF
SCRIPT

$write_worker_config=<<SCRIPT
cat <<EOF > /etc/rancher/rke2/config.yaml
server: https://${1}:9345
token: helloworld
EOF
SCRIPT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.provider :libvirt do |libvirt|
    libvirt.driver = "kvm"
    libvirt.graphics_type = 'vnc'
  end

  config.vm.box = "generic/ubuntu1804"
  config.vm.box_version = "3.4.2"

  config.vm.provision "shell", privileged: true, inline: <<-SHELL
    set -e -x -u
    export DEBIAN_FRONTEND=noninteractive

    apt-get update -y
    apt-get install -y git vim curl openssh-server
    apt-get install -y jq open-iscsi nfs-common
  SHELL

  # Server node
  config.vm.define "server" do |node|
    node.vm.hostname = "server"
    node.vm.network "public_network",
      :dev => "br0",
      :mode => "bridge",
      :type => "bridge",
      :ip => "#{server_ip}"
    node.vm.provider "libvirt" do |v|
      v.cpus = 4
      v.memory = 4096
      v.graphics_ip = '0.0.0.0'
      v.graphics_port = "#{server_vnc_port}"
      v.graphics_passwd = '1234'
    end
    node.vm.provision "shell", privileged: true, inline: "curl -sfL https://get.rke2.io | sh -"
    node.vm.provision "shell", privileged: true, inline: "mkdir -p /etc/rancher/rke2"
    node.vm.provision "shell", privileged: true, inline: $write_server_config
    node.vm.provision "shell", privileged: true, inline: "systemctl enable rke2-server.service"
    node.vm.provision "shell", privileged: true, inline: "systemctl start rke2-server.service"
    node.vm.provision "shell", privileged: true, inline: "sh -c 'echo export KUBECONFIG=/etc/rancher/rke2/rke2.yaml >> /root/.bashrc'"
    node.vm.provision "shell", privileged: true, inline: "sh -c 'echo export PATH=$PATH:/var/lib/rancher/rke2/bin >> /root/.bashrc'"
  end

  # Worker nodes
  workers.each do |worker|
    config.vm.define worker[:hostname] do |node|
      node.vm.hostname = worker[:hostname]
      node.vm.network "public_network",
        :dev => "br0",
        :mode => "bridge",
        :type => "bridge",
        :ip => worker[:ip]
      node.vm.provider "libvirt" do |v|
        v.cpus = 4
        v.memory = 4096
        v.graphics_ip = '0.0.0.0'
        v.graphics_port = worker[:vnc_port]
        v.graphics_passwd = '1234'
      end
      node.vm.provision "shell", privileged: true, inline: "curl -sfL https://get.rke2.io | INSTALL_RKE2_TYPE=\"agent\" sh -"
      node.vm.provision "shell", privileged: true, inline: "systemctl enable rke2-agent.service"
      node.vm.provision "shell", privileged: true, inline: "mkdir -p /etc/rancher/rke2"
      node.vm.provision "shell", privileged: true, inline: $write_worker_config, args: "#{server_ip}"
      #node.vm.provision "shell", privileged: true, inline: "systemctl start rke2-agent.service"
    end
  end
end
