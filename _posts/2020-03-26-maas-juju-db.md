---
layout: post
title: "MAASとJujuのDB"
date: 2020-03-26 00:00:00 +0900
lang: ja
---

## MAAS

MAASの内部構造を調査する際、ファイルやデータベースの場所が明らかでない場合があります。従来の `/etc/maas` や `/var/lib/maas` ディレクトリが存在しないことがあるのは、Snapパッケージ化されているためです。

### Snapベースの場合

```bash
ubuntu@maas:/etc/init.d$ snap list
Name Version Rev Tracking Publisher Notes
charm 2.7.3 440 stable canonical✓ classic
core 16-2.43.3 8689 stable canonical✓ core
core18 20200124 1668 stable canonical✓ base
juju 2.7.4 10906 stable canonical✓ classic
maas 2.7.0-8235-g.fea3a1678 5177 2.7 canonical✓ -
maas-cli 0.6.5 13 stable canonical✓ -
openstackclients train 38 stable james-page classic
```

MAASコマンドはSnapから実行されます：

```bash
ubuntu@maas:/etc/init.d$ which maas
/snap/bin/maas
```

サービスの管理とログ確認方法：

```bash
ubuntu@maas:/etc/init.d$ snap services
Service Startup Current Notes
maas.supervisor enabled active -

ubuntu@maas:/etc/init.d$ sudo snap restart maas
Restarted.

ubuntu@maas:/etc/init.d$ sudo snap logs maas
2020-03-19T02:02:57Z dhcpd[12848]: DHCPOFFER on 192.168.122.2 to 52:54:00:00:01:aa via ens3
...
```

Snapディレクトリ構造は以下のようになっています：

```bash
ubuntu@maas:/snap/maas$ ls -la
total 8
drwxr-xr-x 4 root root 4096 Mar 19 01:52 .
drwxr-xr-x 10 root root 4096 Mar 15 07:48 ..
drwxr-xr-x 10 root root 142 Feb 21 18:51 4951
drwxr-xr-x 10 root root 142 Mar 16 16:02 5177
lrwxrwxrwx 1 root root 4 Mar 19 01:52 current -> 5177
```

MAASはPostgreSQLを使用していますが、Snap内に含まれています：

```bash
ubuntu@maas:/snap$ ps -fade | grep post
root 14262 14176 0 02:09 ? 00:00:00 python3 /snap/maas/5177/bin/run-postgres
snap_da+ 14295 14262 0 02:09 ? 00:00:00 /snap/maas/5177/bin/postgres -D /var/snap/maas/common/postgres/data -k /var/snap/maas/common/postgres/sockets -h
```

PostgreSQL設定ファイルは以下の場所にあります：

```bash
ubuntu@maas:/snap$ sudo find / -name pg_hba.conf
/var/snap/maas/common/postgres/data/pg_hba.conf
```

### 従来のMAASインストール（Snapなし）

従来のMAASの場合、以下のサービスが実行されます：

```bash
ubuntu@maas:~$ systemctl list-units --type=service | grep maas
maas-dhcpd.service loaded active running MAAS instance of ISC DHCP server for IPv4
maas-proxy.service loaded active running MAAS Proxy
maas-rackd.service loaded active running MAAS Rack Controller
maas-regiond-worker@1.service loaded active running MAAS Region Controller (Worker 1)
maas-regiond-worker@2.service loaded active running MAAS Region Controller (Worker 2)
maas-regiond-worker@3.service loaded active running MAAS Region Controller (Worker 3)
maas-regiond-worker@4.service loaded active running MAAS Region Controller (Worker 4)
maas-regiond.service loaded active exited MAAS Region Controller
```

Regiondの設定ファイルにはデータベース情報が含まれています：

```bash
ubuntu@maas:~$ sudo cat /etc/maas/regiond.conf
database_host: localhost
database_name: maasdb
database_pass: xxxxxxx
database_port: 5432
database_user: maas
maas_url: http://192.168.122.2:5240/MAAS
```

Rackdの設定：

```bash
ubuntu@maas:~$ sudo cat /etc/maas/rackd.conf
cluster_uuid: 378f5de1-02bf-419f-9f4f-6cf36dc6e552
maas_url: http://localhost:5240/MAAS
```

PostgreSQLサービスの確認：

```bash
ubuntu@maas:~$ systemctl list-units --type=service | grep post
postgresql.service loaded active exited PostgreSQL RDBMS
postgresql@9.5-main.service loaded active running PostgreSQL Cluster 9.5-main
```

