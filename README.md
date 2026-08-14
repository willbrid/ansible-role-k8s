# Ansible-role-k8s

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/willbrid/ansible-role-k8s/blob/main/LICENSE)

The **ansible-role-k8s** role installs and configures a multi-node Kubernetes cluster on Red Hat (RHEL, Rocky Linux) and Debian (Debian, Ubuntu) based distributions. It supports a primary control plane, additional control planes for a High Availability (HA) architecture, and worker nodes.

- Container runtime: **CRI-O**
- Cluster bootstrap: **kubeadm**, driven by a rendered `kubeadm-config.yaml` instead of ad-hoc command line flags
- Network plugin: **Calico** with the nftables dataplane, or **Cilium** with the eBPF dataplane, both deployed with Helm
- Package sources: upstream repositories, or **internal mirrors** for hosts with no direct internet access

## Requirements

- `ansible-core` >= 2.18 on the controller
- Root privileges on the managed nodes (`become: true`)
- The following collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

| Collection | Minimum version |
|------------|-----------------|
| `ansible.posix` | 2.2.2 |
| `community.general` | 9.0.0 |
| `kubernetes.core` | 5.0.0 |

The inventory must reflect the node types through the possible values of `kubernetes_role`: **primary_control_plane**, **secondary_control_plane** and **node**.

**Example of a cluster with 3 control plane nodes (1 primary and 2 secondary) and 4 worker nodes**

```ini
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

> The host declared in `kubernetes_control_plane_ip` must belong to the inventory: joining a node delegates the `kubeadm token create` call to it.

## Node requirements

The role runs a preflight phase before touching the system, and stops with an explicit message when a requirement is not met.

**Every node**

| Requirement | Value |
|-------------|-------|
| Distribution | Debian or RedHat family |
| Architecture | x86_64, aarch64, ppc64le, s390x |
| Init system | systemd |
| vCPU | >= 2 |
| RAM | >= 1700 MB |
| Free space under `/var` | >= 10 GB |
| Kubernetes version | >= 1.30 |
| Package repositories | reachable |

**Calico, nftables dataplane**

| Requirement | Value |
|-------------|-------|
| Linux kernel | >= 5.13 |
| `nft` userspace | >= 1.0.1 |
| `nf_tables` kernel module | built in or loadable |
| Kubernetes version | >= 1.31, kube-proxy nftables mode is only GA from 1.33 |

The role always couples Calico with the nftables dataplane, so kube-proxy is configured in `nftables` mode as well: Kubernetes service rules and Calico rules then share the same packet filtering framework.

**Cilium, eBPF dataplane**

| Requirement | Value |
|-------------|-------|
| Linux kernel | >= 5.10, waived on RHEL 8 whose 4.18 kernel carries the backports |
| Kernel build options | `CONFIG_BPF`, `CONFIG_BPF_SYSCALL`, `CONFIG_BPF_JIT`, `CONFIG_BPF_EVENTS`, `CONFIG_CGROUPS`, `CONFIG_CGROUP_BPF`, `CONFIG_NET_CLS_BPF`, `CONFIG_NET_CLS_ACT`, `CONFIG_NET_SCH_INGRESS`, `CONFIG_PERF_EVENTS`, `CONFIG_CRYPTO_SHA1`, `CONFIG_CRYPTO_USER_API_HASH`, `CONFIG_FIB_RULES`, plus `CONFIG_VXLAN` or `CONFIG_GENEVE` |
| BTF metadata | `/sys/kernel/btf/vmlinux` present |
| cgroup v2 | unified hierarchy on `/sys/fs/cgroup`, required by the kube-proxy replacement |

## Description of Variables

### Node role and versions

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_role`|str|Role played by the host. Possible values: **primary_control_plane**, **secondary_control_plane**, **node**|no|`"primary_control_plane"`
`kubernetes_node_role_label`|str|Node label carrying `kubernetes_role`. Empty string to label nothing|no|`"willbrid.k8s/role"`
`kubernetes_version`|str|Kubernetes minor version in x.y form, drives the package repositories. Minimum 1.30|no|`"1.34"`
`kubernetes_patch_version`|str|Exact package version in x.y.z form, must belong to `kubernetes_version`|no|`"1.34.10"`

