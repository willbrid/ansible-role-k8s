# Sandbox with VirtualBox 7.0.24

This page builds the virtual machines by hand, to be paired with the playbooks of the [Example Playbook](README.md#example-playbook) section. For a sandbox that is already assembled, with its inventory generator, its scenario files and its verification playbooks, use [examples/](examples) instead.

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

  box_distro = ENV['BOX_DISTRO'] || "bento/rockylinux-9"
  box_version = ENV['BOX_DISTRO_VERSION'] || "202510.26.0"

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

To boot the machines according to the Rocky Linux 9, Debian 12, Ubuntu 24.04, Ubuntu 22.04 distributions, proceed as follows:

> Note: Calico is only installed with the nftables dataplane, which requires a kernel >= 5.13. Ubuntu 22.04 ships a 5.15 kernel and Debian 12 a 6.1 kernel, both are fine, but Debian 11 (5.10) is rejected by the preflight checks. Use `kubernetes_cni: cilium` there, or pick a more recent box.

> Note: the preflight also requires 10 GB free under `/var`, which rules out some boxes. The official `rockylinux/9` box only carries a 10 GB disk, leaving about 4.4 GB free, hence `bento/rockylinux-9` above. Its disk is a VMDK, so Vagrant cannot grow it in place either. Check any new box with `df -h /var` before running the playbook.

- for rockylinux9

```
vagrant up
```

> Note: By default for rockylinux9, BOX_DISTRO="bento/rockylinux-9" and BOX_DISTRO_VERSION="202510.26.0" 

- for ubuntu24.04

```
BOX_DISTRO="bento/ubuntu-24.04" BOX_DISTRO_VERSION="202502.21.0" vagrant up
```

- pour debian12

```
BOX_DISTRO="generic/debian12" BOX_DISTRO_VERSION="4.3.12" vagrant up
```
