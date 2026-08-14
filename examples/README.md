# Sandbox and reference playbooks

A complete, runnable environment for the `ansible-role-k8s` role: three Vagrant
machines (two control planes and one worker) and the playbooks that install,
verify and tear down a cluster on them.

It doubles as a set of copy-paste examples. The playbooks, the scenario files
and the group variables are the same ones you would write for a real
deployment; only the inventory and the `Vagrantfile` are sandbox specific.

```
examples/
├── Vagrantfile                     3 VMs: 2 control planes + 1 worker
├── ansible.cfg
├── requirements.yml                the role and the collections it needs
├── bin/gen-inventory.sh            writes inventory/hosts.ini from Vagrant
├── inventory/
│   ├── hosts.ini.example           reference inventory, see step 2
│   └── group_vars/all.yml          settings shared by every node
├── scenarios/
│   ├── calico-nftables.yml         Calico, nftables dataplane
│   ├── cilium-ebpf.yml             Cilium, eBPF dataplane, no kube-proxy
│   ├── custom-repo-rpm.yml         internal mirror mode (RedHat family)
│   └── custom-repo-deb.yml         internal mirror mode (Debian family)
└── playbooks/
    ├── site.yml                    the three plays in the required order
    ├── primary-control-plane.yml
    ├── secondary-control-plane.yml
    ├── worker-node.yml
    ├── verify.yml                  cluster health + dataplane checks
    ├── test-network-policy.yml     connectivity and NetworkPolicy enforcement
    └── reset.yml                   tears the cluster down, keeps the VMs
```

## Requirements

| Tool | Tested version |
|------|----------------|
| VirtualBox | 7.2.14 |
| Vagrant | 2.4.9 |
| `vagrant-vbguest` plugin | 0.32.0 (optional) |
| `ansible-core` | 2.20 (2.18 minimum) |

> Vagrant only added VirtualBox 7.2 to its list of supported providers in
> 2.4.9, so the two versions above go together.
>
> The Rocky Linux 10 box makes VirtualBox 7.2 a hard requirement: RHEL 10 and
> its rebuilds are compiled for the x86-64-v3 baseline, and older VirtualBox
> releases do not pass `FMA` and `F16C` through to the guest. The guest then
> dies before the login prompt, with `Kernel panic - not syncing: Attempted to
> kill init` on the console and a boot timeout on the Vagrant side.

The `Vagrantfile` mounts the VirtualBox Guest Additions image if it finds one in
this directory. It is not shipped here; download it only if you use the
`vagrant-vbguest` plugin:

```bash
wget https://download.virtualbox.org/virtualbox/7.2.14/VBoxGuestAdditions_7.2.14.iso
```

## Step 0 — Install the role and the collections

```bash
ansible-galaxy install -r requirements.yml
```

This installs the role under the name `ansible-role-k8s`, which is the name the
playbooks reference. Pin `version:` in [requirements.yml](requirements.yml) to a
tag rather than `main` for a reproducible run.

**Testing a local checkout instead.** To run the sandbox against your own
working copy of the role rather than the published one, point Ansible at the
directory *containing* the clone. This works as is when the clone kept its
default name, `ansible-role-k8s`, since that is the name the playbooks look up:

```bash
# from examples/, inside the clone
export ANSIBLE_ROLES_PATH="$(cd ../.. && pwd)"
```

Every change to the role is then picked up on the next run, with no
reinstallation.

## Step 1 — Start the machines

```bash
vagrant up
```

| Machine | IP | Role | vCPU / RAM |
|---------|----|------|------------|
| control-server1 | 192.168.56.6 | `primary_control_plane` | 2 / 2560 |
| control-server2 | 192.168.56.7 | `secondary_control_plane` | 2 / 2560 |
| worker-server | 192.168.56.8 | `node` | 4 / 4096 |

The default distribution is **Rocky Linux 9** (`bento/rockylinux-9`
202510.26.0). To change it:

```bash
BOX_DISTRO="bento/rockylinux-10" BOX_DISTRO_VERSION="202512.01.0" vagrant up
BOX_DISTRO="bento/ubuntu-24.04"  BOX_DISTRO_VERSION="202502.21.0" vagrant up
BOX_DISTRO="bento/ubuntu-22.04"  BOX_DISTRO_VERSION="202407.23.0" vagrant up
BOX_DISTRO="generic/debian12"    BOX_DISTRO_VERSION="4.3.12"      vagrant up
```

