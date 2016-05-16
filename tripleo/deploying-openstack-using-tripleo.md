#Deploying OpenStack using TripleO *(WIP)*

_Gerard Braad <me@gbraad.nl>_



## Description
In this article we will deploy an OpenStack environment using TripleO using the `tripleo-quickstart` scripts. It will create a virtualized environment which consists of 1 Undercloud node and in total 9 nodes in the Overcloud; 3 controller nodes (High Availability), 3 compute nodes and 3 Ceph storage nodes.


## Prerequisites
It is preferred to use a dedicated server for the following instructions. Consider using 32G of memory and have enough diskspace available in `/home`.


## What is TripleO, _undercloud_ and _overcloud_?
[TripleO](https://) is a deployment method in which a dedicated OpenStack management environment is used to deploy another OpenStack environment which is used for the workload. The management environment is contained in a single machine, called the _undercloud_. This OpenStack environment takes care of the monitoring and management of what is known as the _overcloud_. The _overcloud_ is the actual OpenStack environment that will run the workload.


## Prepare for deployment
The deployment can be done from a workstation targeting a server, or from the server itself. Whatever your preferred method is, you will need the [tripleo-quickstart](https://github.com/openstack/tripleo-quickstart) scripts.

```
$ chmod u+x quickstart.sh
$ ./quickstart.sh --install-deps
```



## Deployment configuration file

```

```

Note: additional node is defined to sue to scale out later.


## Perform deployment


	* `--tags all`


```
$ export $VIRTHOST=[fqdn]
$ ./quickstart.sh --tags all
```


## Tags and scripts



## Login to undercloud node

```
$ ssh -F ~/.quickstart/ssh.config.ansible
```


## Login to overcloud nodes
If you need to inspect a node in the overcloud (workload), you can login using the following command:

```
# From the undercloud node
$ ssh -i ~/.ssh/id_rsa heat-admin@[nodeip]
```


## Scale out


## More information
See [scratchpad](https://github.com/gbraad/openstack-tripleo-scratchpad) for notes on 
TripleO.
