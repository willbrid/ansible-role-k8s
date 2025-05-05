# Ansible-role-k8s

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/willbrid/ansible-role-k8s/blob/main/LICENSE)

Ce rôle Ansible configure et installe un cluster Kubernetes multi-nœuds pour les distributions basées sur RedHat (RHEL, CentOS, Rocky Linux) et Debian (Debian, Ubuntu). Il prend en charge la configuration d'un plan de contrôle principal, de nœuds de travail et d'autres plans de conrôle secondaires pour une architecture HA (High Availability).

## Exigences

- **Inventaire**

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

- **Variables obligatoires**

Les variables suivantes doivent être définies pour garantir que le rôle fonctionne correctement : 

--- **kubernetes_role** : pour préciser le rôle du noeud à configurer parmi : **primary_control_plane**, **secondary_control_plane** et **node**

--- **kubernetes_control_plane_ip** : pour préciser l'adresse ip du noeud plan de contrôle

--- **kubernetes_control_plane_endpoint** : pour préciser l'adresse ip ou nom dns du endpoint du plan de contrôle.

## Description des Variables

|Nom|Type|Description|Valeur par défaut|
|---|----|-----------|-----------------|
`kubernetes_version`|string|Version de kubernetes à installer. Format : x.y|`"1.29"`
`kubernetes_specific_version`|string|Version spécifique de kubernetes. Format : x.y.z|`"1.29.13"`
`kubernetes_role`|string|Rôle du noeud à configurer. Valeurs possibles : **primary_control_plane**, **secondary_control_plane**, **node**|`"primary_control_plane"`
`kubernetes_control_plane_ip`|string|Adresse IP du noeud plan de contrôle|`""`
`kubernetes_control_plane_endpoint`|string|Adresse IP ou nom dns du endpoint du plan de contrôle|`""`
`kubernetes_cni_network`|dict|Variable de configuration du plugin réseau|Voir détails ci-dessous (`kubernetes_cni_network.cni`,`kubernetes_cni_network.cidr`, `kubernetes_cni_network.pod_host_port`, `kubernetes_cni_network.manifest`)
`kubernetes_cni_network.cni`|string|Nom du plugin réseau. Valeurs possibles : **calico**, **flannel**, **weave**|`"calico"`
`kubernetes_cni_network.cidr`|string|Plage réseau cidr récommandée par le plugin réseau|`"172.16.0.0/16"`
`kubernetes_cni_network.pod_host_port`|interger|Port d'hôte d'intercommunication entre les pods du plugin réseau|`179`
`kubernetes_cni_network.manifest`|string|Fichier manifest d'installation du plugin réseau|`"https://docs.projectcalico.org/manifests/calico.yaml"`
`kubernetes_control_plane_ports`|list[string]|Ports réseau à autoriser pour le bon fonctionnement du noeud plan de contrôle|`['6443', '2379-2380', '10250', '10257', '10259']`
`kubernetes_node_ports`|list[string]|Ports réseau à autoriser pour le bon fonctionnement du noeud worker|`['10250', '10256', '30000-32767']`

## Dépendances

Aucune.

## Exemples Playbook

- Installation du rôle et définition du fichier d'inventaires

```bash
mkdir -p $HOME/install-k8s/roles
```

```bash
vim $HOME/install-k8s/requirements.yml
```

```yaml
- name: ansible-role-k8s
  src: https://github.com/willbrid/ansible-role-k8s.git
  version: v0.0.3
```

```bash
cd $HOME/install-k8s && ansible-galaxy install -r requirements.yml --roles-path roles
```

```bash
vim $HOME/install-k8s/hosts.ini
```

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

> Note: Les adresses IP définies dans le fichier `$HOME/install-k8s/hosts.ini` sont fournies à titre d'exemple et doivent être remplacées par les vôtres.

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
    kubernetes_control_plane_ip: "192.168.1.5"
    kubernetes_control_plane_endpoint: "192.168.1.5"

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
    kubernetes_control_plane_ip: "192.168.1.5"
    kubernetes_control_plane_endpoint: "192.168.1.5"

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
    kubernetes_control_plane_ip: "192.168.1.5"
    kubernetes_control_plane_endpoint: "192.168.1.5"

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
