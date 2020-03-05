# Ansible Kubernetes

Ansible playbooks for creating multiple Kubernetes clusters with kubeadm.

## Description

These playbooks allow you to create multiple Kubernetes clusters simultaneously.

In particular, you can create any number of clusters with a single master node and any number of worker nodes.

_Clusters with multiple master nodes are currently not supported._

## Playbooks

There are multiple playbooks corresponding to the different stages of the installation process:

- [`install-kubeadm.yml`](playbooks/install-kubeadm.yml): install kubeadm on all nodes
- [`init.yml`](playbooks/init.yml): run `kubeadm init` on the master nodes (installs control plane)
- [`join.yml`](playbooks/join.yml): run `kubeadm join` on the worker nodes (makes worker nodes join their master node)
- [`reset.yml`](playbooks/reset.yml): run `kubeadm reset`  on all nodes  (uninstalls Kubernetes)
- [`remove-kubeadm.yml`](playbooks/remove-kubeadm.yml): remove kubeadm from all nodes

## Inventory

An example inventory for a single Kubernetes cluster with 1 master node and 2 worker nodes looks as follows:

```ini
[all]
k8s-master ansible_host=34.65.216.227
k8s-worker-1 ansible_host=34.65.247.83
k8s-worker-2 ansible_host=34.65.192.69

[master]
k8s-master

[worker]
k8s-worker-1 master=k8s-master
k8s-worker-2 master=k8s-master
```

Note the following things:

- Each node must have the `ansible_host` variable containing a publicly accessible IP address of the node
- Each worker node must have a `master` variable containing the name of its master node

For defining multiple clusters, just add additional nodes with appropriate variables to the `[all]`, `[master]`, and `[worker]` groups (see example usage below).

## Example usage

Imagine, you want to create two Kubernetes clusters `foo` and `bar`:

- `foo`: 1 master node, 2 worker nodes
- `bar`: 1 master node, 1 worker node

Here's how to do it.

### 1. Create the inventory

Edit the `inventory.ini` file to look as follows (adapt IP addresses to your use case):

```ini
[all]
master-foo ansible_host=34.65.216.227
master-bar ansible_host=34.65.229.42
worker-1-foo ansible_host=34.65.247.83
worker-2-foo ansible_host=34.65.192.69
worker-1-bar ansible_host=34.65.38.189

[master]
master-foo
master-bar

[worker]
worker-1-foo master=master-foo
worker-2-foo master=master-foo
worker-1-bar master=master-bar

[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

> Note: the `ansible_python_interpreter` variable might be necessary with the current version of Ansible if your target machines do only have Python 3, and not Python 2, installed.

### 2. Install kubeadm

Install kubeadm on all the nodes:

```bash
ansible-playbook playbooks/install-kubeadm.yml
```

### 3. Install Kubernetes on the master nodes

Run `kubeadm init` on the master nodes:

```bash
ansible-playbook playbooks/init.yml
```

This installs the Kubernetes control plane components on the master nodes and puts a kubeconfig file for each cluster in the `kubeconfig` directory in your current working directory.

You can now already access your clusters with kubectl as follows:

```bash
kubectl --kubeconfig kubeconfig/master-foo.conf get nodes
```

This should output a single node (the master node) in the `NotReady` state.

> The node is `NotReady` because there is no CNI plugin installed.

### 4. Install a CNI plugin

At this point, you [should install a CNI plugin](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#pod-network) in your clusters.

For example, you can install [Cilium](https://cilium.readthedocs.io/en/stable/gettingstarted/k8s-install-default/) in the `foo` cluster with:

```bash
kubectl --kubeconfig kubeconfig/master-foo.conf \
  apply -f https://raw.githubusercontent.com/cilium/cilium/1.7.0/install/kubernetes/quick-install.yaml
```

After installing a CNI plugin, the master node should become `Ready`.

> Installing the CNI plugin right after `kubeadm init` is recommended. However, you can also first run `kubeadm join` and only after that install a CNI plugin.

### 4. Install Kubernetes on the worker nodes

Run `kubeadm join` on the worker nodes:

```bash
ansible-playbook playbooks/join.yml
```

The worker nodes should now be part of the cluster. You can verify that with:

```bash
kubectl --kubeconfig kubeconfig/master-foo.conf get nodes
```

There should now be three nodes (all in the `Ready` state if you already installed a CNI plugin).

### 5. Cleaning up

If you want to start over with the installation of Kubernetes, you can run `kubeadm reset` on all the nodes:

```bash
ansible-playbook playbooks/reset.yml
```

This uninstalls all Kubernetes components from your nodes. You can now do a fresh install of Kubernetes with the `init.yml` and `join.yml` playbooks.

If you want to uninstall kubeadm on all the nodes, you can use:

```bash
ansible-playbook playbooks/remove-kubeadm.yml
```
