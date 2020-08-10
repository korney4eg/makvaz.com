---
layout: post
title: Kubernetes The Hard Way on your laptop. Setup Virtual Machines.
archives: "2020"
tags: [kubernetes, howto, vagrant]
---
<img src="/2020/07/29/kubernetes-the-hard-way-on-laptop-preparation/laptop.png" alt="" width="120" style="float: left; margin-right: 15px">

_In previous [post](/2020/07/29/kubernetes-the-hard-way-on-laptop-preparation/) we overviewed how the Kubernetes cluster would look like and have installed some programs. Today we gonna set up Virtual Machines using VirtualBox and Vagrant._
<!--more-->
## Structure of this How-to:

1. [Intro. Cluster overview. Prepare local environment](/2020/07/29/kubernetes-the-hard-way-on-laptop-preparation/)
2. [Setup Vagrant, VirtualBox. Create Virtual Machines](#)
4. Provisioning all needed Certificates and keys, generating Kubernetes configuration files
5. Bootstrapping the etcd Cluster. Bootstrapping the Kubernetes Control Plane
7. Bootstrapping the Kubernetes Worker Nodes
8. Setup kubectl, provision needed Add-ons, and plugins. Testing


## VirtualBox

All Kubernetes control nodes, worker nodes, and the load balancer will be running in VirtualBox VMs.

##### Install on MacOS

```bash
brew install virtualbox
```

##### Install on Linux

[Here](https://www.virtualbox.org/wiki/Downloads) you can download the latest version for your Linux distro.

## Vagrant

Vagrant allows to bootstrap VMs with needed settings using automation. So if I need to set up 6 VMs I only need proper Vagrant file.


##### Install on MacOS

```bash
brew install vagrant
```

##### Install on Linux

[Link to download](https://www.vagrantup.com/downloads)

Here you can download *deb* package for Debian-based distros, *rpm* package for CentOS or download binary, and put it to the `/usr/local/bin` directory.

{{< notice tip >}}
There is a great plugin for Vagrant - **hostsupdater**. It allows us to resolve the IP address by name of Virtual Machine on the host instance.

The plugin could be installed with the command:

```bash
vagrant plugin update vagrant-hostsupdater
```
{{< /notice >}}

## Creating Virtual Machines(VMs)

First of all let's create a new directory, where we gonna store all needed files:
```bash
mkdir ~/local-k8s && cd ~/local-k8s
```

Using Vagrant we can just describe in `Vagrantfile` what we want to setup. Let's create `Vagrantfile` with your favorite editor:
{{< highlight ruby "linenos=table" >}}
Vagrant.configure("2") do |config|
  hostnames = {
    'controller0': {:ip:'192.168.50.100',:ram:512 },
    'controller1': {:ip:'192.168.50.101',:ram:512 },
    'controller2': {:ip:'192.168.50.102',:ram:512 },
    'lb'         : {:ip:'192.168.50.200',:ram:512 },
    'worker0'    : {:ip:'192.168.50.10', :ram:1024},
    'worker1'    : {:ip:'192.168.50.11', :ram:1024},
  }

  hosts_file = "127.0.0.1	localhost\n"
  hostnames.each do |hostname,settings|
    hosts_file += "#{settings[:ip]} #{hostname}\n"
  end
  config.vm.provision "shell", inline: "echo \"#{hosts_file}\" > /etc/hosts"

  hostnames.each do |hostname,settings|
    config.vm.define hostname do |instance|
      instance.vm.box = "bento/ubuntu-20.04"
      instance.vm.network "private_network", ip: hostnames[hostname][:ip]
      instance.vm.hostname = hostname
      instance.vm.provider :virtualbox do |v|
        v.customize ["modifyvm", :id, "--memory", hostnames[hostname][:ram]]
        v.customize ["modifyvm", :id, "--name", hostname]
      end
    end
  end
end
{{< / highlight >}}

### Explaining Vagrantfile

#### Starting configuration
{{< highlight ruby  >}}
Vagrant.configure("2") do |config|
  ...
end
{{< / highlight >}}
This line says that we gonna configure vagrant with configuration version **2**.

#### Parameters map of each VM
{{< highlight ruby  >}}
  instances = {
    'controller0': {:ip:'192.168.50.100',:ram:512 },
    'controller1': {:ip:'192.168.50.101',:ram:512 },
    'controller2': {:ip:'192.168.50.102',:ram:512 },
    'lb'         : {:ip:'192.168.50.200',:ram:512 },
    'worker0'    : {:ip:'192.168.50.10', :ram:1024},
    'worker1'    : {:ip:'192.168.50.11', :ram:1024},
  }
{{< / highlight >}}
Basically we create a table of settings that would be used later and assign to variable `instances`. For example from here I know that Virtual Mashine `lb` will have `512` MB RAM and ip address `192.168.50.200`.

#### Populate `/etc/hosts` file with all hostnames
{{< highlight ruby  >}}
  hosts_file = "127.0.0.1	localhost\n"
  instances.each do |hostname,settings|
    hosts_file += "#{settings[:ip]} #{hostname}\n"
  end
  config.vm.provision "shell", inline: "echo \"#{hosts_file}\" > /etc/hosts"
{{< / highlight >}}
Here we generate `/etc/hosts` file for all VMs, so they know about each other. That would look like:
```
127.0.0.1       localhost
192.168.50.10 worker0
192.168.50.11 worker1
192.168.50.100 controller0
192.168.50.101 controller1
192.168.50.102 controller2
192.168.50.200 lb
```

#### Set configuration of all VMs
{{< highlight ruby  >}}
  instances.each do |hostname,settings|
    config.vm.define hostname do |instance|
      instance.vm.box = "bento/ubuntu-20.04"
      instance.vm.network "private_network", ip: instances[hostname][:ip]
      instance.vm.hostname = hostname
      instance.vm.provider :virtualbox do |v|
        v.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
        v.customize ["modifyvm", :id, "--memory", instances[hostname][:ram]]
        v.customize ["modifyvm", :id, "--name", hostname]
      end
    end
  end
{{< / highlight >}}

Let's go deeper here and check line by line.

{{< highlight ruby  >}}
  instances.each do |hostname,settings|
    ...
  end
{{< / highlight >}}
Get from table of settings(variable `instances`) variables `hostname` and `settings` - map of VM settings(ip address, RAM)

{{< highlight ruby  >}}
    config.vm.define hostname do |instance|
      ...
    end
{{< / highlight >}}
Takes a single required parameter(variable `hostname`) which would be the name of the Virtual Machine.

{{< highlight ruby  >}}
      instance.vm.box = "bento/ubuntu-20.04"
{{< / highlight >}}
All VMs will have Ubuntu  20.04 installed, as it's currently the latest version of stable release(August 2020)

{{< highlight ruby  >}}
      instance.vm.network "private_network", ip: instances[hostname][:ip]
{{< / highlight >}}
By default VMs created by Vagrant have predefined first network adapter with **NAT**. This adapter allows to access the Internet, but it doesn't allow to access VM through it. We create a second adapter with type **Internal Network** and assign the needed IP address.

{{< highlight ruby  >}}
      instance.vm.hostname = hostname
{{< / highlight >}}
Set hostname of each VM.

{{< highlight ruby  >}}
      instance.vm.provider :virtualbox do |v|
        ...
      end
{{< / highlight >}}
Set VirtualBox specific configuration

{{< highlight ruby  >}}
        v.customize ["modifyvm", :id, "--memory", instances[hostname][:ram]]
{{< / highlight >}}
Set RAM for each VM.

{{< highlight ruby  >}}
        v.customize ["modifyvm", :id, "--name", hostname]
{{< / highlight >}}
Set name of VM to hostname.

### Running vagrant

To create VMs simply run the command:
```bash
vagrant up
```

{{< notice info >}}
You will probably see a password prompt. Don't worry, it's all right, `hostsupdater` vagrant plugin wants to add newly create virtual machines to local `/etc/hosts` file, for that it needs root permissions.
{{< /notice >}}

You will have similar output:
```
Bringing machine 'worker0' up with 'virtualbox' provider...
Bringing machine 'worker1' up with 'virtualbox' provider...
Bringing machine 'controller0' up with 'virtualbox' provider...
Bringing machine 'controller1' up with 'virtualbox' provider...
Bringing machine 'controller2' up with 'virtualbox' provider...
Bringing machine 'lb' up with 'virtualbox' provider...
==> worker0: Importing base box 'bento/ubuntu-20.04'...
==> worker0: Matching MAC address for NAT networking...
==> worker0: Checking if box 'bento/ubuntu-20.04' version '202007.17.0' is up to date...
==> worker0: Setting the name of the VM: local-k8s_worker0_1596841981923_77771
==> worker0: Clearing any previously set network interfaces...
==> worker0: Preparing network interfaces based on configuration...
    worker0: Adapter 1: nat
    worker0: Adapter 2: hostonly
==> worker0: Forwarding ports...
    worker0: 22 (guest) => 2222 (host) (adapter 1)
==> worker0: Running 'pre-boot' VM customizations...
==> worker0: Booting VM...
==> worker0: Waiting for machine to boot. This may take a few minutes...
    worker0: SSH address: 127.0.0.1:2222
    worker0: SSH username: vagrant
    worker0: SSH auth method: private key
    worker0: Warning: Connection reset. Retrying...
    worker0:
    worker0: Vagrant insecure key detected. Vagrant will automatically replace
    worker0: this with a newly generated keypair for better security.
    worker0:
    worker0: Inserting generated public key within guest...
    worker0: Removing insecure key from the guest if it's present...
    worker0: Key inserted! Disconnecting and reconnecting using new SSH key...
==> worker0: Machine booted and ready!
==> worker0: Checking for guest additions in VM...
    worker0: The guest additions on this VM do not match the installed version of
    worker0: VirtualBox! In most cases this is fine, but in rare cases it can
    worker0: prevent things such as shared folders from working properly. If you see
    worker0: shared folder errors, please make sure the guest additions within the
    worker0: virtual machine match the version of VirtualBox you have installed on
    worker0: your host and reload your VM.
    worker0:
    worker0: Guest Additions Version: 6.1.12
    worker0: VirtualBox Version: 6.0
==> worker0: [vagrant-hostsupdater] Checking for host entries
==> worker0: [vagrant-hostsupdater] Writing the following entries to (/etc/hosts)
==> worker0: [vagrant-hostsupdater]   192.168.50.10  worker0  # VAGRANT: c561d73cbe3d4c6c1959c9a662ec99c1 (worker0) / 31bfb971-8060-4ea1-a17a-4751db2764b7
==> worker0: [vagrant-hostsupdater] This operation requires administrative access. You may skip it by manually adding equivalent entries to the hosts file.
==> worker0: Setting hostname...
==> worker0: Configuring and enabling network interfaces...
==> worker0: Mounting shared folders...
    worker0: /vagrant => /Users/akarneyeu/local-k8s
==> worker0: Running provisioner: shell...
    worker0: Running: inline script
...
==> lb: Mounting shared folders...
    lb: /vagrant => /Users/akarneyeu/local-k8s
==> lb: Running provisioner: shell...
    lb: Running: inline script

```

Now we can check if one of VMs is reacheble and we can ping it:
```bash
$ ping controller1
PING controller1 (192.168.50.101): 56 data bytes
64 bytes from 192.168.50.101: icmp_seq=0 ttl=64 time=0.595 ms
64 bytes from 192.168.50.101: icmp_seq=1 ttl=64 time=0.497 ms
64 bytes from 192.168.50.101: icmp_seq=2 ttl=64 time=0.385 ms
^C
--- controller1 ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 0.385/0.492/0.595/0.086 ms
```

Looks good so far. As you can see we don't have to remember IP addresses.

Now let's try to connect using ssh:
```bash
vagrant ssh controller1

...
vagrant@controller1:~$
```
We successfully reached our `controller1` VM.

### Cleaning up

For now, we can destroy our VMs and recreate them later. Just run the command:
```bash
vagrant destroy -f
```
## Conclusion

Today we created our cluster and it doing nothing for now. Later we gonna set up Kubernetes on these VMs and test it. In the next part, we will generate all certificates and configuration files needed for Kubernetes.

Thanks for reading, have a great week.
