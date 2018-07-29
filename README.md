# rancher-vagrant
Vagrantfile for deploying Rancher 2.0 in VirtualBox.

## Pre-requisites

* VirtualBox
* [Vagrant plug-in Hostmanager](https://github.com/devopsgroup-io/vagrant-hostmanager)

## Installation

```bash
$ git clone https://github.com/genebean/rancher-vagrant.git
$ cd rancher-vagrant
```

## Configuration

The Rancher platform you get with this Vagrantfile is:

* Single Rancher master server (2x CPU/2GB memory)
* Three Rancher node servers (1x CPU/1GB memory)
* Vagrant box genebean/centos-7-docker-ce

The Vagrantfile configures each box to use the stock docker that is supported by
Rancher by default. If you would prefer to use the latest version of Docker CE
change

```ruby
USE_STOCK_DOCKER = true
```

to `false` near the top of the Vagrantfile.

Before you run `vagrant up` you should review the Vagrantfile settings to map
your requirements.

### Vagrant box

```ruby
BOX_IMAGE = "genebean/centos-7-docker-ce"
```

### Number of Rancher node servers

```ruby
NODE_COUNT = 3  # Minimum one node. Maximum 244 nodes
```

### Rancher password

```ruby
RANCHER_PASSWORD = 'password'
```

### Rancher cluster name

```ruby
CLUSTER_NAME = 'playground'
```

### Master server hardware

```ruby
master.vm.provider :virtualbox do |vb|
  vb.memory = 2048  # Recommended 4GB
  vb.cpus = 2
end
```

### Node server hardware

```ruby
config.vm.provider "virtualbox" do |vb|
  vb.memory = 1024
  vb.cpus = 1
  vb.linked_clone = true
end
```

### Hostname and IP address

**master**

```ruby
master.vm.hostname = "master.rancher.local"
master.vm.network :private_network, ip: "192.168.34.10"
```

**node**

For the node servers provisioning a loop is used. You should only change the
three first octets of the IP address `192.168.34.` and not the formula
`#{i + 10}`.

```ruby
node.vm.hostname = "node#{i}.rancher.local"
node.vm.network :private_network, ip: "192.168.34.#{i + 10}"
```

## Usage

```bash
$ vagrant up
```

Depending on your hardware performance and Internet speed, a single Rancher node
platform (master is always required) can take around 5-10 minutes to come up.

You can check the Rancher cluster status on ***https://master.rancher.local***
