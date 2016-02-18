# Building a multi-tier application
## using OpenStack (PackStack)


```
$ chkconfig NetworkManager off
$ chkconfig network on
$ systemctl stop NetworkManager.service
$ systemctl start network.service
```

```
$ vi /etc/environment
```
    LANG=en_US.utf-8
    LC_ALL=en_US.utf-8


```
$ yum install -y https://www.rdoproject.org/repos/rdo-release.rpm
$ sudo yum update -y
$ sudo yum install -y openstack-packstack
$ packstack --allinone --os-neutron-lbaas-install=y
```

```
$ source ~/keystonerc_admin
```

```
$ openstack network list
```

```
$ openstack security group create web
```

```
$ openstack security group create database
```

```
$ openstack security group create ssh
```

```
$ openstack security group list
```

```
$ neutron security-group-rule-create --direction ingress --protocol TCP --port-range-min 80 --port-range-max 80 web
```

```
$ neutron security-group-rule-create --direction ingress --protocol TCP --port-range-min 3306 --port-range-max 3306 --remote-group-id web database
```

```
$ neutron security-group-rule-create --direction ingress --protocol TCP --port-range-min 22 --port-range-max 22 --remote-group-id ssh database
```

```
$ neutron security-group-rule-create --direction ingress --protocol TCP --port-range-min 22 --port-range-max 22 --remote-group-id ssh web
```

```
$ neutron security-group-rule-create --direction ingress --protocol tcp  --port-range-min 22 --port-range-max 22 ssh
```

```
$ openstack network list
```

```
$ openstack image list
```

```
$ openstack flavor list
```

```
$ nova boot --image cirros --nic net-id=84aff6b0-2291-41b5-9871-d3d24906e358 --security_groups web --flavor 1 web_server1
```

```
$ nova boot --image cirros --nic net-id=84aff6b0-2291-41b5-9871-d3d24906e358 --security_groups web --flavor 1 web_server2
```

```
$ nova boot --image cirros --nic net-id=84aff6b0-2291-41b5-9871-d3d24906e358 --security_groups database --flavor 1 database_server
```

```
$ nova boot --image cirros --nic net-id=84aff6b0-2291-41b5-9871-d3d24906e358 --security_groups ssh --flavor 1 jumphost
```

```
$ nova boot --image cirros --nic net-id=84aff6b0-2291-41b5-9871-d3d24906e358 --flavor 1 client
```

```
$ openstack server list
```

```
$ neutron floatingip-create public
```

```
$ neutron port-list
```

```
$ neutron floatingip-associate ce6efd31-97e9-428b-a3ac-b5a14e77a305 5d98aa79-8bc0-4512-a128-078281aae2bc
```

```
$ neutron floatingip-list
```

```
$ ssh cirros@172.24.4.228
```

```
$ ssh 10.0.0.3
$ ssh 10.0.0.4
$ ssh 10.0.0.5
```

```
# On web_server 1 (10.0.0.3)
$ while true; do echo -e 'HTTP/1.0 200 OK\r\n\r\n'`hostname` | sudo nc -l -p 80 ; done &
$ exit

# on web_server 2 (10.0.0.4)
$ while true; do echo -e 'HTTP/1.0 200 OK\r\n\r\n'`hostname` | sudo nc -l -p 80 ; done &
$ exit
```

```
$ wget -O - http://10.0.0.3/
$ wget -O - http://10.0.0.4/
```

```
# On web_server 1 
$ while true; do echo -e 'HTTP/1.0 200 OK\r\n\r\n'`hostname` | sudo nc -l -p 81 ; done &

$ wget -O - http://10.0.0.3:81
```

```
$ neutron subnet-list
```

```
$ neutron lb-pool-create --name http-pool --lb-method ROUND_ROBIN --protocol HTTP --subnet-id 92432fb8-8c29-4abe-98d8-de8bf161a18b
```

```
$ neutron lb-pool-list
```

```
$ neutron lb-pool-show http-pool
```

```
$ neutron lbaas-healthmonitor-create --delay 3 --type HTTP --max-retries 3 --timeout 3 --pool webserver-pool
```

```
$ neutron lb-member-create --address 10.0.0.3 --protocol-port 80 http-pool
```

```
$ neutron lb-member-create --address 10.0.0.4 --protocol-port 80 http-pool
```

```
$ neutron lb-member-list
```

```
$ neutron lb-healthmonitor-create --delay 3 --type HTTP --max-retries 3 --timeout 3
```

```
$ neutron lb-healthmonitor-associate 05cbf7f9-d01b-483b-934e-8955b14a1653 http-pool
```

```
$ neutron lb-vip-create --name webserver-vip --protocol-port 80 --protocol HTTP --subnet-id 92432fb8-8c29-4abe-98d8-de8bf161a18b http-pool
```

```
$ for i in $(seq 1 4) ; do wget -O - http://10.0.0.8/ ; done
```

```
$ neutron floatingip-create public
```

```
$ neutron port-list
```

```
$ neutron floatingip-associate 3c15f8a4-bdc6-4154-8bfa-e8b0674079ca ecbadc53-5266-4146-bbb3-d1b6d383be10
```

```
$ neutron security-group-list
```

```
$ neutron port-update ecbadc53-5266-4146-bbb3-d1b6d383be10 --security_groups list=true a98fcd2f-a828-4a88-92aa-36e3c1223a92
```

```
$ for i in $(seq 1 4) ; do wget -O - http://172.24.4.229/ ; done
```

```
$ openstack server delete web_server1
```

```
$ for i in $(seq 1 4) ; do wget -O - http://172.24.4.229/ ; done
```
