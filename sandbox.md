# Sandbox avec virtualbox 7.0.24

```bash
mkdir -p $HOME/install-k8s && cd $HOME/install-k8s
```

```bash
wget https://download.virtualbox.org/virtualbox/7.0.24/VBoxGuestAdditions_7.0.24.iso
```

```bash
vim Vagrantfile
```

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vbguest.auto_update = false
  config.vbguest.no_remote = true
  config.vbguest.iso_path = "./VBoxGuestAdditions_7.0.24.iso"

  box_distro = ENV['BOX_DISTRO'] || "rockylinux/9"
  box_version = ENV['BOX_DISTRO_VERSION'] || "5.0.0"

  # General Vagrant VM configuration
  config.ssh.insert_key = false
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.box = box_distro
  config.vm.box_version = box_version
  
  SERVERS = [
    { hostname: "control-server1", ip: "192.168.56.6", vcpu: 2, mem: 2048 },
    { hostname: "control-server2", ip: "192.168.56.7", vcpu: 2, mem: 2048 },
    { hostname: "worker-server", ip: "192.168.56.8", vcpu: 4, mem: 4096 }
  ]

  SERVERS.each do |server|
    config.vm.define server[:hostname] do |srv|
      srv.vm.hostname = server[:hostname]
      srv.vm.network :private_network, ip: server[:ip]
      srv.vm.provider :virtualbox do |v|
        v.cpus = server[:vcpu]
        v.memory = server[:mem]
      end
    end
  end
end
```

Pour démarrer les machines en fonction des distributions rocky linux9, debian12, debian11, ubuntu24.04, ubuntu22.04, procédez comme suit :

- pour rockylinux9

```
vagrant up
```

> Note: par défaut pour rockylinux9, BOX_DISTRO="rockylinux/9" et BOX_DISTRO_VERSION="5.0.0" 

- pour ubuntu24.04

```
BOX_DISTRO="bento/ubuntu-24.04" BOX_DISTRO_VERSION="202502.21.0" vagrant up
```

- pour ubuntu22.04

```
BOX_DISTRO="bento/ubuntu-22.04" BOX_DISTRO_VERSION="202407.23.0" vagrant up
```

- pour debian12

```
BOX_DISTRO="generic/debian12" BOX_DISTRO_VERSION="4.3.12" vagrant up
```

- pour debian11

```
BOX_DISTRO="generic/debian11" BOX_DISTRO_VERSION="4.3.12" vagrant up
```
