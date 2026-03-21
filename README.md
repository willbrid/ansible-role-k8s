# Ansible-role-k8s

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/willbrid/ansible-role-k8s/blob/main/LICENSE)

The **ansible-role-k8s** role allows you to install and configure a multi-node Kubernetes cluster for Red Hat (RHEL, CentOS, Rocky Linux) and Debian (Debian, Ubuntu) based distributions. It supports the configuration of a primary control plane, worker nodes, and other secondary control planes for a High Availability (HA) architecture.

## Requirements

The inventory file must be organized to reflect the different types of nodes in your Kubernetes cluster through the different possible role values: **primary_control_plane**, **secondary_control_plane** and **node**.

**Example for a cluster with 3 control plane nodes (1 primary node and 2 secondary nodes) and 4 worker nodes**

```bash
[primary_control_plane]
192.168.1.5

[secondary_control_plane]
192.168.1.6
192.168.1.7

[node]
192.168.1.8
192.168.1.9
192.168.1.10
192.168.1.11
```

## Description of Variables

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_version`|str|Kubernetes version to install. Format: x.y|no|`"1.29"`
`kubernetes_specific_version`|str|Specific version of Kubernetes. Format: x.y.z|no|`"1.29.13"`
`kubernetes_role`|str|Role of the node to be configured. Possible values : **primary_control_plane**, **secondary_control_plane**, **node**|no|`"primary_control_plane"`
`kubernetes_control_plane_ip`|str|Control plane node IP address|yes|`""`
`kubernetes_control_plane_endpoint`|str|IP address or DNS name of the control plane endpoint|yes|`""`
`kubernetes_cni_network`|dict|Network plugin configuration variable | Yes | See details below (`kubernetes_cni_network.cni`,`kubernetes_cni_network.cidr`, `kubernetes_cni_network.pod_host_port`, `kubernetes_cni_network.manifest`)
`kubernetes_cni_network.cni`|str|Network plugin name. Possible values : **calico**, **flannel**, **weave**|no|`"calico"`
`kubernetes_cni_network.cidr`|str|CIDR network range recommended by the network plugin|no|`"172.16.0.0/16"`
`kubernetes_cni_network.pod_host_port`|int|intercommunication host port between the network plugin pods|no|`179`
`kubernetes_cni_network.manifest`|str|network plugin installation manifest file|no|`"https://docs.projectcalico.org/manifests/calico.yaml"`
`kubernetes_control_plane_ports`|list[str]|Network ports to allow for the proper functioning of the control plane node (the documentation on the ports to allow for the control plane node should be consulted for the chosen version of Kubernetes)|no|`['6443', '2379-2380', '10250', '10257', '10259']`
`kubernetes_node_ports`|list[str]|Network ports to allow for the proper functioning of the worker node (the documentation on the ports to allow for worker nodes should be consulted for the chosen version of Kubernetes)|no|`['10250', '10256', '30000-32767']`

## Dependencies

None.

## Example Playbook

- Role installation and inventory file definition in relation to the sandbox

```bash
mkdir -p $HOME/install-k8s
```

```bash
vim $HOME/install-k8s/requirements.yml
```

```yaml
- name: ansible-role-k8s
  src: git+https://github.com/willbrid/ansible-role-k8s.git
  version: v0.0.4
```

```bash
cd $HOME/install-k8s && ansible-galaxy install --force -r requirements.yml
```

```bash
vim $HOME/install-k8s/hosts.ini
```

```bash
[primary_control_plane]
192.168.56.6

[secondary_control_plane]
192.168.56.7

[node]
192.168.56.8
```

> Note: The IP addresses defined in the file `$HOME/install-k8s/hosts.ini` are provided as an example relative to the sandbox environment and must be replaced with your own.

- Control plane node configuration and installation

```bash
vim $HOME/install-k8s/primary-control-plan.yml
```

```yaml
---
- hosts: "{{ kubernetes_role }}"
  become: yes

  vars:
    kubernetes_role: "primary_control_plane"
    kubernetes_control_plane_ip: "192.168.56.6"
    kubernetes_control_plane_endpoint: "192.168.56.6"

  roles:
    - ansible-role-k8s
```

```bash
cd $HOME/install-k8s && ansible-playbook -i hosts.ini primary-control-plan.yml
```

- Configuration and installation of secondary nodes control plane

```bash
vim $HOME/install-k8s/secondary-control-plan.yml
```

```yaml
---
- hosts: "{{ kubernetes_role }}"
  become: yes

  vars:
    kubernetes_role: "secondary_control_plane"
    kubernetes_control_plane_ip: "192.168.56.6"
    kubernetes_control_plane_endpoint: "192.168.56.6"

  roles:
    - ansible-role-k8s
```

```bash
cd $HOME/install-k8s && ansible-playbook -i hosts.ini secondary-control-plan.yml
```

- Configuration and installation of worker nodes

```bash
vim $HOME/install-k8s/worker-node.yml
```

```yaml
---
- hosts: "{{ kubernetes_role }}"
  become: yes

  vars:
    kubernetes_role: "node"
    kubernetes_control_plane_ip: "192.168.56.6"
    kubernetes_control_plane_endpoint: "192.168.56.6"

  roles:
    - ansible-role-k8s
```

```bash
cd $HOME/install-k8s && ansible-playbook -i hosts.ini worker-node.yml
```

## License

MIT

## Author Information

William Bridge NGASSAM
