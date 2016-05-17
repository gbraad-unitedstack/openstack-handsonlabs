#Deploying OpenStack using TripleO *(WIP)*

_Gerard Braad <me@gbraad.nl>_


## Description
In this article we will deploy an OpenStack environment using TripleO using
the `tripleo-quickstart` scripts. It will create a virtualized environment
which consists of 1 Undercloud node and in total 9 nodes in the Overcloud;
3 controller nodes (High Availability), 3 compute nodes and 3 Ceph storage
nodes.


## What is TripleO, _undercloud_ and _overcloud_?
[TripleO](https://) is a deployment method in which a dedicated OpenStack
management environment is used to deploy another OpenStack environment which is
used for the workload. The management environment is contained in a single
machine, called the _undercloud_. This OpenStack environment takes care of the
monitoring and management of what is known as the _overcloud_. The _overcloud_
is the actual OpenStack environment that will run the workload.


## Prerequisites
It is preferred to use a dedicated server for the following instructions.
Consider using 32G of memory and have enough diskspace available in `/home`.
This target node has to be using CentOS 7 (or RHEL7).


## Prepare for deployment
The deployment can be done from a workstation targeting a server, or from the
server itself. Whatever your preferred method is, you will need the 
[tripleo-quickstart](https://github.com/openstack/tripleo-quickstart) scripts.

```
$ curl -O https://raw.githubusercontent.com/openstack/tripleo-quickstart/master/quickstart.sh
$ chmod u+x quickstart.sh
$ ./quickstart.sh --install-deps
```

This will install the dependencies used by the `quickstart.sh` script. Now you
need to prepare the target node. Make sure you can login to this server without
a password.

```
$ export VIRTHOST=[target]
$ ssh-copy-id root@$VIRTHOST
$ ssh root@$VIRTHOST uname -a
```

This article witll not describe the setup of passwordless or key-based
authentication with SSH. If issues occur, please verify you have a generated
keyset. If not, generate one with `ssh-keygen`. For further information please
consult the `man` page or verify online.


## Cache deployment images locally
Although this step is not necessary, you can pre-download the images and cache
them locally. This can be helpful if you want to perform the deployment using
different images and/or suffer from bad connectivity.

The location if the images is currently at `http://artifacts.ci.centos.org/rdo/images/{{release}}/delorean/stable/undercloud.qcow2`.

To download the _mitaka_ image, you can do this with

```
$ mkdir /var/lib/oooq-images
$ curl http://artifacts.ci.centos.org/rdo/images/mitaka/delorean/stable/undercloud.qcow2 -o /var/lib/oooq-images/undercloud-mitaka.qcow2
```


## Deployment configuration file
We will create a virtual environment for out TripleO deployment. This will
provide you with the knowledge you need to perform a bare-metal installation.

You can define the node deployment inside a configuration file, eg. called 
`deploy-config.yml`. This will contain the following:

```
overcloud_nodes:
  - name: control_0
    flavor: control
  - name: control_1
    flavor: control
  - name: control_2
    flavor: control

  - name: compute_0
    flavor: compute
  - name: compute_1
    flavor: compute
  - name: compute_2
    flavor: compute

  - name: storage_0
    flavor: ceph
  - name: storage_1
    flavor: ceph
  - name: storage_2
    flavor: ceph

extra_args: "--control-scale 3 --ceph-storage-scale 3 -e /usr/share/openstack-tripleo-heat-templates/environments/puppet-pacemaker.yaml --ntp-server pool.ntp.org"

```

Nodes can be assigned a role by setting the flavor:

  * `control` sets up a controller node, which also handles the network.
  * `compute` sets up a Nova compute node.
  * `ceph` sets up a node for Ceph storage.


The extra arguments allow you to modify the deployment that is performed.

  * `--control-scale 3` instructs the deployment to assign 3 nodes with the
      controller role.
  * `--ceph-storage-scale 3` instructs the deployment to assign 3 nodes to
      be used for the Ceph storage backend.
  * `-e puppet-pacemaker.yaml` will setup pacemaker HA for the controller
      nodes
  * `--ntp-server pool.ntp.org` will sync time on the nodes using NTP.

Note: if you want to use all compute nodes at once, include `--compute-scale 3`. But in this article I am using these additional nodes for scale out.


## Perform deployment
Now you can perform the _undercloud_ deployment using:

```
$ ./quickstart.sh --config deploy-config.yml --undercloud-image-url file:///var/lib/oooq-images/undercloud-mitaka.qcow2 $VIRTHOST
```

But before you do, please continue reading about the `Deployment scripts`.

The previous command will target the node as specified with the `$VIRTHOST` 
environment variable, and according to the `deploy-config.yml` which we defined
earlier.

It will login to this node and create a `stack` user which will be running the
virtual machines. Later we will inspect this. After creating the virtual
machines it will prepare the `undercloud` machine. After this, you still need to
start the actual deployment. 


## Deployment scripts
To login to the undercloud:

```
$ ssh -F $OPT_WORKDIR/ssh.config.ansible undercloud
```

The undercloud is not fully prepared, you would have to do so with the following
scripts.

Undercloud (management)

  * `undercloud-install.sh` will run the undercloud install and execute
    diskimage elements.
  * `undercloud-post-install.sh` will perform all pre-deploy steps, such as
    uploading the images to _glance_

Overcloud (workload)

  * `overcloud-deploy.sh` will deploy the overcloud, creating a _heat_ stack
    and will use the nodes as defined in `instack-env.json` and the extra
    arguments given in the deployment configuration.
  * `overcloud-deploy-post.sh` will do any post-deploy configuration
    such as writing a local `/etc/hosts` file.
  * `overcloud-validate.sh` will run post-deploy validation, like a
    _pingtest_ and possible _tempest_


You can run these scripts one by one... or install the whole _undercloud_ and
_overcloud_ using the command:

```
$ ./quickstart.sh --tags all --config deploy-config.yml --undercloud-image-url file:///var/lib/oooq-images/undercloud-mitaka.qcow2 $VIRTHOST
```

Using `--tags all` will instruct ansible to perform all the steps and scripts
as previously described. I suggest you to run the steps first each one by one
and look into the scripts itself to understand how they interact with
`python-tripleoclient` (eg. `openstack undercloud` and `openstack overcloud`).


```
[stack@undercloud ~]$ ./undercloud-install.sh
[stack@undercloud ~]$ ./undercloud-post-install.sh
[stack@undercloud ~]$ ./overcloud-deploy.sh
[stack@undercloud ~]$ ./overcloud-deploy-post.sh
[stack@undercloud ~]$ ./overcloud-validate.sh
```


## Undercloud node
On the _undercloud_ node 



## Login to overcloud nodes
If you need to inspect a node in the overcloud (workload), you can login to these nodes from the undercloud using the following command:


```
[stack@undercloud ~]$ ssh -i ~/.ssh/id_rsa heat-admin@[nodeip]
```


## Scale out


## Diskimage building


## More information
See [scratchpad](https://github.com/gbraad/openstack-tripleo-scratchpad) for notes on 
TripleO.