Kubernetes draws no line between a primary and a secondary control plane: `primary_control_plane` and `secondary_control_plane` differ only in how they were bootstrapped, `kubeadm init` against `kubeadm join --control-plane`, and end up with strictly identical labels, taints and etcd voting rights. Losing the primary promotes nobody, because there is nothing to promote.

Since that information cannot be read back from the API, the role writes it down as a node label:

```bash
kubectl get nodes -L willbrid.k8s/role
NAME              STATUS   ROLES           AGE   VERSION    ROLE
control-server1   Ready    control-plane   37m   v1.34.10   primary_control_plane
control-server2   Ready    control-plane   33m   v1.34.10   secondary_control_plane
worker-server     Ready    <none>          31m   v1.34.10   node
```

It is documentation, not behaviour: nothing in the role reads that label back. It answers "which node do I delegate a `kubeadm token create` to" long after the install.

### Cluster endpoints and networking

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_control_plane_ip`|str|IP address of the primary control plane node, must be an inventory host|yes|`""`
`kubernetes_control_plane_endpoint`|str|IP address or DNS name every node uses to reach the API server, point it to a load balancer for a HA cluster|yes|`""`
`kubernetes_control_plane_port`|int|TCP port the API server listens on|no|`6443`
`kubernetes_node_ip`|str|IP address advertised by the kubelet of the current host|no|`"{{ ansible_host \| default(ansible_facts.default_ipv4.address) }}"`
`kubernetes_kubeconfig_dir`|path|Directory receiving the administrator kubeconfig on the control plane nodes|no|`"/root/.kube"`
`kubernetes_pod_network_cidr`|str|IPv4 CIDR allocated to the pods, also used for the CNI IP pool|no|`"172.16.0.0/16"`
`kubernetes_service_cidr`|str|IPv4 CIDR allocated to the services|no|`"10.96.0.0/12"`
`kubernetes_image_repository`|str|Registry kubeadm pulls the control plane images from, empty means `registry.k8s.io`|no|`""`

### Package sources

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_package_source`|str|**internet** uses the upstream repositories, **custom** uses the mirrors of `kubernetes_package_repositories`|no|`"internet"`
`kubernetes_package_repositories`|dict|Per repository configuration, see below|no|see below
`kubernetes_package_repositories.<repo>.baseurl`|str|Base URL of the mirror, trailing slash included|only in custom mode|`""`
`kubernetes_package_repositories.<repo>.gpg_key_url`|str|URL of the ASCII armored signing key|no|`""`
`kubernetes_package_repositories.<repo>.gpg_key_content`|str|Inline ASCII armored signing key, takes precedence over `gpg_key_url`|no|`""`
`kubernetes_package_repositories.<repo>.gpg_check`|bool|Verify the package signatures|no|`true`
`kubernetes_packages_hold`|bool|Pin `kubelet`, `kubeadm`, `kubectl` and `cri-o` so a distribution upgrade never moves them|no|`true`

`<repo>` is either `kubernetes` or `crio`. In **internet** mode the role uses:

- Kubernetes: `https://pkgs.k8s.io/core:/stable:/v<version>/{deb,rpm}/`
- CRI-O: `https://download.opensuse.org/repositories/isv:/cri-o:/stable:/v<version>/{deb,rpm}/`

> CRI-O left `pkgs.k8s.io` and now publishes its packages on `download.opensuse.org`, which is why the CRI-O base URL no longer shares the Kubernetes one.

### CNI