> The role preflight thresholds are 2 vCPU, 1700 MB of RAM and 10 GB free under
> `/var`. The `Vagrantfile` values sit just above them; `CP_MEM`, `CP_CPU`,
> `WORKER_MEM` and `WORKER_CPU` raise them.
>
> The disk threshold also constrains the choice of box, and is the reason for
> the `bento/rockylinux-9` default: the official `rockylinux/9` box ships a
> 10 GB disk with about 4.4 GB free and is rejected by the preflight. That disk
> being VMDK, Vagrant cannot grow it in place. If you switch boxes, check its
> disk with `ansible k8s_cluster -m shell -a 'df -h /var'` before running the
> playbook.

## Step 2 — Generate the inventory

```bash
./bin/gen-inventory.sh
```

The script reads `vagrant ssh-config` and writes `inventory/hosts.ini` with the
user and private key actually in use. That file is not committed, since the key
path is specific to your machine; [inventory/hosts.ini.example](inventory/hosts.ini.example)
shows what it looks like.

The inventory hostnames **are the private network IP addresses**, and that is
not cosmetic: the role uses `ansible_host` as the kubelet node IP, and delegates
the `kubeadm token create` call to `kubernetes_control_plane_ip`, which
therefore has to be an inventory host.

Check the result:

```bash
ansible k8s_cluster -m ping
```

## Step 3 — Check the requirements without changing anything

The `preflight` tag runs the checks only:

```bash
ansible-playbook playbooks/site.yml -e @scenarios/calico-nftables.yml --tags preflight
```

This is the quickest way to tell whether a distribution is eligible. On failure
the message says exactly what is missing, for example:

```
The Calico nftables dataplane requires a Linux kernel >= 5.13, this host runs 5.10.0-30-amd64.
Upgrade the kernel, or select kubernetes_cni: cilium.
```

## Step 4 — Install the cluster

### Scenario A — Calico with the nftables dataplane

```bash
ansible-playbook playbooks/site.yml -e @scenarios/calico-nftables.yml
```

### Scenario B — Cilium replacing kube-proxy

```bash
ansible-playbook playbooks/site.yml -e @scenarios/cilium-ebpf.yml
```

### Scenario C — Internal mirror mode

The scenario file sets `kubernetes_package_source: custom` while pointing at the
upstream URLs: the "internal mirror" code path is genuinely exercised without
having to stand a mirror up.

```bash
# RedHat family
ansible-playbook playbooks/site.yml \
  -e @scenarios/calico-nftables.yml -e @scenarios/custom-repo-rpm.yml

# Debian family
ansible-playbook playbooks/site.yml \
  -e @scenarios/calico-nftables.yml -e @scenarios/custom-repo-deb.yml
```

For a full air-gapped test, replace the `baseurl` values with your own mirrors
and add `kubernetes_image_repository`, `kubernetes_helm.archive_url` and the
`chart_repo_url` / `image_registry` keys of the CNI.

> `kubernetes_cni` has to be identical across the three groups: it drives the
> kube-proxy mode, the kernel preparation and the firewall rules of **every**
> node, not just the CNI deployment. Hence the scenario file passed with `-e`,
> which applies to the whole `site.yml`.

To install node by node rather than in one pass:

```bash
ansible-playbook playbooks/primary-control-plane.yml   -e @scenarios/calico-nftables.yml
ansible-playbook playbooks/secondary-control-plane.yml -e @scenarios/calico-nftables.yml
ansible-playbook playbooks/worker-node.yml             -e @scenarios/calico-nftables.yml
```

## Step 5 — Verify the cluster

```bash
ansible-playbook playbooks/verify.yml -e @scenarios/calico-nftables.yml
```

This playbook checks that:

- the 3 nodes are `Ready`
- no pod is stuck outside `Running` / `Succeeded`
- **Calico**: kube-proxy is in `mode: nftables`, the `cali*` tables are present
  in the node nftables ruleset, and the `calico-node` DaemonSet is rolled out
