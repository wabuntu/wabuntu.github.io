---
layout: post
title: "MySQL HA Cluster"
date: 2023-04-14 00:00:00 +0900
lang: ja
---

下記２つを参考に

- https://jaas.ai/hacluster
- https://wiki.ubuntu.com/ServerTeam/OpenStackHA

## 初期状態はこんな感じ

| Unit | Workload | Agent | Machine | Public address | Ports | Message |
|------|----------|-------|---------|----------------|-------|---------|
| ceph-mon/0 | active | idle | 1/lxd/0 | 192.168.122.18 | | Unit is ready and clustered |
| ceph-mon/1* | active | idle | 2/lxd/0 | 192.168.122.15 | | Unit is ready and clustered |
| ceph-mon/2 | active | idle | 3/lxd/0 | 192.168.122.14 | | Unit is ready and clustered |
| ceph-osd/0 | active | idle | 1 | 192.168.122.5 | | Unit is ready (1 OSD) |
| ceph-osd/1 | active | idle | 2 | 192.168.122.6 | | Unit is ready (1 OSD) |
| ceph-osd/2* | active | idle | 3 | 192.168.122.7 | | Unit is ready (1 OSD) |
| ceph-radosgw/0* | active | idle | 0/lxd/0 | 192.168.122.10 | 80/tcp | Unit is ready |
| cinder/0* | active | idle | 1/lxd/1 | 192.168.122.17 | 8776/tcp | Unit is ready |
| cinder-ceph/0* | active | idle | | 192.168.122.17 | | Unit is ready |
| glance/0* | active | idle | 2/lxd/1 | 192.168.122.8 | 9292/tcp | Unit is ready |
| keystone/0* | active | idle | 3/lxd/1 | 192.168.122.12 | 5000/tcp | Unit is ready |
| mysql/0* | active | idle | 0/lxd/1 | 192.168.122.9 | 3306/tcp | Unit is ready |
| neutron-api/0* | active | idle | 1/lxd/2 | 192.168.122.19 | 9696/tcp | Unit is ready |
| neutron-gateway/0* | active | idle | 0 | 192.168.122.4 | | Unit is ready |
| ntp/0* | active | idle | | 192.168.122.4 | 123/udp | ntp: Ready |
| nova-cloud-controller/0* | active | idle | 2/lxd/2 | 192.168.122.16 | 8774/tcp | Unit is ready |
| nova-compute/0 | active | idle | 1 | 192.168.122.5 | | Unit is ready |
| neutron-openvswitch/2 | active | idle | | 192.168.122.5 | | Unit is ready |
| ntp/3 | active | idle | | 192.168.122.5 | 123/udp | ntp: Ready |
| nova-compute/1 | active | idle | 2 | 192.168.122.6 | | Unit is ready |
| neutron-openvswitch/1 | active | idle | | 192.168.122.6 | | Unit is ready |
| ntp/2 | active | idle | | 192.168.122.6 | 123/udp | ntp: Ready |
| nova-compute/2* | active | idle | 3 | 192.168.122.7 | | Unit is ready |
| neutron-openvswitch/0* | active | idle | | 192.168.122.7 | | Unit is ready |
| ntp/1 | active | idle | | 192.168.122.7 | 123/udp | ntp: Ready |
| openstack-dashboard/0* | active | idle | 3/lxd/2 | 192.168.122.13 | 80/tcp,443/tcp | Unit is ready |
| rabbitmq-server/0* | active | idle | 0/lxd/2 | 192.168.122.11 | 5672/tcp | Unit is ready |

空きIPは122.18以降

```bash
ubuntu@maas:~/bundle$ juju deploy hacluster mysql-hacluster
Located charm "cs:hacluster-65".
Deploying charm "cs:hacluster-65".
```

HAClusterを追加

```bash
ubuntu@maas:~/bundle$ juju deploy -n 2 mysql
Located charm "cs:mysql-58".
Deploying charm "cs:mysql-58".
ERROR cannot add application "mysql": application already exists
```

上記失敗。

```bash
ubuntu@maas:~/bundle$ juju add-unit mysql -n 2
```

上記は新規マシンを追加しようとして失敗。（予備で使ってないマシンがない限り）

```bash
ubuntu@maas:~/bundle$ juju add-unit mysql -n 2 –to lxd:1,lxd:2
```

Mysql/0がLXDの０にDeployされているので、LXD1と２を指定してみた。

```bash
ubuntu@maas:~/bundle$ juju status

mysql/0* active idle 0/lxd/1 192.168.122.9 3306/tcp Unit is ready
mysql/4 maintenance executing 1/lxd/3 192.168.122.21 (install) installing charm software
mysql/5 maintenance executing 2/lxd/3 192.168.122.20 (install) installing charm software
```

行けそうなのでVIPとRelationを設定

```bash
ubuntu@maas:~/bundle$ juju set mysql vip="192.168.122.30"
ERROR unrecognized command: juju set
ubuntu@maas:~/bundle$ juju config mysql vip="192.168.122.30"
ubuntu@maas:~/bundle$ juju add-relation mysql mysql-hacluster
```

それぞれがつながり始めた