> `kubernetes_helm`, `kubernetes_calico` and `kubernetes_cilium` are merged key by key over the defaults listed below, so only set the keys you want to change. Writing `kubernetes_calico: {bgp: "Enabled"}` keeps every other Calico default, it does not reset them.

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_cni`|str|Network plugin. Possible values: **calico**, **cilium**|no|`"calico"`
`kubernetes_helm.version`|str|Helm version installed on the primary control plane|no|`"3.21.3"`
`kubernetes_helm.archive_url`|str|Alternate URL of the Helm tarball, empty means `get.helm.sh`|no|`""`
`kubernetes_helm.checksum`|str|Checksum of the tarball, in the `algorithm:value` form|no|`""`
`kubernetes_helm.binary_path`|path|Where the Helm binary is installed|no|`"/usr/local/bin/helm"`

**Calico**, read when `kubernetes_cni` is `calico`:

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_calico.version`|str|Version of the `tigera-operator` and CRD charts|no|`"v3.32.1"`
`kubernetes_calico.namespace`|str|Namespace hosting the Tigera operator|no|`"tigera-operator"`
`kubernetes_calico.chart_repo_url`|str|Helm repository serving the Calico charts|no|`"https://docs.tigera.io/calico/charts"`
`kubernetes_calico.chart_ref`|str|Direct reference to the `tigera-operator` chart, local path, `.tgz` or `oci://` URL, bypasses `chart_repo_url`|no|`""`
`kubernetes_calico.crds_chart_ref`|str|Direct reference to the `crd.projectcalico.org.v1` chart|no|`""`
`kubernetes_calico.image_registry`|str|Registry the Calico images are pulled from|no|`""`
`kubernetes_calico.image_path`|str|Path prefix appended to `image_registry` when the mirror does not reproduce the upstream layout|no|`""`
`kubernetes_calico.image_pull_secret`|str|Name of an existing pull secret giving access to the mirror|no|`""`
`kubernetes_calico.encapsulation`|str|Overlay encapsulation. Possible values: **VXLAN**, **VXLANCrossSubnet**, **IPIP**, **IPIPCrossSubnet**, **None**|no|`"VXLANCrossSubnet"`
`kubernetes_calico.bgp`|str|BGP routing. Possible values: **Enabled**, **Disabled**|no|`"Disabled"`
`kubernetes_calico.wireguard_enabled`|bool|Encrypt the pod to pod traffic with WireGuard|no|`false`
`kubernetes_calico.api_server`|bool|Deploy the Calico API server|no|`true`
`kubernetes_calico.block_size`|int|Size of the IPAM blocks carved out of the pod CIDR|no|`26`
`kubernetes_calico.values`|dict|Extra Helm values merged on top of the ones rendered by the role|no|`{}`

**Cilium**, read when `kubernetes_cni` is `cilium`:

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_cilium.version`|str|Version of the `cilium` chart|no|`"1.20.0"`
`kubernetes_cilium.namespace`|str|Namespace hosting the Cilium agent and operator|no|`"kube-system"`
`kubernetes_cilium.chart_repo_url`|str|Helm repository serving the `cilium` chart|no|`"https://helm.cilium.io"`
`kubernetes_cilium.chart_ref`|str|Direct reference to the `cilium` chart, local path, `.tgz` or `oci://` URL, bypasses `chart_repo_url`|no|`""`
`kubernetes_cilium.image_registry`|str|Replacement for the `quay.io/cilium` image prefix, for example `registry.corp.local/cilium`|no|`""`
`kubernetes_cilium.kube_proxy_replacement`|bool|Let Cilium replace kube-proxy, kubeadm then skips the kube-proxy addon entirely|no|`true`
`kubernetes_cilium.operator_replicas`|int|Number of `cilium-operator` replicas. The operator runs on the host network with fixed host ports and an anti affinity rule, so a node carries at most one replica: asking for more replicas than the cluster has nodes leaves the extra ones `Pending` until nodes join. The CNI is installed while the cluster is still the single node that ran `kubeadm init`, hence the default of one. Set it to `2` for an active/standby operator on a multi node cluster|no|`1`
`kubernetes_cilium.routing_mode`|str|Routing mode. Possible values: **tunnel**, **native**|no|`"tunnel"`
`kubernetes_cilium.tunnel_protocol`|str|Encapsulation protocol, read when `routing_mode` is `tunnel`. Possible values: **vxlan**, **geneve**|no|`"vxlan"`
`kubernetes_cilium.wireguard_enabled`|bool|Encrypt the pod to pod traffic with WireGuard|no|`false`
`kubernetes_cilium.hubble_enabled`|bool|Deploy Hubble and its relay|no|`false`
`kubernetes_cilium.values`|dict|Extra Helm values merged on top of the ones rendered by the role|no|`{}`

