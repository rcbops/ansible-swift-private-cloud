Ansible Swift Install
=====================

This repo contains a set of Ansible playbooks to fire up the [Swift Private Cloud](https://github.com/rcbops-cookbooks/swift-private-cloud) Chef cookbooks in the reference implementation.

Quick Install
=============

Install Ansible and PyRax
-------------------------

    $ [sudo] pip install ansible
    $ [sudo] pip install markupsafe
    $ [sudo] pip install pyrax

Provision Servers in Rackspace Public Cloud
-------------------------------------------

This repo includes a playbook to provision a set of Rackspace Cloud Servers. It will create:

* Isolated Network ( "swift_management", "192.168.122.0/24" )
* Chef Server x 1
* Management Server x 1
* Storage Servers x 3
* Proxy Servers x 2

Each server will belong to PublicNet, ServiceNet and the swift_management network.

First, ensure you have a credentials file configured with your `username` and `api_key`:

    # file: ~/.rackspace_cloud_credentials
    [rackspace_cloud]
    username = yourusername
    api_key = yourapikey

We're assuming you have an `~/.ssh/id_rsa.pub` and it will be uploaded to the root account.

Then run the `rackspace.yml` playbook:

    $ ansible-playbook rackspace.yml

When this completes, it will create a hosts inventory file called `rackapace-swift`:

    [management-server]
    192.0.2.2

    [storage-servers]
    192.0.2.3
    192.0.2.4
    192.0.2.5

    [proxy-servers]
    192.0.2.6
    192.0.2.7

    [chef-server]
    192.0.2.1

    [chef-clients:children]
    management-server
    proxy-servers
    storage-servers

By default this will spin up CentOS images. To use Ubuntu instead, override the image variable:

    $ ansible-playbook rackspace.yml -e image=25de7af5-1668-46fb-bd08-9974b63a4806

The following options can be overriden using `-e`:

* image: Set the glance image to be used. Default: e0ed4adb-3a00-433e-a0ac-a51f1bc1ea3d (CentOS 6.4)
* flavor: Set the flavor to be used. Default: 4  (2GB Standard Instance)
* region: Set the region to operate in. Default: IAD
* prefix: Set the prefix when naming servers/inventory files. Default: ""

Once complete, ensure you can ssh to each instance properly:

    $ ansible all -i rackspace-swift -m command -a "whoami"

Additional provision playbooks will be created on an as needed basis.

You can also create your own hosts inventory file and point it to already existing servers as long as the following are true:

* the group names match the above names
* you've already configured ssh public key access for the root account

Provision Servers in Vagrant
----------------------------

This repo includes a playbook to provision a set of servers locally using Vagrant/VirtualBox. It will create:

* Chef Server x 1
* Management Server x 1
* Storage Servers x 3
* Proxy Servers x 2

Each server will have 3 interfaces: eth0 (NAT), eth2 (Host Only 10.10.122.x), and eth3 (Host Only 192.168.122.x)

First, ensure you have a working VirtualBox/Vagrant install.
We're assuming you have an `~/.ssh/id_rsa.pub` and it will be uploaded to the root account.

Then run the `vagrant.yml` playbook:

    $ ansible-playbook vagrant.yml

This will create an inventory file with the same groups specified above and wil be named `vagrant-swift`.

By default, servers will be created usingthe Puppet Labs CentOS 6.4 base image. To use Ubuntu instead, override the `box` and `box_url` variables:

    $ ansible-playbook vagrant.yml -e box="Puppetlabs Ubuntu 12.04 x86_64" box_url="http://puppet-vagrant-boxes.puppetlabs.com/ubuntu-server-12042-x64-vbox4210-nocm.box"

Configure Servers
-----------------

To configure the servers in the inventory file, run the `site.yml`:

    $ ansible-playbook -i rackspace-swift site.yml

This will:

* Install Chef Server on any servers in the `chef-server` group.
* Install Chef Client on any servers in the `chef-clients` group, including configuring and registering nodes to talke to the chef server.
* Run the `spc-starter-controller` chef role on server in the `management-server` group.
* Run the `spc-starter-storage` chef role on the servers in the `storage-servers` group.
* Run the `spc-starter-proxy` chef role on the servers in the `proxy-severs` group.
* Re-run the `spc-starter-controller` chef role on server in the `management-server` group to gather all the other nodes.
* Re-run the `spc-starter-proxy` chef role on the proxy servers again to propgate all memcached servers addresses.

While running `site.yml` runs everything, you can also run them role by role:

    $ ansible-playbook -i rackspace-swift chef-server.yml
    $ ansible-playbook -i rackspace-swift chef-clients.yml
    $ ansible-playbook -i rackspace-swift management-server.yml
    ...

...and even server by server:

    $ ansible-playbook -i rackspace-swift storage-servers.yml --limit 192.0.2.3

Running Tests
-------------

This repository includes a `test.yml` playbook/role that runs all of the tests that are being run in Jenkins against the current inventory/cluster.

    $ ansible-playbook -i rackspace-swift test.yml

Caveat Executor
===============

This is just a first pass. Batteries not included. We are making some assumptions:

* The isolated network being created matches the swift environment file: `192.168.122.0/24`
* If you use your own servers instead, you have one network interface matching `192.168.122.0/24`
* You can root ssh to these machines using ssh public keys. (This can probably be changed via `ansible.cfg` or command line args)
* You're running CentOS 6.4. Ubuntu 12.04 needs to be cussed out.

Many things are hard coded but will become more flexible in the future as they slowly move from cookbooks into playbooks. Not responsible if your dog catches fire or your significant other files for divorce.

Repository Structure
====================

This repository is configured close to the [Ansible Best Practices](http://www.ansibleworks.com/docs/playbooks_best_practices.html#directory-layout) with the exception of the `provision` folder. This may become a role in the future, but I didn't want to intermingle it and cause confusion right now.

The roles in this repo mimic the top level recipes in the Chef cookbooks. If we move forward with Ansible, we can just move things from the recipes into the exists top level roles without having to change the site/role playbooks.
