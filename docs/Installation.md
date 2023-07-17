# K3s Installation
K3s - Lightweight Kubernetes. Easy to install, half the memory, all in a binary of less than 100 MB.
- [K3s Installation](#k3s-installation)
  - [Install Script](#install-script)
  - [Check the Status of K3s](#check-the-status-of-k3s)

## Install Script
K3s provides an installation script that is a convenient way to install it as a service on systemd or openrc based systems. This script is available at https://get.k3s.io. To install K3s you can easily follow the [Quick Start Guide](https://docs.k3s.io/quick-start).

After running this installation:
* The K3s service will be configured to automatically restart after node reboots or if the process crashes or is killed
* Additional utilities will be installed, including kubectl, crictl, ctr, k3s-killall.sh, and k3s-uninstall.sh
* A kubeconfig file will be written to /etc/rancher/k3s/k3s.yaml and the kubectl installed by K3s will automatically use it

For detailed information about the installation process and other installation options, please refer to the [official K3s website](https://docs.k3s.io/).

## Check the Status of K3s
To check that the installation of the K3s service was successful and the Kubernetes cluster is running, execute the following command:
```bash
sudo kubectl get nodes
```
You should see a similar output:
```bash
NAME      STATUS   ROLES                  AGE    VERSION
siemens   Ready    control-plane,master   1h9m   v1.27.3+k3s1
```
