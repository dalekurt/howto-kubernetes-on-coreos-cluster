Install and Configure Kubernetes on CoreOS Cluster
==================================================

Launch CoreOS Cluster on Amazon EC2
-----------------------------------

[CoreOS CloudFormation Templates  ][1]

[1]: <https://coreos.com/docs/running-coreos/cloud-providers/ec2/ >

Create the Flannel Network Fabric Layer
---------------------------------------

`sudo mkdir -p /opt/bin`

Start GO Container

```
docker run -i -t google/golang /bin/bash -c "go get github.com/coreos/flannel"
```

Get the contaner ID

```
docker ps -l -q
```

The result will be the ID that looks like this:

```
c0a6dda1ccb5
```

We can use this ID to specify a copy operation into the /opt/bin directory. The
binary has been placed at `/gopath/bin/flannel` within the container

```
sudo docker cp c0a6dda1ccb5:/gopath/bin/flannel /opt/bin/
```

 

Build the Kubernetes Binaries
-----------------------------

The first step is to clone the project from its GitHub repository. We will clone
it into our home directory:

```
cd ~
git clone https://github.com/GoogleCloudPlatform/kubernetes.git
```

 

Build the binary files

```
cd kubernetes/build/
./make-build-image.sh
./run.sh hack/build-go.sh
```

You will be prompt to download a local copy of the golang docker image. This may
take awhile

```
+++ [0612 04:43:57] Verifying Prerequisites....
You don't have a local copy of the golang docker image. This image is 450MB.
Download it now? [y/n] y
```

When the build process is completed, you will be able to find the binaries in
the `~/kubernetes/_output/dockerized/bin/linux/amd64` directory:

```
cd ~/kubernetes/_output/dockerized/bin/linux/amd64
ls
```

```
e2e          kube-apiserver           kube-proxy      kubecfg  kubelet
integration  kube-controller-manager  kube-scheduler  kubectl  kubernetes
```

We will transfer these to the /opt/bin directory that we created earlier:

```
sudo cp * /opt/bin
```

 

Transfer Binaries to the other CoreOS servers
---------------------------------------------

```
cd /opt/bin
```

```
tar -czf - . | ssh core@172.17.8.102 "sudo mkdir -p /opt/bin; cd /opt/bin; sudo tar xzvf -"
```

You now have all of the binaries in place on your three machines.

 

Setting up Master-Specific Services
-----------------------------------

We will be placing these files in the /etc/systemd/system directory.

```
cd /etc/systemd/system
```

Now we can begin building our service files. We will create two files on master
only and five files that also belong on the minions. All of these files will be
in `/etc/systemd/system/*.service`.

 

### Master files:

-   apiserver.service

-   controller-manager.service

### Minion files for all servers:

-   scheduler.service

-   flannel.service

-   docker.service

-   proxy.service

-   kubelet.service

 

Create the API Server Unit File
-------------------------------

```
sudo vim apiserver.service
```

 

```
[Unit]
Description=Kubernetes API Server
After=etcd.service
After=docker.service
Wants=etcd.service
Wants=docker.service

[Service]
ExecStart=/opt/bin/kube-apiserver \
-address=127.0.0.1 \
-port=8080 \
-etcd_servers=http://127.0.0.1:4001 \
-portal_net=10.100.0.0/16 \
-logtostderr=true
ExecStartPost=-/bin/bash -c "until /usr/bin/curl http://127.0.0.1:8080; do echo \"waiting for API server to come online...\"; sleep 3; done"
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

 

Create the Controller Manager Unit File
---------------------------------------

```
sudo vim controller-manager.service
```

 

```
[Unit]
Description=Kubernetes Controller Manager
After=etcd.service
After=docker.service
After=apiserver.service
Wants=etcd.service
Wants=docker.service
Wants=apiserver.service

[Service]
ExecStart=/opt/bin/kube-controller-manager \
-master=http://127.0.0.1:8080 \
-machines=172.17.8.101,172.17.8.102,172.17.8.103
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

 