### PostgreSQLへの接続

PostgreSQL設定ファイルで重要な設定項目を確認できます：

```bash
ubuntu@maas:~$ sudo cat /etc/postgresql/9.5/main/postgresql.conf | grep ...
data_directory = '/var/lib/postgresql/9.5/main'
hba_file = '/etc/postgresql/9.5/main/pg_hba.conf'
ident_file = '/etc/postgresql/9.5/main/pg_ident.conf'
external_pid_file = '/var/run/postgresql/9.5-main.pid'
port = 5432
max_connections = 100
unix_socket_directories = '/var/run/postgresql'
ssl = true
ssl_cert_file = '/etc/ssl/certs/ssl-cert-snakeoil.pem'
ssl_key_file = '/etc/ssl/private/ssl-cert-snakeoil.key'
```

データベースに直接接続できます：

```bash
ubuntu@maas:~$ sudo su - postgres
postgres@maas:~$ psql
psql (9.5.19)
Type "help" for help.

postgres=# \l
List of databases
Name | Owner | Encoding | Collate | Ctype | Access privileges
-----------+----------+----------+-------------+-------------+-----------------------
maasdb | postgres | UTF8 | en_US.UTF-8 | en_US.UTF-8 |
postgres | postgres | UTF8 | en_US.UTF-8 | en_US.UTF-8 |
template0 | postgres | UTF8 | en_US.UTF-8 | en_US.UTF-8 | =c/postgres +
| | | | | postgres=CTc/postgres
template1 | postgres | UTF8 | en_US.UTF-8 | en_US.UTF-8 | =c/postgres +
| | | | | postgres=CTc/postgres

postgres-# \c maasdb
You are now connected to database "maasdb" as user "postgres".

maasdb=# \dt
List of relations
Schema | Name | Type | Owner
--------+--------------------------------------------------+-------+-------
public | auth_group | table | maas
public | auth_group_permissions | table | maas
public | auth_permission | table | maas
public | auth_user | table | maas
public | auth_user_groups | table | maas
public | auth_user_user_permissions | table | maas
public | django_content_type | table | maas
public | django_migrations | table | maas
public | django_session | table | maas
public | django_site | table | maas
public | maasserver_blockdevice | table | maas
public | maasserver_bmc | table | maas
```

ノード情報をクエリできます：

```bash
maasdb=# SELECT hostname,status,power_state FROM maasserver_node;
hostname | status | power_state
---------------------+--------+-------------
node02 | 6 | on
juju-3513fa-1-lxd-2 | 0 | unknown
node04 | 6 | on
juju-3513fa-0-lxd-2 | 0 | unknown
node03 | 6 | on
juju-3513fa-3-lxd-1 | 0 | unknown
juju-3513fa-3-lxd-2 | 0 | unknown
juju-3513fa-2-lxd-0 | 0 | unknown
controller | 6 | on
juju-3513fa-2-lxd-1 | 0 | unknown
juju-3513fa-2-lxd-2 | 0 | unknown
juju-3513fa-1-lxd-1 | 0 | unknown
juju-3513fa-0-lxd-1 | 0 | unknown
node01 | 6 | on
juju-3513fa-3-lxd-0 | 0 | unknown
juju-3513fa-1-lxd-0 | 0 | unknown
juju-3513fa-0-lxd-0 | 0 | unknown
maas | 0 | unknown
```

## Juju

### コントローラーサービス

Jujuコントローラーでは以下のサービスが実行されます：

```bash
ubuntu@controller:~$ sudo snap list
No snaps are installed yet. Try 'snap install hello-world'.

ubuntu@controller:~$ sudo systemctl list-units | grep juju
juju-db.service loaded active running juju state database
jujud-machine-0.service loaded active running juju agent for machine-0
```

### MongoDBサービス

Jujuはデータベースエンジンとして「MongoDBを使用」しており、以下のプロセスで実行されます：

```bash
ubuntu@controller:/etc/init.d$ ps -de | grep mongo
root 1440 1 1 01:53 ? 00:00:58 /usr/bin/mongod --auth --bind_ip_all --dbpath /var/lib/juju/db --ipv6 --journal --keyFile /var/lib/juju/shared-secret --oplogSize 512 --port 37017 --quiet --replSet juju --sslMode requireSSL --sslPEMKeyFile /var/lib/juju/server.pem --sslPEMKeyPassword=xxxxxxx --storageEngine wiredTiger --syslog
```