- **Cilium**: no `kube-proxy` DaemonSet exists, and the agent answers `OK` to
  `cilium-dbg status`

## Step 6 — Test the NetworkPolicies

```bash
ansible-playbook playbooks/test-network-policy.yml
```

Deploys a `web` service (2 replicas, preferred anti-affinity) and a `client`
pod, then checks in order:

1. the client reaches the service through cluster DNS and the service VIP —
   which validates CoreDNS, the load balancing and the overlay
2. after a `default-deny-ingress` NetworkPolicy, the traffic is **blocked**
3. after an `allow-client-to-web` policy, the traffic flows again

The test namespace is deleted at the end.

## Inspecting the cluster by hand

```bash
vagrant ssh control-server1
sudo kubectl get nodes -o wide
sudo kubectl get nodes -L willbrid.k8s/role                              # bootstrap role of each node
sudo kubectl get pods -A
sudo kubectl -n kube-system get cm kube-proxy -o yaml | grep mode:       # calico
sudo nft list tables                                                     # calico
sudo kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg status  # cilium
sudo helm list -A
```

Or from the controller, by fetching the kubeconfig. It belongs to root on the
VM, so it has to go through `sudo cat`:

```bash
vagrant ssh control-server1 -c "sudo cat /etc/kubernetes/admin.conf" > /tmp/sandbox.kubeconfig
chmod 600 /tmp/sandbox.kubeconfig
KUBECONFIG=/tmp/sandbox.kubeconfig kubectl get nodes -o wide
```

The server it references is `192.168.56.6:6443`, reachable from the host through
the host-only network; no rewriting is needed.

## Starting a test over

Quick teardown, keeping the VMs, the packages and the repositories already
installed:

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/site.yml -e @scenarios/cilium-ebpf.yml
```

Guaranteed clean slate, somewhat slower:

```bash
vagrant destroy -f && vagrant up && ./bin/gen-inventory.sh
```

> Switching from Calico to Cilium (or back) without a `reset` does not work: the
> kube-proxy mode is frozen at `kubeadm init`, and the two dataplanes do not
> coexist. Always run `reset.yml` or `vagrant destroy` between two CNI
> scenarios.

## Distribution compatibility matrix

| Distribution | Kernel | Calico + nftables | Cilium |
|--------------|--------|-------------------|--------|
| Rocky Linux 10 | 6.12 | yes | yes |
| Rocky Linux 9 | 5.14 | yes | yes |
| Debian 12 | 6.1 | yes | yes |
| Ubuntu 22.04 | 5.15 | yes | yes |
| Ubuntu 24.04 | 6.8 | yes | yes |
| Debian 11 | 5.10 | **no** (< 5.13) | yes |

## Troubleshooting

| Symptom | Likely cause |
|---------|--------------|
| Rocky Linux 10: boot timeout, and `Attempted to kill init` on the VM console | VirtualBox older than 7.2, which does not give the guest the x86-64-v3 instruction set EL10 is built for |
| Rocky Linux 10: `Unknown resource type 32768 in hardware item` while importing the box | Same cause: the box OVF is EFI/NVRAM, which older VirtualBox releases cannot read |
| `Vagrant has detected ... version of VirtualBox ... not supported` | Vagrant older than 2.4.9 against VirtualBox 7.2 |
| `ansible k8s_cluster -m ping` fails with permission denied | Inventory not generated: run `./bin/gen-inventory.sh` |
| `the role 'ansible-role-k8s' was not found` | Step 0 skipped, or `ANSIBLE_ROLES_PATH` not exported in this shell |
| Preflight: "Repository ... is unreachable" | No internet access from the VMs, switch to `kubernetes_package_source: custom` |
| Preflight: "A Kubernetes node needs at least 2 vCPU and 1700 MB" | Re-run `vagrant up` with a higher `CP_MEM`. The control planes default to 2560 rather than 2048 because Rocky Linux 10 reports only 1686 MB usable out of 2048 |
| `kubeadm join` fails on the worker | The primary control plane is not installed, or `kubernetes_control_plane_ip` is not an inventory host |
| Pods stay `Pending` | Both control planes are `NoSchedule` by default; only `worker-server` takes workloads |
