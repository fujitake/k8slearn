## Introducion
With following steps, you will be able to get a Kubernetes Cluster.
The later part of this procedure referred to other article.

Creating a Kubernetes Cluster at your own hand would help you understanding how it works without keep paying for a cloud services, so I thought I'd try to write up [Create Kubernetes Cluster with Raspberry Pi](https://github.com/fujitake/k8slearn/blob/main/docs/eng/configure_k3s_w_rasppi.md). This document is for the Notebook PC version (X86-64 version in terms of container image).

*It is recommended to read other articles to understand the basics of Kubernetes itself, Pod, Deployment, StatefulSet, Namespace, CNI, etc. Finally, you need to know those words, if you want to understand Kubernetes as a Cluster.*

## Purpose
Selected K3s because we wanted to reduce the learning cost as much as possible. You don't need worry about installing the original Kubernetes and which CNI you should use.
When I started to learn Kubernetes, I followed step by step and installed Kubernetes manually, but I strongly suggest you to learn with below steps, if you want to learn Kubernetes as a cluster. If you are ok to use as a single node, Minikube or Docker for Desktop would be enough.

## Prepare the Environment
- Internet access environment
- Knowledge of Linux using CLI commands

## Preparations
### Hardware
- Lenovo E460 Intel Core i5-6200U 2 core/4 thread 16GB Memory  
*Better to use a PC that you DON'T have a plan to use in the future.*
- Wired LAN
- MAC or Windows (Host for OS image creation and Host for SSH access)

