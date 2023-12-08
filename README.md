K8test
======

Installs a Kubernetes cluster at cloudscale.ch for integration tests.

⛔️ This is not a production-ready Kubernetes cluster and should not be used as such. The cluster is neither secure, nor resilient. We only use it for short-lived test environments focused on integration tests.

## Kubernetes Setup

The Kubernetes cluster is installed using kubeadm as documented on the official homepage: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

We use the "Without a package manager" approach, to be able to run this on any Linux with a modern-ish kernel and systemd present.

The Kubernetes cluster that is started has the following properties:

- The first control's public IP address is the control plane endpoint.
- Uses CRI-O as container runtime.
- Uses Cilium as CNI plugin.
- Includes helm and k9s on the controls.

## Requirements

To use k8test, you need git, a recent Python 3 release, Linux/macOS, and an API token for cloudscale.ch.

## Installation

Before using k8test, you need to install it as follows:

```bash
python3 -m venv venv
source venv/bin/activate
pip install poetry
poetry install

export CLOUDSCALE_API_TOKEN=...
```

## Usage

To create a Kubernetes cluster, use the `playbooks/create-cluster.yml`. Once created, the cluster can be removed using `playbooks/destroy-cluster.yml`.

Other playbooks can be used for various tasks:

- `playbooks/deploy-chart.yml` - deploy a local Helm chart on the cluster.
- `playbooks/update-secrets.yml` - update cloudscale.ch secrets on the cluster.

### Cluster Creation

To create a new cluster, run the creation playbook as follows (no need for `ansible-playbook`, the playbooks are executable).

```bash
playbooks/create-cluster.yml \
  -e kubernetes=<kubernetes release to install, defaults to the latest one> \
  -e cluster_prefix=<string in front of all hostnames, defaults to k8test> \
  -e zone=<rma1|lpg1> \
  -e ssh_key=<path-to-public-ssh-key> \
  -e control_count=<number of controls, defaults to 1> \
  -e worker_count=<number of workers, defaults to 2> \
  -e image=<image slug to use, defaults to ubuntu-22.04> \
  -e flavor=<flavor to use, defaults to flex-8-4> \
  -e volume_size_gb=<root volume size, defaults to 25> \
  -e kubelet_extra_args<passed to kubelet, not defined by default>
```

The Kubernetes version installed is the latest one found. You can specify another version, either exactly (e.g. `1.27.1`, or roughly (`1.27`). To see the exact set of versions used, run `helpers/release-set <version>`.

There's currently no easy way to re-use the same exact set of versions for each run, but it would be possible to add that functionality.

Once the cluster is created, you'll find the following files in the `cluster` directory:

- `cluster/inventory.yml` - inventory of the created servers.
- `cluster/admin.conf` - kubeconfig of the cluster.

Note that running the `create-cluster.yml` playbook against an existing cluster may or may not work as expected: It's not a goal of this project to support that - these clusters are meant to be short-lived.

### Cluster Removal

To delete the created cluster, run the following command:

```bash
playbooks/destroy-cluster.yml -i cluster/inventory.yml
```

### Update Secrets

To store/update the `CLOUDSCALE_API_TOKEN` of the current environment in a well-known location in the cluster, run the following command:

```bash
playbooks/update-secrets.yml -i cluster/inventory.yml
```

### Deploy a Chart

To deploy a local chart, you can use the following playbook. It will delete the old deployment of the same name, copy the chart to the control, and execute it:

```bash
playbooks/deploy-chart.yml -i cluster/inventory.yml \
  -e chart=<local chart directory>
  -e ns=<namespace> \
  -e deployment=<name of the deployment> \
  -e extra=<additional arguments passed to helm>
```

Note that spaces in `-e extra` have to be quoted. For example:

```bash
playbooks/deploy-chart.yml \
  -i cluster/inventory.yml \
  -e ns='kube-system' \
  -e deployment='csi' \
  -e chart=./charts/csi-cloudscale \
  -e extra='"--set controller.image.tag=foo --set controller.image.tag=foo"'
```

### Build a Container Image

This playbook allows building a Dockerfile on the nodes in a way that makes the resulting image available on the cluster, without the need for an external registry.

This is done by installing Podman, which uses the same underlying storage as CRI-O, and just building an image locally (no push needed).

The resulting image can be used with `imagePullPolicy: IfNotPresent`, provided that the image tag is not `latest` (which always causes a pull).

```bash
playbooks/build-image.yml -i cluster/inventory.yml \
  -e dockerfile=<path to local Dockerfile> \
  -e tag=<tag to apply> \
  -e extra=<extra args for podman>
```

Just like with the `extra` in `playbooks/deploy-chart.yml`, quotes are necessary if there are spaces:

```bash
/playbooks/build-image.yml -i cluster/inventory.yml \
  -e dockerfile=./Dockerfile \
  -e tag=quay.io/cloudscalech/cloudscale-cloud-controller-manager:test \
  -e extra='"--build-arg=BAR=foo --build-arg=FOO=bar"'
```