### Firewall and system

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_firewall_manage`|bool|Let the role drive the host firewall: firewalld on the RedHat family, ufw on the Debian family. Set it to `false` to keep the role away from both|no|`true`
`kubernetes_control_plane_ports`|list[str]|TCP ports opened on the control plane nodes, Cilium only|no|`['6443', '2379-2380', '10250', '10257', '10259']`
`kubernetes_node_ports`|list[str]|TCP ports opened on the worker nodes, Cilium only|no|`['10250', '10256', '30000-32767']`
`kubernetes_extra_tcp_ports`|list[str]|Additional TCP ports opened on every node, Cilium only|no|`[]`
`kubernetes_extra_udp_ports`|list[str]|Additional UDP ports opened on every node, Cilium only|no|`[]`
`kubernetes_disable_swap`|bool|Turn swap off and comment out the swap entries of `/etc/fstab`|no|`true`
`kubernetes_selinux_permissive`|bool|Switch SELinux to permissive on the RedHat family|no|`true`

**What the role does depends on the CNI**, because the two dataplanes do not coexist with a host firewall in the same way.

**Calico — the host firewall is stopped and disabled.** The nftables dataplane rewrites the host ruleset with a newer `nft` than the one packaged by the distribution, and the distribution tooling then crashes reading it back: on Rocky Linux 9.6, `nft list ruleset` segfaults as soon as Calico is up, and firewalld dies with it since it calls the same `libnftables`. Tigera documents this and [requires firewalld and iptables managers to be disabled](https://docs.tigera.io/calico/latest/getting-started/kubernetes/requirements). Network policy is then enforced by Calico itself, and the four port variables above are not applied. Protect the nodes upstream, at the hypervisor, the cloud security group or an external firewall.

**Cilium — firewalld and ufw are configured.** The eBPF datapath leaves the host nftables ruleset alone, so the host firewall keeps working. The ports are derived automatically from the Cilium configuration and do not have to be listed:

| Setting | Ports opened |
|---------|--------------|
| always | 4240/tcp (health) |
| `routing_mode: tunnel` | 8472/udp with VXLAN, 6081/udp with Geneve |
| `hubble_enabled: true` | 4244-4245/tcp |
| `wireguard_enabled: true` | 51871/udp |

The interfaces Cilium drives (`cilium_host`, `cilium_net`, `cilium_vxlan`, `cilium_geneve`, `lxc+`, plus `cilium_wg0` with WireGuard) are moved to the `trusted` zone of firewalld. On the Debian family, ufw accepts the whole pod network instead.

For reference, the ports Calico needs if you run an external firewall: 5473/tcp (Typha), 179/tcp with BGP, 4789/udp with VXLAN, 51820-51821/udp with WireGuard, and IP protocol 4 (`ipencap`) with IPIP.

Two requirements are handled for both CNIs:

- `/etc/NetworkManager/conf.d/99-kubernetes-cni.conf` marks the CNI interfaces unmanaged, which Calico requires. Without it NetworkManager claims them and takes the datapath down on its next reload;
- the matching kernel modules (`vxlan`, `geneve`, `ipip`, `wireguard`) are loaded and made persistent according to the encapsulation actually configured.

**RHEL 10 and its rebuilds need one extra package.** `br_netfilter` moved from `kernel-modules-core`, installed with the base kernel, to `kernel-modules-extra`, which a minimal image does not carry, and `modprobe` then stops the run:

```
modprobe: FATAL: Module br_netfilter not found in directory /lib/modules/6.12.0-211.16.1.el10_2.0.1.x86_64
```

The role probes the running kernel with `modinfo` and installs `kernel-modules-extra` for **that exact kernel** when a module is missing, so RHEL 9 is left untouched and no node ends up with modules built for a kernel it is not running. If the running kernel is older than the one the repositories now carry, the pinned package no longer exists there and the role stops with a message asking for a `dnf -y update kernel\*` and a reboot.

### Preflight checks

|Name|Type|Description|Mandatory|Default value|
|----|----|-----------|---------|-------------|
`kubernetes_preflight_enabled`|bool|Check the host and CNI requirements before touching the system|no|`true`
`kubernetes_preflight_min_cpu`|int|Minimum number of vCPU|no|`2`
`kubernetes_preflight_min_memory_mb`|int|Minimum amount of RAM in MB|no|`1700`
`kubernetes_preflight_min_free_disk_mb`|int|Minimum free space in MB on the filesystem hosting `/var`|no|`10240`

## Tests

```bash
ansible-playbook tests/assert-variables.yml
```

Runs every derivation the role makes from its public interface against a table of expected results, on `localhost`, without a cluster or even a managed node. It covers the dictionary merge of `kubernetes_helm` / `kubernetes_calico` / `kubernetes_cilium`, the firewall port list, the trusted and NetworkManager-unmanaged interfaces, the kernel modules, the kube-proxy mode and the package repository URLs.

Add a case to the `cases` list of [tests/assert-variables.yml](tests/assert-variables.yml): `given` holds the public variables, `expect` the internal constants and their expected value. A case that leaves a key out inherits it from a snapshot of `defaults/main.yml`, never from a literal, so changing a default is caught rather than written down twice.

The one exception is `kubernetes_node_role_label`, deliberately pinned to its literal value: the label key is a published interface people write queries against, so moving it should require an explicit decision rather than slipping through.

What this does not cover: the `when:` gates, `kubernetes_firewall_manage`, `kubernetes_preflight_enabled` and `kubernetes_selinux_permissive`, which decide whether a block runs rather than what it computes. Checking those means running the tasks, which is Molecule territory.

On a real cluster, [examples/](examples) carries the other half: `playbooks/verify.yml` asserts that every node is `Ready`, that no pod is stuck and that the selected dataplane is actually the one in place, and `playbooks/test-network-policy.yml` checks that a default-deny NetworkPolicy is enforced and that an explicit allow restores the traffic.

## Dependencies

None, apart from the collections listed above.

## Example Playbook

The snippets below build a deployment from scratch, one file at a time. [examples/](examples) holds the same thing already assembled and runnable: a three node Vagrant sandbox, the scenario files for each CNI, and playbooks to install, verify, exercise the NetworkPolicies and tear the cluster back down. Start there to try the role out, come back here to write your own.

### Role installation and inventory

```bash
mkdir -p $HOME/install-k8s && cd $HOME/install-k8s
```

```bash
vim $HOME/install-k8s/requirements.yml
```

```yaml
---
roles:
  - name: ansible-role-k8s
    src: git+https://github.com/willbrid/ansible-role-k8s.git
    version: v1.0.0

