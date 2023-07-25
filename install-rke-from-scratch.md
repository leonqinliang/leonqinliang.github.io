# Installing RKE Cluster From Scratch

## Step 1: Create VirtualBox VMs

### Hardware Requirements

- 2 CPUs

- 2GB RAM

- 15GB disk space for OS

- choose "NAT Networks" for VM network type

### OS Requirements

- choose "openSUSE" as OS type

- install openSUSE LEAP

- disable firewall service at installation

## Step 2: Configure Linux OS

### Update OS

```bash
zypper up
```

### Disable SELinux

**To do**

### Disable Firewall(if not)

```bash
systemctl disable firewalld.service
```

```bash
systemctl stop firewalld.service
```

### Enable IPv4 Forwarding and Iptables can see the bridged traffic

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay
```

```bash
sudo modprobe br_netfilter
```

### Sysctl Parameters Required by Setup

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
```

### Apply Sysctl Parameters without Reboot

```bash
sudo sysctl --system
```

### Install Containerd

Find latest version of containerd at [this github repository](https://github.com/containerd/containerd/releases).

Download the gz file and run:

```bash
tar Cxzvf /usr/local containerd-1.7.2-linux-amd64.tar.gz
```

Download [the service file](https://raw.githubusercontent.com/containerd/containerd/main/containerd.service). The actual content is as below. You can copy and paste directly into a file named *'containerd.service'*.

```yaml
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
#uncomment to enable the experimental sbservice (sandboxed) version of containerd/cri integration
#Environment="ENABLE_CRI_SANDBOXES=sandboxed"
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

Make a directory for containerd

```bash
mkdir -p /usr/local/lib/systemd/system/
```

Copy the service file into the directory

```bash
cp containerd.service /usr/local/lib/systemd/system/
```

Start containerd service

```bash
systemctl daemon-reload
```

```bash
systemctl enable --now containerd.service
```

### Install runc

Download the binary from [https://github.com/opencontainers/runc/releases](https://github.com/opencontainers/runc/releases)

Use below commands to install:

```bash
install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Disable Swap

```bash
swapoff -a
```

Make sure the swap is disabled

```bash
vim /etc/fstab
```

Mark the swap space with comment mark '#'

### Install CNI Plugin

Download the latest version [here](https://github.com/containernetworking/plugins/releases/).

Make a directory for install CNI plugin

```bash
mkdir -p /opt/cni/bin/
```

Install CNI plugin

```bash 
sudo tar -C /opt/cni/bin/ -xzf cni-plugins-linux-amd64-v1.3.0.tgz
```

### Configure cgroup driver

Generate default configuration file

```bash
containerd config default > /etc/containerd/config.toml
```

Set "systemd" as cgroup driver

```bash
vim /etc/containerd/config.toml
```

```bashag-0-1h5tofl63ag-1-1h5tofl63
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
 ...
 [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
 SystemdCgroup = true
```

then restart containerd service

```bash
systemctl restart containerd.service
```

### Configure ssh auto authentication

Generating public/private key pair

```bash
ssh-keygen
```

Copy public key to worker nodes

```bash
ssh-copy-id root@rke-worker01
```

Verify the auto-authentication function

```bash
ssh root@rke-worker01
```

## Step 3: Install RKE


