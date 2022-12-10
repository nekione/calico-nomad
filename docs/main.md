# Index

- [Index](#index)
- [calico-nomad](#calico-nomad)
  - [Steps to setup](#steps-to-setup)
    - [Step 1: Setup Nodes](#step-1-setup-nodes)
    - [Step 2: Install containerd and runc on both Nodes](#step-2-install-containerd-and-runc-on-both-nodes)
    - [Step 3: install CNI Plugins](#step-3-install-cni-plugins)
    - [Step 4: install nerdctl](#step-4-install-nerdctl)
    - [Step 5: install etcd cluster](#step-5-install-etcd-cluster)
    - [Step 6: installing Calicoctl, calico-plugins and calico-node](#step-6-installing-calicoctl-calico-plugins-and-calico-node)
- [Calico with Nomad](#calico-with-nomad)
  - [Steps to Setup](#steps-to-setup-1)
    - [Step 1: install Nomad](#step-1-install-nomad)
    - [Step 2: install Containerd Driver](#step-2-install-containerd-driver)
    - [Step 3: Run Sample test](#step-3-run-sample-test)
    - [Step 4: Lauch Container with Calico CNI](#step-4-lauch-container-with-calico-cni)
- [Automated Installation](#automated-installation)
  - [Run Nomad Job](#run-nomad-job)


# calico-nomad
> Calico is a CNI plugin that can be coupled with containerd which can scale up very easily across nodes. Calico is used by kubernetes to provide networking layer for pods. This documentation will help you setup Calico without kubernetes using etcd and containerd.

## Steps to setup 

### Step 1: Setup Nodes 

Machine Configuration:

```yaml
node1:
    cpu: 
        cores: 2
        threads: 2
    ram: 3Gib
    ipaddress: '172.16.6.10/16'
    gateway: '172.16.0.1'
    hostname: node1
    os: ubuntu-20.04-liveServer
    virtualization: bare-metal

node2:
    cpu: 
        cores: 2
        threads: 2
    ram: 3Gib
    ipaddress: '172.16.6.11/16'
    gateway: '172.16.0.1'
    hostname: node1
    os: ubuntu-20.04-liveServer
    virtualization: bare-metal
```

install basic tools: 
```bash
apt update && apt install vim net-tools curl wget -y
```

> **Skip to [Automated Installation](#automated-installation) if you want to run ansible playbook to setup the cluster.**

### Step 2: Install containerd and runc on both Nodes

[Official Guide](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

Download the `containerd-<VERSION>-<OS>-<ARCH>.tar.gz` archive from https://github.com/containerd/containerd/releases ,
verify its sha256sum, and extract it under `/usr/local`:

```console
wget https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz
```

```console
tar Cxzvf /usr/local containerd-1.6.8-linux-amd64.tar.gz 
```

Setup Service for containerd

copy and paste these lines to `/usr/local/lib/systemd/system/containerd.service` if directory not present use `mkdir -p /usr/local/lib/systemd/system`

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

enable containerd service

```bash
systemctl daemon-reload
systemctl enable --now containerd
```

installing runc

Download the runc.<ARCH> binary from https://github.com/opencontainers/runc/releases , verify its sha256sum, and install it as /usr/local/sbin/runc.

```console
wget https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
```

### Step 3: install CNI Plugins 

Download the cni-plugins-<OS>-<ARCH>-<VERSION>.tgz archive from https://github.com/containernetworking/plugins/releases , verify its sha256sum, and extract it under /opt/cni/bin:

```console
wget https://github.com/containernetworking/plugins/releases/download/v1.1.1/cni-plugins-linux-amd64-v1.1.1.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```

### Step 4: install nerdctl

Download nerdctl binary from release page https://github.com/containerd/nerdctl/releases/

```console
wget https://github.com/containerd/nerdctl/releases/download/v0.23.0/nerdctl-0.23.0-linux-amd64.tar.gz
tar -xvf nerdctl-0.23.0-linux-amd64.tar.gz
mv nerdctl /usr/local/bin/
chmod +x /usr/local/bin/nerdctl
```

test run nerdctl 
```console
nerdctl run -d --name nginx -p 80:80 nginx:alpine
```
to remove container use 
```console
nerdctl stop nginx
nerdctl rm nginx 
```

CNI testing with nerdctl [Official Guide](https://github.com/containerd/nerdctl/blob/master/docs/cni.md)

customize your CNI network by providing configuration files. For example you have one configuration file(`/etc/cni/net.d/10-mynet.conf`) for bridge network:

```json
{
  "cniVersion": "1.0.0",
  "name": "mynet",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "subnet": "172.19.0.0/24",
    "routes": [
      { "dst": "0.0.0.0/0" }
    ]
  }
}
```

This will configure a new CNI network with the name mynet, and you can use this network to create a container:

```console
nerdctl run -it --net mynet --rm alpine ip addr show
```

> note use this docker container for network tools https://github.com/nicolaka/netshoot

### Step 5: install etcd cluster
> Community Guide by https://github.com/justmeandopensource/kubernetes/blob/master/kubeadm-external-etcd/2%20simple-cluster-tls.md

**Generate TLS Certificates**
> Perform these actions on Node1 or Node2 then copy TLS certificates to other node
donwload cfssl binaries
```console
{
  wget -q --show-progress \
    https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssl \
    https://storage.googleapis.com/kubernetes-the-hard-way/cfssl/1.4.1/linux/cfssljson
  
  chmod +x cfssl cfssljson
  sudo mv cfssl cfssljson /usr/local/bin/
}
```

**generate CA Certificates**

```bash
{

cat > ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "8760h"
        },
        "profiles": {
            "etcd": {
                "expiry": "8760h",
                "usages": ["signing","key encipherment","server auth","client auth"]
            }
        }
    }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "etcd cluster",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "GB",
      "L": "India",
      "O": "calico",
      "OU": "ETCD-CA",
      "ST": "Nashik"
    }
  ]
}
EOF

cfssl gencert -initca ca-csr.json | cfssljson -bare ca

}
```

**Create TLS certificates**

```bash
{

ETCD1_IP="172.16.6.10"
ETCD2_IP="172.16.6.11"

cat > etcd-csr.json <<EOF
{
  "CN": "etcd",
  "hosts": [
    "localhost",
    "127.0.0.1",
    "${ETCD1_IP}",
    "${ETCD2_IP}"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "GB",
      "L": "England",
      "O": "Kubernetes",
      "OU": "etcd",
      "ST": "Cambridge"
    }
  ]
}
EOF

cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=etcd etcd-csr.json | cfssljson -bare etcd

}
```
transfer tls certificates

```bash
{

declare -a NODES=(172.16.6.11)

for node in ${NODES[@]}; do
  scp ca.pem etcd.pem etcd-key.pem root@$node: 
done

}
```

**perform these actions on all etcd nodes**
> Perform all commands logged in as root user or prefix each command with sudo as appropriate

**Copy the certificates to a standard location**

```bash
{
  mkdir -p /etc/etcd/pki
  mv ca.pem etcd.pem etcd-key.pem /etc/etcd/pki/ 
}
```

**Download etcd & etcdctl binaries from Github**
```bash
{
  ETCD_VER=v3.5.1
  wget -q --show-progress "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz"
  tar zxf etcd-v3.5.1-linux-amd64.tar.gz
  mv etcd-v3.5.1-linux-amd64/etcd* /usr/local/bin/
  rm -rf etcd*
}
```

**Create systemd unit file for etcd service**
> Set NODE_IP to the correct IP of the machine where you are running this

```bash
{

#NODE_IP="172.16.6.10"
NODE_IP="172.16.6.11"

ETCD_NAME=$(hostname -s)

ETCD1_IP="172.16.6.10"
ETCD2_IP="172.16.6.11"


cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/pki/etcd.pem \\
  --key-file=/etc/etcd/pki/etcd-key.pem \\
  --peer-cert-file=/etc/etcd/pki/etcd.pem \\
  --peer-key-file=/etc/etcd/pki/etcd-key.pem \\
  --trusted-ca-file=/etc/etcd/pki/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/pki/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${NODE_IP}:2380 \\
  --listen-peer-urls https://${NODE_IP}:2380 \\
  --advertise-client-urls https://${NODE_IP}:2379 \\
  --listen-client-urls https://${NODE_IP}:2379,https://127.0.0.1:2379 \\
  --initial-cluster-token etcd-cluster-1 \\
  --initial-cluster node1=https://${ETCD1_IP}:2380,node2=https://${ETCD2_IP}:2380 \\
  --initial-cluster-state new
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

}
```


 **Enable and Start etcd service**
```
{
  systemctl daemon-reload
  systemctl enable --now etcd
}
```

 **Verify Etcd cluster status**
> In any one of the etcd nodes
```
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/pki/ca.pem \
  --cert=/etc/etcd/pki/etcd.pem \
  --key=/etc/etcd/pki/etcd-key.pem \
  member list
```
Better to export these as environment variables and connect to the clutser instead of a specific node
```
export ETCDCTL_API=3 
export ETCDCTL_ENDPOINTS=https://172.16.6.10:2379,https://172.16.6.11:2379
export ETCDCTL_CACERT=/etc/etcd/pki/ca.pem
export ETCDCTL_CERT=/etc/etcd/pki/etcd.pem
export ETCDCTL_KEY=/etc/etcd/pki/etcd-key.pem
```
**And now its a lot easier**
```
etcdctl member list
etcdctl endpoint status
etcdctl endpoint health
```

### Step 6: installing Calicoctl, calico-plugins and calico-node

**Installing Calicoctl**

[Official Guide](https://projectcalico.docs.tigera.io/maintenance/clis/calicoctl/install)

download calicoctl binary

```bash
curl -L https://github.com/projectcalico/calico/releases/download/v3.24.1/calicoctl-linux-amd64 -o calicoctl
```

move file to `/usr/local/bin/` and set executable permissions
```bash
mv calicoctl /usr/local/bin/
chmod +x /usr/local/bin/calicoctl
```

**Install calico plugins**

[Official Guide](https://projectcalico.docs.tigera.io/getting-started/kubernetes/hardway/install-cni-plugin)

download plugin binaries
```bash
curl -L -o /opt/cni/bin/calico https://github.com/projectcalico/cni-plugin/releases/download/v3.14.0/calico-amd64
chmod 755 /opt/cni/bin/calico
curl -L -o /opt/cni/bin/calico-ipam https://github.com/projectcalico/cni-plugin/releases/download/v3.14.0/calico-ipam-amd64
chmod 755 /opt/cni/bin/calico-ipam
```

make cni configuration directory
```bash
mkdir -p /etc/cni/net.d/
```

add CNI config for calico

```bash
cat > /etc/cni/net.d/10-testnet.conflist <<EOF
{
  "name": "testnet",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "etcdv3",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      }
    }
  ]
}
EOF
```


**Setup Calico Node**

> note: delete calico node `calicoctl delete node node1 --allow-version-mismatch`

Run these commands on both machines

Create calicoctl config file at `/etc/calico/calicoctl.cfg`
```yaml
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  etcdEndpoints: https://172.16.6.10:2379,https://172.16.6.11:2379
  etcdKeyFile: /etc/etcd/pki/etcd-key.pem
  etcdCertFile: /etc/etcd/pki/etcd.pem
  etcdCACertFile: /etc/etcd/pki/ca.pem
```

start calico node with nerdctl
```bash
mkdir -p /var/log/calico
mkdir -p /var/run/calico
mkdir -p /var/lib/calico

nerdctl run --net=host --privileged --name=calico-node -d --restart=always -e ETCD_DISCOVERY_SRV= \
-e ETCD_CA_CERT_FILE=/etc/calico/certs/ca_cert.crt \
-e ETCD_KEY_FILE=/etc/calico/certs/key.pem \
-e ETCD_CERT_FILE=/etc/calico/certs/cert.crt \
-e NODENAME=$(hostname -s) \
-e CALICO_NETWORKING_BACKEND=bird \
-e ETCD_ENDPOINTS=https://172.16.6.10:2379,https://172.16.6.11:2379 \
-v /var/log/calico:/var/log/calico \
-v /var/run/calico:/var/run/calico \
-v /var/lib/calico:/var/lib/calico \
-v /lib/modules:/lib/modules \
-v /run:/run \
-v /etc/etcd/pki/ca.pem:/etc/calico/certs/ca_cert.crt \
-v /etc/etcd/pki/etcd-key.pem:/etc/calico/certs/key.pem \
-v /etc/etcd/pki/etcd.pem:/etc/calico/certs/cert.crt \
quay.io/calico/node:latest
```

check if nodes are launched
```console
node1$ calicoctl get node -o wide --allow-version-mismatch
NAME    ASN       IPV4             IPV6   
node1   (64512)   172.16.6.10/16          
node2   (64512)   172.16.6.11/16   
```

add alias to .bashrc to run without allow-version-mismatch
```
alias calico='calicoctl --allow-version-mismatch'
```
now it becomes more simple
```console
node1$ calico get nodes
NAME    
node1   
node2 
```

export environment variable
```bash
export DATASTORE_TYPE=etcdv3
export ETCD_ENDPOINTS=https://172.16.6.10:2379,https://172.16.6.11:2379
export ETCD_CA_CERT_FILE=/etc/etcd/pki/ca.pem
export ETCD_CERT_FILE=/etc/etcd/pki/etcd.pem
export ETCD_KEY_FILE=/etc/etcd/pki/etcd-key.pem
```

launch container with testnet network

```bash
nerdctl run -it --net testnet nicolaka/netshoot /bin/bash
```
 
# Calico with Nomad

## Steps to Setup

### Step 1: install Nomad
[Official Guide](https://learn.hashicorp.com/tutorials/nomad/get-started-install?in=nomad/get-started)

**Add Hashicorp GPG key**
```bash
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
```
**Add official repository**
```bash
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
```
**Update and Install**
```bash
sudo apt-get update && sudo apt-get install nomad
```

### Step 2: install Containerd Driver

> Note: Make sure containerd and cni plugins are installed properly
> Github repo for [Roblox Containerd-Driver](https://github.com/Roblox/nomad-driver-containerd/)
```bash
mkdir -p /opt/nomad/data/plugins && cd /opt/nomad/data/plugins
curl https://github.com/Roblox/nomad-driver-containerd/releases/download/v0.9.3/containerd-driver
```

add these lines to `/etc/nomad.d/nomad.hcl`

```groovy
# Full configuration options can be found at https://www.nomadproject.io/docs/configuration

data_dir  = "/opt/nomad/data"
bind_addr = "0.0.0.0"

server {
  # license_path is required as of Nomad v1.1.1+
  #license_path = "/opt/nomad/license.hclic"
  enabled          = true
  bootstrap_expect = 1
}

# Insert from here

plugin "containerd-driver" {
  config {
    enabled = true
    containerd_runtime = "io.containerd.runc.v2"
    stats_interval = "5s"
  }
}

# Upto here

client {
  enabled = true
  servers = ["127.0.0.1"]
}
```

**Add env file for nomad service in** `/etc/nomad.d/nomad.env`
```yaml
ETCD_ENDPOINTS=https://172.16.6.10:2379, https://172.16.6.11:2379
ETCD_KEY_FILE=/etc/etcd/pki/etcd-key.pem
ETCD_CERT_FILE=/etc/etcd/pki/etcd.pem
ETCD_CA_CERT_FILE=/etc/etcd/pki/ca.pem
```

**Reload nomad service**
```bash
systemctl daemon-reload
systemctl restart nomad
```

### Step 3: Run Sample test

save this file as example.nomad

```groovy
job "nginx" {
    datacenters = ["dc1"]
    group "nginx-group" {
        reschedule {
            attempts = 1
            interval = "1h"
            delay = "10s"
            unlimited = false
        }

        task "nginx-task" {
            driver = "containerd-driver"
            config {
                image = "docker.io/library/nginx:alpine"
            }
            resources {
                cpu = 500
                memory = 256
            }
        }

        network {
            mode = "bridge"
            port "http" {
                to = "80"
            }
            port "https" {
                to = "443"
            }
        }
    }
}
```
**Run Nomad Job**
```console
$ nomad job run example.nomad

==> 2022-09-15T05:26:08Z: Monitoring evaluation "88e8738b"
    2022-09-15T05:26:08Z: Evaluation triggered by job "nginx"
    2022-09-15T05:26:09Z: Evaluation within deployment: "5476f47f"
    2022-09-15T05:26:09Z: Allocation "dede76be" created: node "496f4ddf", group "nginx-group"
    2022-09-15T05:26:09Z: Evaluation status changed: "pending" -> "complete"
==> 2022-09-15T05:26:09Z: Evaluation "88e8738b" finished with status "complete"
==> 2022-09-15T05:26:09Z: Monitoring deployment "5476f47f"
  ✓ Deployment "5476f47f" successful
    
    2022-09-15T05:26:25Z
    ID          = 5476f47f
    Job ID      = nginx
    Job Version = 0
    Status      = successful
    Description = Deployment completed successfully
    
    Deployed
    Task Group   Desired  Placed  Healthy  Unhealthy  Progress Deadline
    nginx-group  1        1       1        0          2022-09-15T05:36:23Z
```

### Step 4: Lauch Container with Calico CNI

**Create a CNI config file in** `/opt/cni/config/10-testnet.conflist`
```bash
mkdir -p /opt/cni/config
touch /opt/cni/config/10-testnet.conflist
```
add these lines to cni config

```groovy
{
  "name": "testnet",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "calico",
      "log_level": "info",
      "datastore_type": "etcdv3",
      "mtu": 1500,
      "ipam": {
          "type": "calico-ipam"
      }
    }
  ]
}
```

**Make sure to restart Nomad service**

```bash
systemctl restart nomad
```

**Create new nomad Job with the given config**

```groovy
job "netshoot-1" {
    datacenters = ["dc1"]
    group "netshoot-group" {
        reschedule {
            attempts = 1
            interval       = "1h"
            delay = "10s"
            unlimited      = false
        }

        task "netshoot-task-1" {
            driver = "containerd-driver"
            config {
                image = "nicolaka/netshoot"
                command = "sh"
                args = ["-c", "while true; do echo 'hello'; sleep 5; done"]
            }
            resources {
                cpu = 500
                memory = 256
            }
        }

        network {
            mode = "cni/testnet"
                port "http" {
                        to = 8080
                }
        }
    }
}
```
# Automated Installation

> clone this repository and run these commands

create virtual environment
```bash
python3 -m virtualenv venv
source venv/bin/activate
```

install dependencies
```bash
pip install -r requirements.txt
```

run the playbook
```bash
ansible-playbook -i hosts setup.yml
```

## Run Nomad Job

```bash
nomad job run example.nomad
```