### 設定ファイル

```bash
ubuntu@controller:/etc/init.d$ cat /etc/default/mongodb
ENABLE_MONGODB=no

ubuntu@controller:/etc$ cat juju-proxy-systemd.conf
# To allow juju to control the global systemd proxy settings,
# create symbolic links to this file from within /etc/systemd/system.conf.d/
# and /etc/systemd/users.conf.d/.
[Manager]
DefaultEnvironment=

ubuntu@controller:/etc$ cat juju-proxy.conf
```

### Jujuエージェントプロセス

```bash
ubuntu@controller:/etc/default$ ps -ef | grep juju
root 818 1 0 01:52 ? 00:00:00 bash /etc/systemd/system/jujud-machine-0-exec-start.sh
root 843 818 1 01:52 ? 00:01:24 /var/lib/juju/tools/machine-0/jujud machine --data-dir /var/lib/juju --machine-id 0 --debug
root 1440 1 1 01:53 ? 00:01:01 /usr/bin/mongod --auth --bind_ip_all --dbpath /var/lib/juju/db --ipv6 --journal --keyFile /var/lib/juju/shared-secret --oplogSize 512 --port 37017 --quiet --replSet juju --sslMode requireSSL --sslPEMKeyFile /var/lib/juju/server.pem --sslPEMKeyPassword=xxxxxxx --storageEngine wiredTiger --syslog
```

### Systemdサービス設定

Jujuエージェント起動スクリプト：

```bash
ubuntu@controller:/etc/systemd/system$ cat jujud-machine-0-exec-start.sh
#!/usr/bin/env bash

# Set up logging.
touch '/var/log/juju/machine-0.log'
chown syslog:adm '/var/log/juju/machine-0.log'
chmod 0640 '/var/log/juju/machine-0.log'
exec >> '/var/log/juju/machine-0.log'
exec 2>&1

# Run the script.
'/var/lib/juju/tools/machine-0/jujud' machine --data-dir '/var/lib/juju' --machine-id 0 --debug
```

Juju DBサービス定義：

```bash
ubuntu@controller:/etc/systemd/system$ cat juju-db.service
[Unit]
Description=juju state database
After=syslog.target
After=network.target
After=systemd-user-sessions.service

[Service]
LimitAS=infinity
LimitCPU=infinity
LimitFSIZE=infinity
LimitMEMLOCK=infinity
LimitNOFILE=64000
LimitNPROC=64000
ExecStart=/usr/bin/mongod --auth --bind_ip_all --dbpath '/var/lib/juju/db' --ipv6 --journal --keyFile '/var/lib/juju/shared-secret' --oplogSize 512 --port 37017 --quiet --replSet juju --sslMode requireSSL --sslPEMKeyFile '/var/lib/juju/server.pem' --sslPEMKeyPassword=ignored --storageEngine wiredTiger --syslog
Restart=on-failure
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```

Jujuエージェントサービス定義：

```bash
ubuntu@controller:/etc/systemd/system$ cat jujud-machine-0.service
[Unit]
Description=juju agent for machine-0
After=syslog.target
After=network.target
After=systemd-user-sessions.service

[Service]
LimitNOFILE=64000
ExecStart=/etc/systemd/system/jujud-machine-0-exec-start.sh
Restart=on-failure
TimeoutSec=300

[Install]
WantedBy=multi-user.target
```

### MongoDBへの接続

MongoDBへのアクセス方法は「Jujuの公式Wikiで」文書化されています：  
https://github.com/juju/juju/wiki/Login-into-MongoDB

エージェント設定からパスワードを取得：

```bash
root@controller:/etc/systemd/system# cat /var/lib/juju/agents/machine-0/agent.conf | grep statepass
statepassword: eyCkU5u....
```

mongoコマンドを使用してデータベースに接続：

```bash
root@controller:/etc/systemd/system# find / -name mongo
/usr/bin/mongo

root@controller:/etc/systemd/system# mongo 127.0.0.1:37017/juju -u "machine-0" -p "<PASSWORD>" --sslAllowInvalidCertificates --ssl --authenticationDatabase admin
MongoDB shell version v3.6.3
connecting to: mongodb://127.0.0.1:37017/juju

juju:PRIMARY> show dbs
admin 0.000GB
blobstore 0.052GB
config 0.000GB
juju 0.006GB
local 0.075GB
logs 0.011GB

juju:PRIMARY> show collections
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2020/03/26/maas%e3%81%a8juju%e3%81%aedb/).*