collections:
  - name: ansible.posix
  - name: community.general
  - name: kubernetes.core
```

```bash
cd $HOME/install-k8s && ansible-galaxy install --force -r requirements.yml
```

```bash
vim $HOME/install-k8s/hosts.ini
```

```ini
[primary_control_plane]
192.168.56.6

[secondary_control_plane]
192.168.56.7

[node]
192.168.56.8
```

> The IP addresses are the ones of the sandbox described in [sandbox.md](sandbox.md), replace them with your own.

### Primary control plane

```bash
vim $HOME/install-k8s/primary-control-plane.yml
```

```yaml
---
- name: Install the primary control plane
  hosts: "{{ kubernetes_role }}"
  become: true

  vars:
    kubernetes_role: "primary_control_plane"
    kubernetes_control_plane_ip: "192.168.56.6"
    kubernetes_control_plane_endpoint: "192.168.56.6"
    kubernetes_cni: "calico"

  roles:
    - ansible-role-k8s
```

```bash
cd $HOME/install-k8s && ansible-playbook -i hosts.ini primary-control-plane.yml
```

### Additional control planes

```yaml
---
- name: Install the additional control planes
  hosts: "{{ kubernetes_role }}"
  become: true

  vars:
    kubernetes_role: "secondary_control_plane"
    kubernetes_control_plane_ip: "192.168.56.6"
    kubernetes_control_plane_endpoint: "192.168.56.6"
    kubernetes_cni: "calico"

  roles:
    - ansible-role-k8s
