# Introduction
Why is that every "Kubernetes getting started" guide takes tens of manuals steps? There has to be a simpler way.

Ansinetes (Ansible + Kubernetes) is a single command that lets you build multi-node high-availability clusters on CoreOS (on Vagrant or in your datacenter).
Ansinetes may not exhibit Ansible best practices but is a good starting point to (re)create non-trivial clusters with complex configuration easily.

It has already taken some decisions for you:
* Uses CoreOS
* Installs Kubernetes from the binary relase on Github
* Uses flannel for the pod/overlay network
* Configures TLS everywhere possible

# Prerequisites
* Linux or Mac
* Docker running locally (there will be volumes being host-mounted)
* Vagrant (only if you're going to use it)

# Install
```bash
$ curl -Ls -O https://raw.githubusercontent.com/jvassev/ansinetes/master/ansinetes
$ chmod +x ansinetes
```

This script will pull an image from dockerhub with Ansible 2.x installed. On first run it will populate a local directory with playbooks and their supporting resources.

# Booting a sensible Kubernetes cluster

## Enter the interactive Ansible environment
Pick a project name and start ansinetes. A directory named after the project will be created:

```bash
$ ./ansinetes -p demo
./ansinetes -p demo
 *** First run, copying default configuration
 *** Generating ssh key...
 *** Use this key for ssh:
ssh-rsa ... ansible-generated/initial
 *** Generating config.rb for Vagrant

[ansinetes@demo ~]$
```
You are dropped into a bash session (in a container) with ansible already installed and preconfigured - no need to install anything. The project directory already contains some default configuration as well as the Ansible playbooks. The `-p` parameter expects a path to a folder and will create one if it doesn't exist.

## Start the Vagrant environment
In another shell navigate to demo/vagrant and start the VMs. Four VMs with 1GB of memory are started:
```bash
$ cd demo/vagrant
$ vagrant up
==> core-01: Importing base box 'coreos-beta'...
==> core-01: Matching MAC address for NAT networking...
==> core-01: Checking if box 'coreos-beta' is up to date.
...
```

## Preparing CoreOS
CoreOS is usually configured using cloud-config but as a dev you are going to iterate faster if you stop/reconfigure/start your services, not the VMs. CoreOS needs to first be made [Ansible-compatible](https://galaxy.ansible.com/defunctzombie/coreos-bootstrap/). Luckily, this can be accomplished within Ansible itself.

```bash
[ansinetes@demo ~]$ ansible-playbook /etc/ansible/books/coreos-bootstrap.yml

PLAY [coreos] ******************************************************************
TASK [defunctzombie.coreos-bootstrap : Check if bootstrap is needed] ***********
TASK [defunctzombie.coreos-bootstrap : Run bootstrap.sh] ***********************
TASK [defunctzombie.coreos-bootstrap : Check if we need to install pip] ********
TASK [defunctzombie.coreos-bootstrap : Copy get-pip.py] ************************
TASK [defunctzombie.coreos-bootstrap : Install pip] ****************************
TASK [defunctzombie.coreos-bootstrap : Remove get-pip.py] **********************
TASK [defunctzombie.coreos-bootstrap : Install pip launcher] *******************
TASK [Ensure custom facts directory exists] ************************************
TASK [Install vmware custom facts] *********************************************
TASK [Collect facts] ***********************************************************
```

## Create a CA
Certificates are created using [cfssl](https://github.com/cloudflare/cfssl). You can edit the demo/security/*.json files before creating the CA. The CA will not be auto-created to give you a chanse to configure it. Defaults are pretty good though (at least for development). The json files are in the cfssl format. Finally, run the `kt-ca-init` command:

```bash
[ansinetes@demo ~]$ kt-ca-init
2016/09/02 20:49:24 [INFO] generating a new CA key and certificate from CSR
2016/09/02 20:49:24 [INFO] generate received request
2016/09/02 20:49:24 [INFO] received CSR
2016/09/02 20:49:24 [INFO] generating key: rsa-2048
2016/09/02 20:49:25 [INFO] encoded CSR
2016/09/02 20:49:25 [INFO] signed certificate with serial number 4640577027862920487
```

The ca.pem and ca-key.pem will be used for both etcd and Kubernetes later. The kt-* scripts are thin wrappers around cfssl and cfssljson.

## Configure etcd
Etcd is essential both to Kubernetes and flannel. Ansinetes will deploy a 3-node cluster with the rest of the nodes working as proxies. This will enable every component to talk to localhost and target the etcd cluster. The cluster is bootstrapped using [static configuration](https://coreos.com/etcd/docs/latest/clustering.html#static).

```bash
[ansinetes@demo ~]$ ansible-playbook /etc/ansible/books/etcd-bootstrap.yml

PLAY [Bootstrap etcd cluster] **************************************************
TASK [Stop etcd2] **************************************************************
TASK [Purge etcd data] *********************************************************
TASK [Create ssl dir] **********************************************************
TASK [Create server certificate for client endpoint] ***************************
TASK [Create server certificate for peer endpoint] *****************************
TASK [Create client certificate] ***********************************************
TASK [Upload ssl certificates and keys] ****************************************
TASK [Create drop-in dir] ******************************************************
TASK [Upload config file to /etc/systemd/system/etcd2.service.d/30-ansible] ****
TASK [Upload config file to /etc/etcd2/env] ************************************
TASK [daemon-reload] ***********************************************************
```

Then start and enable the etcd service:
```bash
[ansinetes@demo ~]$ ansible-playbook /etc/ansible/books/etcd-up.yml
PLAY [Start etcd] **************************************************************
TASK [start-etcd] **************************************************************
```

## Configure flannel
```bash
[ansinetes@demo ~]$ ansible-playbook /etc/ansible/books/flannel-bootstrap.yml
PLAY [Configure and start flannel] *********************************************
TASK [Stop flanneld] ***********************************************************
TASK [Configure flannel network in etcd (25.0.0.0/16)] *************************
TASK [Create flannel drop-in dir] **********************************************
TASK [Create flannel config dir (for older flannels running in docker)] ********
TASK [Upload flannel config file] **********************************************
TASK [Upload flannel config file (for older flannels running in docker)] *******
TASK [daemon-reload] ***********************************************************
TASK [Start flanneld] **********************************************************
TASK [Restart docker] **********************************************************
```
The overlay network is 25.0.0.0/16 so it can easily be distinguished from other 10.*
networks lying around. That's fine unless you are going to communicate with the [British Ministry
of Defense](https://en.wikipedia.org/wiki/LogMeIn_Hamachi#Addressing). Flannel subnet can be configured.

## Install Kubernetes
This step is the slowest as it downloads more than a 1G from Github.

```bash
[ansinetes@demo ~]$ ansible-playbook /etc/ansible/books/kubernetes-bootstrap.yml
PLAY [Install kubernetes on all nodes] *****************************************
TASK [Download kubernetes binaries locally] ************************************
TASK [Extract binaries] ********************************************************
TASK [Stop services] ***********************************************************
TASK [Create kube user] ********************************************************
TASK [Create kube user dirs] ***************************************************
TASK [Create kubernetes dirs] **************************************************
TASK [Upload kubernetes binaries] **********************************************
TASK [Upload jsonl policy] *****************************************************
TASK [Upload systemd unit files] ***********************************************
TASK [Create apiserver certificates for every node] ****************************
TASK [Upload ca file] **********************************************************
TASK [Upload apiserver certificates] *******************************************
TASK [Create client certificates] **********************************************
TASK [Upload kube-proxy client certificates] ***********************************
TASK [Upload kubelet client certificates] **************************************
TASK [Upload scheduler client certificates] ************************************
TASK [Upload controller-manager client certificates] ***************************
TASK [Create service account keys] *********************************************
TASK [Upload service account keys] *********************************************
TASK [Render kubecfg's] ********************************************************
TASK [Render kubernetes .env files] ********************************************
TASK [Render cluster add-ons] **************************************************
TASK [Systemd daemon reload] ***************************************************
```

Then you need to start the services:
```bash
[ansinetes@demo ~]$ ansible-playbook /etc/ansible/books/kubernetes-up.yml
PLAY [Start kubernetes] ********************************************************
TASK [Upload systemd unit files] ***********************************************
TASK [Reload daemon] ***********************************************************
TASK [Stop all services] *******************************************************
TASK [Start apiservers] ********************************************************
TASK [Start schedulers] ********************************************************
TASK [Start controller-managers] ***********************************************
TASK [Start kubelet and proxy] *************************************************
```

## Access your environment
Now that your cluser is running in Vagrant it's time to access it. In another shell
run `ansinetes -p demo -s` (-s for shell). It will start your $SHELL with a modified
environment, changed $PATH, changed $PS1 and both `kubectl` and `etcdctl` preconfigured:

```bash
$ ./ansinetes -p demo -s
Installing etcdctl locally...
######################################################################## 100.0%
Installing kubectl locally...
######################################################################## 100.0%
Welcome to ansinetes virtual environment "demo"

# etctl is configured!
$ [*demo*] etcdctl member list
546c6cb8c021e3: name=a42bababcba440978f62d41d8b7e26b6 peerURLs=https://172.33.8.101:2380 clientURLs=https://172.33.8.101:2379 isLeader=false
6565485f164c9da3: name=5f6e37eb65d84199b5f3551959667537 peerURLs=https://172.33.8.102:2380 clientURLs=https://172.33.8.102:2379 isLeader=true
a0aca39404520794: name=adf5c02dd12a48419d681359438a2740 peerURLs=https://172.33.8.103:2380 clientURLs=https://172.33.8.103:2379 isLeader=false

# ... and so is kubectl :)
$ [*demo*] kubectl get no
NAME           STATUS    AGE
172.33.8.101   Ready     22m
172.33.8.102   Ready     22m
172.33.8.103   Ready     22m
172.33.8.104   Ready     22m

$ [*demo*] kubectl get pod --all-namespaces
NAMESPACE     NAME                                    READY     STATUS    RESTARTS   AGE
kube-system   kube-dns-v19-r2o5g                      3/3       Running   0          1m
kube-system   kubernetes-dashboard-2982215621-w1nu4   1/1       Running   0          1m
```

Your old kubecfg will be left intact. In fact every time you enter a shell with `-s` the kubecfg will be enriched with configuration about the cluster and the full path to the project dir will be used as the context and cluster name.

# Deployment description
When vagrant is used 4 VMs are created. This can be changed by editing the vagrant/config.rb script. 3 nodes take part in the etcd quorum while the rest are proxies. There are two api-servers. Controller-manager and Scheduler run with `--leader-elect` option. Components that target the apiserver will talk to the first node from the 'apiservers' group (no HA here). The default `hosts` file describes the role mapping:

```ini
[coreos]
kbt-1
kbt-2
kbt-3
kbt-4

[apiservers]
kbt-2
kbt-4

[schedulers]
kbt-1
kbt-3

[controller-managers]
kbt-4
kbt-3

[etcd-quorum-members]
kbt-1
kbt-2
kbt-3

```

Systemd units are installed uniformly everywhere but the services are started and enabled only on the designated nodes. If you change the service to node mapping you need to `Kubernetes-up.yml` your environment again (it will stop all services at first). Kubelet and Proxy run on every node.

Every component authenticates to the apiserver using a private key. Have a look at the jsonl policy file for details.

Controller-manager runs under a service account verified by public/private key.

Only two addons are deployed: Dashboard and DNS. You may want to secure the Dashboard as it has full access to the cluster (on behalf of the kubelete user). For convinience the dashboard is exposed on port 30033. You may change/disable this. The add-ons yamls are touched a bit to work with the service accounts used to protect the apiserver.

# Customizing the deployment
The easiest way to customize the deployment is to edit the `ansible/group_vars/all.yml` file. You can control many options there. If you want to experiment with other Kubernetes config options you can edit the ansible/k8s-config/*.j2 templates. They are borrowed from the [init](https://github.com/Kubernetes/contrib/tree/master/init/systemd) contrib dir. If you find an option worthy of putting in a group var please contribute! The systemd unit files will hardly need to be touched though. The `hosts` can be changed to remap/resize pools allocated to different components.
An example `all.yml` file is given bellow to
```yaml
kubernetes_install:
  version: v1.3.6
  sha256: sha256:2db7ace2f72a2e162329a6dc969a5a158bb8c5d0f8054c5b1b2b1063aa22020d

flannel:
  network: "25.0.0.0/16"

kubernetes_cluster_ip_range: "10.254.0.0/16"

kubernetes_dns:
  ip: "10.254.0.2"
  replicas: 1
  domain: "cluster.local"

kubernetes_etcd_prefix: /registry

kubernetes_dashboard_port: 30033
```

Ansible must be configured to store facts in json files (the `fact_caching = jsonfile` setting). Facts are used later by the ansinetes script. If you expect facts to change just delete this directory and it will be populated next time a runbook is played.

# Starting over
Run any of the *-bootstrap playbooks as often as you like. After boostraping you may need to run the *-up or *-down playbooks. Runbooks try to be idempotent and do as little work as possible. The boostrap scripts will cause downtime as they will stop all services unconditionally.

## Regenerating the CA
Delete the security/ca.pem file, security/certs/* and run kt-ca-init again. The `etcd-bootstrap.yml` and `Kubernetes-boostrap.yml` need to be run again to distribute the new certificates and keys. By judiciously deleting *.pem file from the `cert` dir you can retrigger key re-generation and redistribution.

## Purging cluster state
`etcd-bootstrap.yml` will purge etcd state. If you've messed up only Kubernetes, you can run only the `kubernetes-purge.yml` playbook.

## Rotate SSH keys
Run the playbook `ssh-keys-rotate.yml`. This will prevent ansible from talking to your vms unless you update the `vagrant/user-data` file with the security/new ansible-ssh-key.pub before the next reboot.

# Recommended workflow
This project directory fully captures the cluster config state including the Ansible scripts and Kubernetes customization. You can keep it under source control. You can later change the playbooks and/or the `hosts` file. The `ansinetes` script is guarateed to work with somewhat modified project defaults.

I also prefer naming the ansible hosts in order not to deal with IPs. The kubelets assigns a label "ansinetes:{name}" to the nodes for usability. But really, every way is the right way as long as the hosts file contains the required groups.

# Why use ansinetes?
You would find ansinetes useful if you:
* want to test Kubernetes in an HA, multi-node deployment
* have some spare CoreOS VMs and want to quickly make a secure kube cluster with sensible config
* want to try the Kubernetes bleeding edge and can't wait for the packages to arrive for your distro
* want to painlessly try out obscure Kubernetes options

# FAQ
* Why CoreOS? Because it nicely integrates docker with flannel. With a little extra work you could run ansinetes on every systemd-compatible distro.
* Do I have to use Vagrant? No, every CoreOS box that you can ssh into can be used. Beware though as it will wipe out your flannel/etcd configurations.
* Is federated Kubernetes supported? Not yet.

# Known issues
Ansible hangs sometimes when uploading files. You may need to Ctrl+C the run and re-run it.

# Resources
* [Kubernetes from scratch](http://Kubernetes.io/v1.0/docs/getting-started-guides/scratch.html)
* [Kubernetes the hard way](https://github.com/kelseyhightower/Kubernetes-the-hard-way)
* [Enabling HTTPS in an existing etcd cluster](https://coreos.com/etcd/docs/latest/etcd-live-http-to-https-migration.html)
* [Configuring flannel for container networking](https://coreos.com/flannel/docs/latest/flannel-config.html)
* [docker-Kubernetes-tls-guide](https://github.com/kelseyhightower/docker-Kubernetes-tls-guide)