```bash
ubuntu@maas:~/bundle$ juju status

mysql/0* active idle 0/lxd/1 192.168.122.9 3306/tcp Unit is ready
mysql-hacluster/2 maintenance executing 192.168.122.9 (config-changed) Setting up corosync
mysql/4 active idle 1/lxd/3 192.168.122.21 3306/tcp Unit is ready
mysql-hacluster/1 maintenance executing 192.168.122.21 (config-changed) Setting up corosync
mysql/5 active idle 2/lxd/3 192.168.122.20 3306/tcp Unit is ready
mysql-hacluster/0* maintenance executing 192.168.122.20 (config-changed) Setting up corosync
neutron-api/0* active idle 1/lxd/2 192.168.122.19 9696/tcp Unit is ready
```

死んだ

```bash
keystone/0* maintenance executing 3/lxd/1 192.168.122.12 5000/tcp (config-changed) Updating NRPE configuration
mysql/0* blocked idle 0/lxd/1 192.168.122.9 3306/tcp MySQL is down. Sequence Number: 90506. Safe To Bootstrap: 0
mysql-hacluster/2 active idle 192.168.122.9 Unit is ready and clustered
mysql/4 blocked idle 1/lxd/3 192.168.122.21 3306/tcp Services not running that should be: mysql
mysql-hacluster/1 active idle 192.168.122.21 Unit is ready and clustered
mysql/5 blocked idle 2/lxd/3 192.168.122.20 3306/tcp Services not running that should be: mysql
mysql-hacluster/0* active idle 192.168.122.20 Unit is ready and clustered
```

このような状況

```bash
ubuntu@juju-3513fa-0-lxd-1:~$ systemctl status mysql
● mysql.service – LSB: Start and stop the mysql (Percona XtraDB Cluster) daemon
Loaded: loaded (/etc/init.d/mysql; bad; vendor preset: enabled)
Active: failed (Result: exit-code) since Fri 2020-03-27 02:05:32 UTC; 12min ago
Docs: man:systemd-sysv-generator(8)
Process: 819 ExecStart=/etc/init.d/mysql start (code=exited, status=1/FAILURE)

Mar 27 02:04:47 juju-3513fa-0-lxd-1 systemd[1]: Starting LSB: Start and stop the mysql (Percona XtraDB Cluster) daemon…
Mar 27 02:04:48 juju-3513fa-0-lxd-1 mysql[819]: * Starting MySQL (Percona XtraDB Cluster) database server mysqld
Mar 27 02:05:32 juju-3513fa-0-lxd-1 mysql[819]: * The server quit without updating PID file (/var/run/mysqld/mysqld.pid).
Mar 27 02:05:32 juju-3513fa-0-lxd-1 mysql[819]: …fail!
Mar 27 02:05:32 juju-3513fa-0-lxd-1 systemd[1]: mysql.service: Control process exited, code=exited status=1
Mar 27 02:05:32 juju-3513fa-0-lxd-1 systemd[1]: Failed to start LSB: Start and stop the mysql (Percona XtraDB Cluster) daemon.
Mar 27 02:05:32 juju-3513fa-0-lxd-1 systemd[1]: mysql.service: Unit entered failed state.
Mar 27 02:05:32 juju-3513fa-0-lxd-1 systemd[1]: mysql.service: Failed with result 'exit-code'.
ubuntu@juju-3513fa-0-lxd-1:~$ sudo systemctl stop mysql
ubuntu@juju-3513fa-0-lxd-1:~$ sudo systemctl start mysql
Job for mysql.service failed because the control process exited with error code. See "systemctl status mysql.service" and "journalctl -xe" for details.
```

```bash
ubuntu@juju-3513fa-0-lxd-1:~$ sudo systemctl start mysql
Job for mysql.service failed because the control process exited with error code. See "systemctl status mysql.service" and "journalctl -xe" for details.
ubuntu@juju-3513fa-0-lxd-1:~$ journalctl -xe
Mar 27 02:19:27 juju-3513fa-0-lxd-1 mysql[22237]: …fail!
Mar 27 02:19:27 juju-3513fa-0-lxd-1 systemd[1]: mysql.service: Control process exited, code=exited status=1
Mar 27 02:19:27 juju-3513fa-0-lxd-1 systemd[1]: Failed to start LSB: Start and stop the mysql (Percona XtraDB Cluster) daemon.
— Subject: Unit mysql.service has failed
— Defined-By: systemd
— Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
—
— Unit mysql.service has failed.
—
— The result is failed.
Mar 27 02:19:27 juju-3513fa-0-lxd-1 systemd[1]: mysql.service: Unit entered failed state.
Mar 27 02:19:27 juju-3513fa-0-lxd-1 systemd[1]: mysql.service: Failed with result 'exit-code'.
Mar 27 02:19:27 juju-3513fa-0-lxd-1 sudo[22234]: pam_unix(sudo:session): session closed for user root
Mar 27 02:19:28 juju-3513fa-0-lxd-1 mysql_monitor(res_mysql_monitor)[23792]: ERROR: Not enough arguments [1] to ocf_log.
Mar 27 02:19:28 juju-3513fa-0-lxd-1 mysql_monitor(res_mysql_monitor)[23799]: MYSQL IS NOT RUNNING:
```

もとにもどそう

```bash
ubuntu@maas:~$ juju remove-relation mysql mysql-hacluster
ubuntu@maas:~$ juju config mysql –reset vip
ubuntu@maas:~$ juju remove-unit mysql/4 mysql/5
removing unit mysql/4
removing unit mysql/5
ubuntu@maas:~$ juju remove-application hacluster mysql-hacluster
removing application hacluster failed: application "hacluster" not found
removing application mysql-hacluster
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/04/14/mysql-ha-cluster/).*
