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
After running these commands, you will have a fully deployed environment.
You can verify this from the _undercloud_ node.

```
[stack@undercloud ~]$ . stackrc
[stack@undercloud ~]$ ironic node-list
```

    +--------------------------------------+-----------+--------------------------------------+-------------+--------------------+-------------+
    | UUID                                 | Name      | Instance UUID                        | Power State | Provisioning State | Maintenance |
    +--------------------------------------+-----------+--------------------------------------+-------------+--------------------+-------------+
    | ccf541fc-180e-4777-bfb6-71e92be50de3 | control-0 | c709bd16-a62a-4898-b9c0-308bcfc8a6b4 | power on    | active             | False       |
    | 90e16fbb-2386-48c3-8d55-1210672aa8b6 | control-1 | afbc52cc-4c5c-48e3-aed5-d68dda436601 | power on    | active             | False       |
    | e9f01263-8d89-45e4-a5da-4b3199e4bac6 | control-2 | 78b5781d-42e0-40d1-940f-869fb3de6c66 | power on    | active             | False       |
    | 2a1225c3-2a3e-415f-8bea-d0f13699a3e2 | compute-0 | 909576b0-1e24-42bc-9c17-a9e658ee13f3 | power on    | active             | False       |
    | 9ec746e2-801e-4fda-8e81-bd4c8169a0c4 | compute-1 | 6fbe0a94-4d5b-4fbd-a3ce-38972e580deb | power on    | active             | False       |
    | 95285ffe-cc9d-4b20-b7a2-d958ed39420a | compute-2 | e55bcb60-c974-456b-b547-8a61f05a143f | power on    | active             | False       |
    | 08244664-df82-45b2-9792-8ce156da3d11 | storage-0 | 3e924d5d-5a37-4be3-ba43-28b1663973c8 | power on    | active             | False       |
    | bbf15617-37df-40b6-8bd9-efee48501738 | storage-1 | eab06a66-cd90-4468-9da3-45ec69a06aad | power on    | active             | False       |
    | 68d70cb0-76a7-4050-bf4f-64be324eaa37 | storage-2 | 9707f1af-c926-4b9e-8fd9-080b52de36a3 | power on    | active             | False       |
    +--------------------------------------+-----------+--------------------------------------+-------------+--------------------+-------------+


This will show a list of nodes that are available in the environment. This information 


## Login to the overcloud
From the undercloud node you can source the stack resource file and use the
_openstack clients_ as usual.

```
[stack@undercloud ~]$ . overcloudrc
[stack@undercloud ~]$ cat overcloudrc
[stack@undercloud ~]$ nova list
```

In the previous output you also see the `OS_AUTH_URL` and the credentials
needed to login from the Horizon dashboard.

Either using ssh portforwaring, or the dynamic proxy option, you can open the
dashboard.

```
$ ssh -F ~/.quickstart/ssh.config.ansible undercloud -D 8080
```

Either using Firefox (with the FoxyProxy extension) or Chrome/Vivaldi (with the
SwitchySharp extension) you can set a SOCKS proxy at `127.0.0.1` and port
`8080`.


## Login to overcloud nodes
If you need to inspect a node in the overcloud (workload), you can login to these nodes from the undercloud using the following command:

```
[stack@undercloud ~]$ ssh -i ~/.ssh/id_rsa heat-admin@[hostname/nodeip]
```

Note: you can find the hostnames and IP addresses on the _undercloud_ in the
`/etc/hosts` file.


## Scale out

```
[stack@undercloud ~]$ for i in $(ironic node-list | grep Available | grep -v UUID | awk ' { print $2 } '); do
> ironic node-set-maintenance $i false;
> done
```


## Diskimage building
The _undercloud_ images can be created using [ansible-role-tripleo-image-build](https://github.com/redhat-openstack/ansible-role-tripleo-image-build).
Using the following commands it will generate the images:

```
$ git clone https://github.com/redhat-openstack/ansible-role-tripleo-image-build.git
$ cd ansible-role-tripleo-image-build/tests/pip
$ sudo ./build.sh -i
$ ./build.sh $VIRTHOST
```

After the command finishes succesfully, the images can be found in
`/var/lib/oooq-images`.


## More information
See [scratchpad](https://github.com/gbraad/openstack-tripleo-scratchpad) for
notes on TripleO.
