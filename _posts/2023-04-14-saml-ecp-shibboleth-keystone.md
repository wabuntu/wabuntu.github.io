---
layout: post
title: "SAML ECP with shibboleth for Keystone"
date: 2023-04-14 00:00:00 +0900
lang: ja
---

## mapping

```json
# cat samltest_mapping.json
[
    {
        "local": [
            {
                "user": {
                    "name": "{0}"
                },
                "group": {
                    "domain": {
                        "name": "Default"
                    },
                    "name": "federated_users"
                }
            }
        ],
        "remote": [
            {
                "type": "REMOTE_USER"
            }
        ]
    }
]
```

## openstack side

```shell
openstack identity provider create --remote-id https://samltest.id/saml/idp samltestopenstack group create federated_usersopenstack project create federated_projectopenstack role add --group federated_users --project federated_project memberopenstack federation protocol create saml2 \--mapping samltest_mapping --identity-provider samltest
```

## keystone.conf

```
# cat /etc/keystone/keystone.conf
# just add saml2
[auth]
methods = external,password,token,oauth1,saml2
[saml2]
remote_id_attribute = Shib-Identity-Provider
```

## cat /etc/keystone/policy.json

if not added

```json
<snip>
    "identity:get_auth_catalog": "",
    "identity:get_auth_projects": "",
    "identity:get_auth_domains": ""
}
```

## keystone

```apache
# cat /etc/apache2/sites-available/keystone.conf
<snip>
<Location /v3/OS-FEDERATION/identity_providers/samltest/protocols/saml2/auth>
    Require valid-user
    AuthType shibboleth
    ShibRequestSetting requireSession 1
    ShibExportAssertion off
    <IfVersion < 2.4>
        ShibRequireSession On
        ShibRequireAll On
    </IfVersion>
</Location>

# probably no need
<Location /v3/auth/OS-FEDERATION/websso/saml2>
    Require valid-user
    AuthType shibboleth
    ShibRequestSetting requireSession 1
    ShibExportAssertion off
    <IfVersion < 2.4>
        ShibRequireSession On
        ShibRequireAll On
    </IfVersion>
</Location>

# probably no need
<Location /v3/auth/OS-FEDERATION/identity_providers/samltest/protocols/saml2/websso>
    Require valid-user
    AuthType shibboleth
    ShibRequestSetting requireSession 1
    ShibExportAssertion off
    <IfVersion < 2.4>
        ShibRequireSession On
        ShibRequireAll On
    </IfVersion>
</Location>

<snip>

# at the end
<Location /Shibboleth.sso>
    SetHandler shib
</Location>
```

```shell
systemctl restart apache2
```

## shibboleth

```shell
apt-get install libapache2-mod-shib2
shib-keygen -y <number of years>
```

```xml
# cat /etc/shibboleth/attribute-map.xml
<Attribute name="urn:oid:0.9.2342.19200300.100.1.1" id="uid" />
# not sure about the followings
<Attribute name="openstack_user" id="openstack_user"/>
<Attribute name="openstack_roles" id="openstack_roles"/>
<Attribute name="openstack_project" id="openstack_project"/>
<Attribute name="openstack_user_domain" id="openstack_user_domain"/>
<Attribute name="openstack_project_domain" id="openstack_project_domain"/>
<Attribute name="openstack_groups" id="openstack_groups"/>
```

```xml
# cat /etc/shibboleth/shibboleth2.xml
<ApplicationDefaults entityID="http://YOURKEYSTONE:5000/shibboleth"
                     REMOTE_USER="uid">
    <snip>
    <SSO entityID="https://samltest.id/saml/idp" ECP="true">
        SAML2 SAML1
    </SSO>
    <MetadataProvider type="XML" uri="https://samltest.id/saml/idp" backingFilePath="federation-metadata.xml" />
```

```shell
systemctl restart shibd
wget https://YOURKEYSTONEWITHOUTPORT/Shibboleth.sso/Metadata
```

- Place to upload your Keystone metadata: https://samltest.id/upload.php

## finding ECP endpoint

```shell
curl -s https://samltest.id/saml/idp | grep urn:oasis:names:tc:SAML:2.0:bindings:SOAP
# Output:
# <SingleSignOnService Binding="urn:oasis:names:tc:SAML:2.0:bindings:SOAP" Location="https://samltest.id/idp/profile/SAML2/SOAP/ECP"/>
```

## rc file

```bash
# cat shib.rc
export OS_AUTH_TYPE=v3samlpassword
export OS_IDENTITY_PROVIDER=samltest
export OS_IDENTITY_PROVIDER_URL=https://samltest.id/idp/profile/SAML2/SOAP/ECP
export OS_PROTOCOL=saml2
export OS_USERNAME=morty
export OS_PASSWORD=ｘｘｘ
export OS_AUTH_URL=http://YOURKEYSTONE:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_PROJECT_NAME=federated_project
export OS_PROJECT_DOMAIN_NAME=Default
openstack federation project list
openstack federation domain list
openstack token issue
```

- Review logs at: https://samltest.id/logs/idp.log


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/04/14/saml-ecp-with-shibboleth-for-keystone/).*