Setting Up Cluster Services
---------------------------

 

These five files should be created on all machines,
in `/etc/systemd/system/*.service`.

-   `scheduler.service`

-   `flannel.service`

-   `docker.service`

-   `proxy.service`

-   `kubelet.service`

 

Create the Scheduler Unit File
------------------------------

 

```
sudo vim scheduler.service
```

 

```
[Unit]
Description=Kubernetes Scheduler
After=etcd.service
After=docker.service
After=apiserver.service
Wants=etcd.service
Wants=docker.service
Wants=apiserver.service

[Service]
ExecStart=/opt/bin/kube-scheduler -master=127.0.0.1:8080
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

 

Create the Flannel Unit File
----------------------------

 

```
cd /etc/systemd/system
```

 

```
sudo vim flannel.service
```

 

```
[Unit]
Description=Flannel network fabric for CoreOS
Requires=etcd.service
After=etcd.service

[Service]
EnvironmentFile=/etc/environment
ExecStartPre=-/bin/bash -c "until /usr/bin/etcdctl set /coreos.com/network/config '{\"Network\": \"10.100.0.0/16\"}'; do echo \"waiting for etcd to become available...\"; sleep 5; done"
ExecStart=/opt/bin/flannel -iface=${COREOS_PRIVATE_IPV4}
ExecStartPost=-/bin/bash -c "until [ -e /run/flannel/subnet.env ]; do echo \"waiting for write.\"; sleep 3; done"
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

 

Create the Docker Unit File
---------------------------

 

```
sudo vim docker.service
```

 

```
[Unit]
Description=Docker container engine configured to run with flannel
Requires=flannel.service
After=flannel.service

[Service]
EnvironmentFile=/run/flannel/subnet.env
ExecStartPre=-/usr/bin/ip link set dev docker0 down
ExecStartPre=-/usr/sbin/brctl delbr docker0
ExecStart=/usr/bin/docker -d -s=btrfs -H fd:// --bip=${FLANNEL_SUBNET} --mtu=${FLANNEL_MTU}
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

 

Create the Proxy Unit File
--------------------------

 

```
sudo vim proxy.service
```

 

```
[Unit]
Description=Kubernetes proxy server
After=etcd.service
After=docker.service
Wants=etcd.service
Wants=docker.service

[Service]
ExecStart=/opt/bin/kube-proxy -etcd_servers=http://127.0.0.1:4001 -logtostderr=true
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

 

Create the Kubelet Unit File
----------------------------

 

```
sudo vim kubelet.service
```

 

```
[Unit]
Description=Kubernetes Kubelet
After=etcd.service
After=docker.service
Wants=etcd.service
Wants=docker.service

[Service]
EnvironmentFile=/etc/environment
ExecStart=/opt/bin/kubelet \
-address=${COREOS_PRIVATE_IPV4} \
-port=10250 \
-hostname_override=${COREOS_PRIVATE_IPV4} \
-etcd_servers=http://127.0.0.1:4001 \
-logtostderr=true
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

 

Enabling the Services
---------------------

```
cd /etc/systemd/system
sudo systemctl enable *
```

This will create a `multi-user.target.wants` directory with symbolic links to
our unit files. This directory will be processed by `systemd` toward the end of
the boot process.

 

Repeat this step on each of your servers.

 

Now reboot the master

```
sudo reboot
```

 

Once the master comes back online, you can reboot your minion servers:

```
sudo reboot
```

 

Once all of your servers are online, make sure your services started correctly.
You can check this by typing:

```
systemctl status service_name
```

 

Or you can go the journal by typing:

```
journalctl -b -u service_name
```

 

Look for an indication that the services are up and running correctly. If there
are any issues, a restart of the specific service might help:

```
sudo systemctl restart service_name
```

 

Check that all of the servers are available by typing:

 

```
kubectl list minions
```

 

```
Minion identifier
----------
172.17.8.101
172.17.8.102
172.17.8.103
```
