# Sandbox avec virtualbox 7.0

```
mkdir $HOME/install-k8s && cd $HOME/install-k8s
```

```
wget https://download.virtualbox.org/virtualbox/7.0.20/VBoxGuestAdditions_7.0.20.iso
```

- **Vagrantfile avec Rocky linux 8.10**

```
vim Vagrantfile-with-rocky
```

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vbguest.auto_update = false
  config.vbguest.no_remote = true
  config.vbguest.iso_path = "./VBoxGuestAdditions_7.0.20.iso"

  # General Vagrant VM configuration
  config.ssh.insert_key = false
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.provider :virtualbox do |v|
    v.memory = 2048
    v.cpus = 2
    v.linked_clone = true
  end
  
  # Rocky Server
  config.vm.define "control-server" do |srv|
    srv.vm.box = "willbrid/rockylinux8"
    srv.vm.box_version = "0.0.3"
    srv.vm.hostname = "control-server"
    srv.vm.network :private_network, ip: "192.168.56.7"
  end

  # Rocky Server
  config.vm.define "worker-server" do |srv|
    srv.vm.box = "willbrid/rockylinux8"
    srv.vm.box_version = "0.0.3"
    srv.vm.hostname = "worker-server"
    srv.vm.network :private_network, ip: "192.168.56.6"
  end
end
```

- **Vagrantfile avec Ubuntu 22.04**

```
vim Vagrantfile-with-ubuntu
```

```
# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vbguest.auto_update = false
  config.vbguest.no_remote = true
  config.vbguest.iso_path = "./VBoxGuestAdditions_7.0.20.iso"

  # General Vagrant VM configuration
  config.ssh.insert_key = false
  config.vm.synced_folder ".", "/vagrant", disabled: true
  config.vm.provider :virtualbox do |v|
    v.memory = 2048
    v.cpus = 2
    v.linked_clone = true
  end
  
  # Ubuntu Server
  config.vm.define "control-server" do |us|
    us.vm.box = "bento/ubuntu-22.04"
    us.vm.box_version = "202407.23.0"
    us.vm.hostname = "control-server"
    us.vm.network :private_network, ip: "192.168.56.7"
  end

  # Ubuntu Server
  config.vm.define "worker-server" do |us|
    us.vm.box = "bento/ubuntu-22.04"
    us.vm.box_version = "202407.23.0"
    us.vm.hostname = "worker-server"
    us.vm.network :private_network, ip: "192.168.56.6"
  end
end
```

- **Démarrage des vm**

```
vagrant up
```