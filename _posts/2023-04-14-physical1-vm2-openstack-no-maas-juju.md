---
layout: post
title: "物理１台、VM２台でOpenStack（juju,MAASなし)"
date: 2023-04-14 00:00:00 +0900
lang: ja
---

## 概要

このブログ記事は、1台の物理マシンと2台の仮想マシンを使用してOpenStackを構築するための詳細なガイドです。Juju や MAAS は使用せず、手動で設定します。

## 環境構成

### 仮想マシン構成

**Controller Node (manual-xenial-01)**
- vCPU: 6
- RAM: 8192 MB
- ディスク: 64 GB (qcow2)
- ホスト名: controller
- IP: 192.168.122.101

**Compute Node (manual-xenial-02)**
- vCPU: 6
- RAM: 8192 MB
- ディスク: 64 GB (qcow2)
- ホスト名: compute
- IP: 192.168.122.102

### 基盤サービスのインストール

#### NTP (Chrony)
```bash
sudo apt install chrony
```

#### データベース (MariaDB)
```bash
sudo apt install mariadb-server
```

#### メッセージキュー (RabbitMQ)
```bash
sudo apt install rabbitmq-server
sudo rabbitmqctl add_user openstack ubuntu
sudo rabbitmqctl set_permissions openstack ".*" ".*" ".*"
```

#### キャッシュ (Memcached)
```bash
sudo apt install memcached python-memcache
```

## Keystone (認証サービス)

### データベース設定

```bash
sudo mysql -u root -p
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'ubuntu';
```

### インストール

```bash
sudo apt install keystone apache2 libapache2-mod-wsgi
```

### 環境変数ファイル

**admin-openrc.sh**
```bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=ubuntu
export OS_AUTH_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

**demo-openrc.sh**
```bash
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=ubuntu
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

### プロジェクトとユーザー作成

```bash
export OS_TOKEN=1303f775b7ef407523e7
export OS_URL=http://controller:35357/v3
export OS_IDENTITY_API_VERSION=3

openstack domain create --description "Default Domain" default
openstack project create --domain default --description "Admin Project" admin
openstack user create --domain default --password-prompt admin
openstack role create admin
openstack role add --project admin --user admin admin

openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" demo
openstack user create --domain default --password-prompt demo
openstack role create user
openstack role add --project demo --user demo user
```

### Keystoneサービスエンドポイント

```bash
openstack service create --name keystone --description "OpenStack Identity" identity

openstack endpoint create --region RegionOne identity public http://controller:5000/v3
openstack endpoint create --region RegionOne identity internal http://controller:5000/v3
openstack endpoint create --region RegionOne identity admin http://controller:35357/v3
```

## Glance (イメージサービス)

### インストール

```bash
sudo apt install glance
```

### イメージアップロード

```bash
wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img

source ./admin-openrc.sh
openstack image create "cirros" \
  --file cirros-0.3.4-x86_64-disk.img \
  --disk-format qcow2 \
  --container-format bare \
  --public
```

## Nova (コンピュートサービス)

### コントローラーノード設定

#### インストール

```bash
sudo apt install nova-api nova-conductor nova-consoleauth \
  nova-novncproxy nova-scheduler
```

#### データベース設定

```bash
mysql -u root -p
CREATE DATABASE nova_api;
CREATE DATABASE nova;
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'ubuntu';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'ubuntu';
```

#### 設定ファイル例

```ini
[DEFAULT]
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
logdir=/var/log/nova
state_path=/var/lib/nova
lock_path=/var/lock/nova
force_dhcp_release=True
libvirt_use_virtio_for_bridges=True
verbose=True
ec2_private_dns_show_ip=True
api_paste_config=/etc/nova/api-paste.ini
enabled_apis=osapi_compute,metadata
rpc_backend = rabbit
auth_strategy = keystone
my_ip = 192.168.122.101
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[oslo_messaging_rabbit]
rabbit_host = controller
rabbit_userid = openstack
rabbit_password = ubuntu

[keystone_authtoken]
auth_uri = http://controller:5000
auth_url = http://controller:35357
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = ubuntu

[vnc]
vncserver_listen = $my_ip
vncserver_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[api_database]
connection = mysql+pymysql://nova:ubuntu@controller/nova_api

[database]
connection = mysql+pymysql://nova:ubuntu@controller/nova
```

#### DB同期

```bash
sudo su
su -s /bin/sh -c "nova-manage api_db sync" nova
su -s /bin/sh -c "nova-manage db sync" nova
```

### コンピュートノード設定

#### インストール

```bash
sudo apt install nova-compute
```

#### 設定例

```ini
[DEFAULT]
enabled_apis=ec2,osapi_compute,metadata
auth_strategy = keystone
my_ip = 192.168.122.102
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[vnc]
enabled = True
vncserver_listen = 0.0.0.0
vncserver_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
api_servers = http://controller:9292

[libvirt]
virt_type = qemu

[neutron]
url = http://controller:9696
auth_url = http://controller:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = ubuntu
```

#### サービス確認

```bash
openstack compute service list -c Binary -c Host -c State
openstack hypervisor list
```

## Neutron (ネットワークサービス)

### コントローラーノード設定

#### インストール

```bash
apt install neutron-server neutron-plugin-ml2 \
  neutron-linuxbridge-agent neutron-l3-agent \
  neutron-dhcp-agent neutron-metadata-agent
```

#### ネットワーク作成

```bash
source ./admin-openrc.sh

# 外部ネットワーク
neutron net-create ext-net --router:external \
  --provider:physical_network provider \
  --provider:network_type flat

neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=192.168.122.200,end=192.168.122.240 \
  --disable-dhcp --gateway 192.168.122.1 192.168.122.0/24

# デモネットワーク
source demo-openrc.sh
neutron net-create demo-net
neutron subnet-create demo-net 192.168.123.0/24 \
  --name demo-subnet --gateway 192.168.123.1 \
  --dns-nameserver 1.1.1.1

# ルーター作成
neutron router-create demo-router
neutron router-interface-add demo-router demo-subnet
neutron router-gateway-set demo-router ext-net
```

### ネットワークエージェント確認

```bash
source admin-openrc.sh
neutron agent-list
```

## インスタンス作成

```bash
source demo-openrc.sh

nova boot --flavor m1.small --image cirros \
  --nic net-id=123182c1-2948-4be0-918e-db8220e05458 \
  --security-group a4b17a49-a32d-4eec-945b-c161ffe3707e cir1
```

## Dashboard (Horizon)

### 設定例

```python
ALLOWED_HOSTS = ['*', ]

OPENSTACK_HOST = "controller"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '192.168.122.101:11211',
    },
}

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "volume": 2,
    "compute": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = 'default'
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

TIME_ZONE = "Asia/Tokyo"
```

#### 再起動

```bash
systemctl restart apache2 memcached
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/04/14/%e7%89%a9%e7%90%86%ef%bc%91%e5%8f%b0%e3%80%81vm%ef%bc%92%e5%8f%b0%e3%81%a7openstack%ef%bc%88jujumaas%e3%81%aa%e3%81%97/).*