### Software
- OS: [Ubuntu Server 20.04.2 LTS 64 bits](https://ubuntu.com/download/server)
- OS Image Creation Tool: [UNetbootin](https://unetbootin.github.io)
- K3s: [v1.20.6+k3s1](https://k3s.io)  
OK to install native Kubernetes, but using K3s would be the best for learning without spending lots of time to dealing with details like what kind of CNI that you want to use, etc.

## Preparations
### Create OS image
- Create a USB recovery drive, etc., to restore the Notebook PC to its factory settings.  
Usually, required a USB recovery drive or similar device, if you want to recover the factory OS (e.g. Windows). If you forgot to prepare it, you may require to **ask the manufacturer for paid support** to get your Notebook PC back to normal. **Please do so with caution**.
- Download Ubuntu Server from the link above
- USB memory: 2GB or more (2GB is not enough when installing Desktop, etc.)
- Create a bootable USB memory with UNetbootin
Specify the OS image file (ISO file) that is downloaded in advance, specify the USB drive, and click OK.
USB drive should be FAT32 formatted beforehand to show up as an option.
![unetbootin.png](../../imgs/unetbootin.png)

### Boot and OS Setup
- Select to boot from the USB flash drive created above in the UEFI/BIOS boot settings at PC startup  
`It is still possible to step back. Is it ok to discard your DATA and OS?`
- Answer the various questions in the installation step and complete the installation  
- Modify the network configuration file (if necessary)

```shell:cloud-init.yaml
sudo cp /etc/netplan/50-cloud-init.yaml /etc/netplan/99-cloud-init.yaml # Copy the file
sudo vi /etc/netplan/99-cloud-init.yaml
sudo netplan apply # Applying setting changes
```
- Modify 99-cloud-init.yaml

```shell:Example of modification
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets: # Wired LAN Settings
        eth0:
            dhcp4: true
            optional: true
    wifis: # Wireless LAN Settings
        wlan0:
            dhcp4: true # For static IP, set to false
            optional: true
            access-points:
              <yourssid>: # Provide the SSID in the <youssid> field, followed by a colon and a new line
                password: “<yourpassword>” # Enclose the password in double quotes

```
- Initial login

```shell:Initial login
Username: ubuntu
Password: ubuntu # Prompted to change it after logging in
```
- Keyboard settings

```shell:Keyboard settings
sudo dpkg-reconfigure keyboard-configuration
```
- Check other settings

```shell:Check other settings
sudo ufw status
Status: inactive # If it is active, sort out the communication requirements of k3s and set it appropriately
```
## K3s Installation
Preparations of the foundation has been completed, but the installation is almost completed.

### K3s Master Node installation - 1st unit
According to the [K3s](https://k3s.io), the procedure is super simple. Run the install command, and wait a moment (30 seconds?),  then use the ```k3s kubectl get node``` to confirm that the node is ready. This is true. Compared to the native K8s, there is no other word for it but awesome.  
But here is what we want to do, so let's proceed with options.

```shell:installation command
curl -sfL https://get.k3s.io | sh -s - --write-kubeconfig-mode 644
[sudo] password for ubuntu:
[INFO]  Finding release for channel stable
[INFO]  Using v1.20.6+k3s1 as release
[INFO]  Downloading hash https://github.com/k3s-io/k3s/releases/download/v1.20.6+k3s1/sha256sum-amd64.txt
[INFO]  Downloading binary https://github.com/k3s-io/k3s/releases/download/v1.20.6+k3s1/k3s
[INFO]  Verifying binary download
[INFO]  Installing k3s to /usr/local/bin/k3s
[INFO]  Creating /usr/local/bin/kubectl symlink to k3s
[INFO]  Creating /usr/local/bin/crictl symlink to k3s
[INFO]  Creating /usr/local/bin/ctr symlink to k3s
[INFO]  Creating killall script /usr/local/bin/k3s-killall.sh
[INFO]  Creating uninstall script /usr/local/bin/k3s-uninstall.sh
[INFO]  env: Creating environment file /etc/systemd/system/k3s.service.env
[INFO]  systemd: Creating service file /etc/systemd/system/k3s.service
[INFO]  systemd: Enabling k3s unit
Created symlink /etc/systemd/system/multi-user.target.wants/k3s.service → /etc/systemd/system/k3s.service.
[INFO]  systemd: Starting k3s

```
Check the status of the Master Node (in K3s, it is called Server Node, but here it is called Master) and that the Pod has been deployed.

```shell:check node status
kubectl get nodes
NAME   STATUS   ROLES                  AGE   VERSION
e460   Ready    control-plane,master   21m   v1.20.6+k3s1
```

```shell:check pods' status
kubectl get pods -A
NAMESPACE     NAME                                      READY   STATUS      RESTARTS   AGE
kube-system   helm-install-traefik-tw4tp                0/1     Completed   0          21m
kube-system   local-path-provisioner-5ff76fc89d-p5vsf   1/1     Running     0          21m
kube-system   metrics-server-86cbb8457f-k7htz           1/1     Running     0          21m
kube-system   svclb-traefik-7lwf4                       2/2     Running     0          21m
kube-system   coredns-854c77959c-hxm94                  1/1     Running     0          21m
kube-system   traefik-6f9cbd9bd4-4snp5                  1/1     Running     0          21m
```
Now that you have set up the first Server Node (Master Node). For use with a single unit, this is all you need to do.

## K3s Worker Node installation
Here is the setup for the cluster configuration.
To read more, please refer to [Article on building a Kubernetes Cluster using a Raspberry Pi](https://github.com/fujitake/k8slearn/blob/main/docs/eng/configure_k3s_w_rasppi.md#5-cluster-configuration-of-k3s).

*This procedure is for adding Worker Node configuration. You can skip it, if you want to use the Master Node as a single configuration.*

## Note
How to Uninstall K3s

```shell:K3s uninstallaton command
/usr/local/bin/k3s-uninstall.sh       # For Master Node
/usr/local/bin/k3s-agent-uninstall.sh # For Worker Node
```

## References
[RANCHER's K3s Reference](https://rancher.com/docs/k3s/latest/en/)  
[RANCHER Labs K3s Manual (Japanese)](https://rancher.co.jp/pdfs/K3s-eBook4Styles0507.pdf)