```

### Worker nodes

```yaml
---
- name: Install the worker nodes
  hosts: "{{ kubernetes_role }}"
  become: true

  vars:
    kubernetes_role: "node"
    kubernetes_control_plane_ip: "192.168.56.6"
    kubernetes_control_plane_endpoint: "192.168.56.6"
    kubernetes_cni: "calico"

  roles:
    - ansible-role-k8s
```

> `kubernetes_cni` and the CNI settings must be identical across the three playbooks: they drive the kube-proxy mode, the kernel preparation and the firewall rules of every node, not only the CNI deployment done on the primary control plane.

### Cilium instead of Calico

```yaml
  vars:
    kubernetes_role: "primary_control_plane"
    kubernetes_control_plane_ip: "192.168.56.6"
    kubernetes_control_plane_endpoint: "192.168.56.6"
    kubernetes_cni: "cilium"
    kubernetes_cilium:
      kube_proxy_replacement: true
      routing_mode: "tunnel"
      tunnel_protocol: "vxlan"
      hubble_enabled: true
```

With `kube_proxy_replacement` enabled, kubeadm skips the kube-proxy addon and Cilium takes over service load balancing.

### Air-gapped installation on internal mirrors

```yaml
  vars:
    kubernetes_role: "primary_control_plane"
    kubernetes_control_plane_ip: "192.168.56.6"
    kubernetes_control_plane_endpoint: "192.168.56.6"

    # Packages
    kubernetes_package_source: "custom"
    kubernetes_package_repositories:
      kubernetes:
        baseurl: "https://mirror.corp.local/kubernetes/v1.34/deb/"
        gpg_key_url: "https://mirror.corp.local/kubernetes/Release.key"
        gpg_check: true
      crio:
        baseurl: "https://mirror.corp.local/cri-o/v1.34/deb/"
        gpg_key_content: |
          -----BEGIN PGP PUBLIC KEY BLOCK-----
          ...
          -----END PGP PUBLIC KEY BLOCK-----
        gpg_check: true

    # Control plane images
    kubernetes_image_repository: "registry.corp.local/kubernetes"

    # Helm client
    kubernetes_helm:
      version: "3.21.3"
      archive_url: "https://mirror.corp.local/helm/helm-v3.21.3-linux-amd64.tar.gz"
      checksum: "sha256:<checksum of the tarball>"

    # CNI charts and images
    kubernetes_cni: "calico"
    kubernetes_calico:
      chart_repo_url: "https://charts.corp.local/calico"
      image_registry: "registry.corp.local"
```

Every element the role downloads is redirectable: the package repositories, the control plane images, the Helm tarball, the CNI charts and the CNI images. For Cilium, `kubernetes_cilium.image_registry` replaces the `quay.io/cilium` prefix, so `registry.corp.local/cilium` makes the agent image `registry.corp.local/cilium/cilium`.

## Tags

| Tag | Effect |
|-----|--------|
| `preflight` | Only run the requirement checks |
| `packages` | Only reconfigure the package repositories |
| `cni` | Only redeploy the CNI, on the primary control plane |

## License

MIT

## Author Information

William Bridge NGASSAM
