---
layout: post
title: "OpenStackの各サービスのDB"
date: 2020-03-26 00:00:00 +0900
lang: ja
---

## Cinder

```
ubuntu@maas:~$ juju ssh cinder/0 sudo cat /etc/cinder/cinder.conf | grep -v '^\s*#' | grep -v '^\s*$' | grep connection
connection = mysql://cinder:WzXGwc64kTyzrWKc8mr9ZZHTnbkyZkjR@192.168.122.9/cinder
```

## Glance

```
ubuntu@juju-3513fa-2-lxd-1:/etc/glance$ grep "connection =" *
glance-api.conf:connection = mysql://glance:tKFnjGWMzpmwndbWndRHqd4FZY8LCMmd@192.168.122.9/glance
glance-registry.conf:connection = mysql://glance:tKFnjGWMzpmwndbWndRHqd4FZY8LCMmd@192.168.122.9/glance
```

## Keystone

```
ubuntu@maas:~$ juju ssh keystone/0 sudo cat /etc/keystone/keystone.conf | grep -v '^\s*#' | grep -v '^\s*$'
connection = mysql://keystone:3jgVBJRTzMnBy4qzdr4d6TbZwP8TT8PN@192.168.122.9/keystone
```

## Rabbitmq

RabbitmqはDBは使わないようだ

```
ubuntu@maas:~$ juju ssh rabbitmq-server/0 sudo ls /etc/rabbitmq/
enabled_plugins rabbitmq.config rabbitmq-env.conf
ubuntu@maas:~$ juju ssh rabbitmq-server/0 sudo grep connection /etc/rabbitmq/*
```

## Neutron

```
ubuntu@maas:~$ juju ssh neutron-gateway/0 sudo grep connection /etc/neutron/neutron.conf
ubuntu@maas:~$ juju ssh neutron-api/0 sudo grep connection /etc/neutron/neutron.conf
```

## Nova

```
ubuntu@maas:~$ juju ssh nova-cloud-controller/0 sudo grep connection /etc/nova/nova.conf
connection_type=libvirt
connection = mysql://nova:zWJdWdmJsxXVcm97L2mJsCYKqxzC7dJR@192.168.122.9/nova
connection = mysql://nova:zWJdWdmJsxXVcm97L2mJsCYKqxzC7dJR@192.168.122.9/nova_api
root@node04:/var/lib/lxd/containers/juju-3513fa-2-lxd-2/rootfs# grep connection etc/nova/nova.conf
connection_type=libvirt
connection = mysql://nova:zWJdWdmJsxXVcm97L2mJsCYKqxzC7dJR@192.168.122.9/nova
connection = mysql://nova:zWJdWdmJsxXVcm97L2mJsCYKqxzC7dJR@192.168.122.9/nova_api
```

## MySQL

```
ubuntu@maas:~$ juju run --unit mysql/0 leader-get
bootstrap-uuid: b4353480-6d9c-11ea-b81d-8789f420c23f
leader-ip: 192.168.122.9
mysql-cinder.passwd: WzXGwc64kTyzxxx
mysql-glance.passwd: tKFnjGWMzpxxx
mysql-keystone.passwd: 3jgVBJRTzxxx
mysql-neutron.passwd: yqPZky8K3xxx
mysql-nova.passwd: zWJdWdmJsxxx
mysql.passwd: pPmGnqLRgghjxxx
root-password: pPmGnqLRgghj5jxx
sst-password: Pg24t9qY2HZVxxx
ubuntu@maas:~$ juju ssh mysql/0
ubuntu@juju-3513fa-0-lxd-1:~$ mysql -u root -p
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| cinder             |
| glance             |
| keystone           |
| mysql              |
| neutron            |
| nova               |
| nova_api           |
| performance_schema |
| test               |
+--------------------+
10 rows in set (0.00 sec)
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2020/03/26/openstack%e3%81%ae%e5%90%84%e3%82%b5%e3%83%bc%e3%83%93%e3%82%b9%e3%81%aedb/).*
