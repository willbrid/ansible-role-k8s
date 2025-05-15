# Ansible-role-k8s

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/willbrid/ansible-role-k8s/blob/main/LICENSE)

Le rôle **ansible-role-k8s** permet d'installer et de configurer un cluster Kubernetes multi-nœuds pour les distributions basées sur RedHat (RHEL, CentOS, Rocky Linux) et Debian (Debian, Ubuntu). Il prend en charge la configuration d'un plan de contrôle principal, de nœuds de travail et d'autres plans de conrôle secondaires pour une architecture HA (High Availability).

## Exigences

Le fichier d'inventaire doit être organisé pour refléter les différents types de nœuds de votre cluster Kubernetes au travers des différentes valeurs possibles de rôle : **primary_control_plane**, **secondary_control_plane** et **node**.

**Exemple pour un cluster avec 3 noeuds plan de contrôle (1 noeud principal et 2 noeuds secondaire) et 4 noeuds worker**

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

## Description des Variables

|Nom|Type|Description|Obligatoire|Valeur par défaut|
|---|----|-----------|-----------|-----------------|
`kubernetes_version`|str|version de kubernetes à installer. Format : x.y|non|`"1.29"`
`kubernetes_specific_version`|str|version spécifique de kubernetes. Format : x.y.z|non|`"1.29.13"`
`kubernetes_role`|str|rôle du noeud à configurer. Valeurs possibles : **primary_control_plane**, **secondary_control_plane**, **node**|non|`"primary_control_plane"`
`kubernetes_control_plane_ip`|str|Adresse IP du noeud plan de contrôle|oui|`""`
`kubernetes_control_plane_endpoint`|str|Adresse IP ou nom dns du endpoint du plan de contrôle|oui|`""`
`kubernetes_cni_network`|dict|variable de configuration du plugin réseau|oui|Voir détails ci-dessous (`kubernetes_cni_network.cni`,`kubernetes_cni_network.cidr`, `kubernetes_cni_network.pod_host_port`, `kubernetes_cni_network.manifest`)
`kubernetes_cni_network.cni`|str|nom du plugin réseau. Valeurs possibles : **calico**, **flannel**, **weave**|non|`"calico"`
`kubernetes_cni_network.cidr`|str|plage réseau cidr récommandée par le plugin réseau|non|`"172.16.0.0/16"`
`kubernetes_cni_network.pod_host_port`|int|port d'hôte d'intercommunication entre les pods du plugin réseau|non|`179`
`kubernetes_cni_network.manifest`|str|fichier manifest d'installation du plugin réseau|non|`"https://docs.projectcalico.org/manifests/calico.yaml"`
`kubernetes_control_plane_ports`|list[str]|ports réseau à autoriser pour le bon fonctionnement du noeud plan de contrôle (la documentation sur les ports à autoriser du noeud plan de contrôle devrait être consultée pour la version de kubernetes choisie)|non|`['6443', '2379-2380', '10250', '10257', '10259']`
`kubernetes_node_ports`|list[str]|ports réseau à autoriser pour le bon fonctionnement du noeud worker (la documentation sur les ports à autoriser des noeuds worker devrait être consultée pour la version de kubernetes choisie)|non|`['10250', '10256', '30000-32767']`

## Dépendances

Aucune.

## Exemples Playbook

- Installation du rôle et définition du fichier d'inventaires relativement à la sandbox

```bash
mkdir -p $HOME/install-k8s/roles
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
cd $HOME/install-k8s && ansible-galaxy install -r requirements.yml --roles-path roles
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

> Note: Les adresses IP définies dans le fichier `$HOME/install-k8s/hosts.ini` sont fournies à titre d'exemple relativement à l'environnement de la sandbox et doivent être remplacées par les vôtres.

- Configuration et Installation du noeud plan de contrôle

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

- Configuration et installation des noeuds secondaires plan de contrôle

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

- Configuration et installation des noeuds worker

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

## Licence

MIT

## Informations sur l'auteur

William Bridge NGASSAM
