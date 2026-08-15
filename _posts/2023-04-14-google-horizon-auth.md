---
layout: post
title: "Using Google for Horizon Authentication"
date: 2023-04-14 00:00:00 +0900
lang: ja
---

## Sample Sites

- [https://indigo-dc.gitbook.io/keystone-with-oidc-documentation/admin-iam-conf/admin-google-conf](https://indigo-dc.gitbook.io/keystone-with-oidc-documentation/admin-iam-conf/admin-google-conf)
- [https://docs.openstack.org/keystone/latest/admin/federation/configure_federation.html#setting-up-openid-connect](https://docs.openstack.org/keystone/latest/admin/federation/configure_federation.html#setting-up-openid-connect)

## Notes

The SP (your keystone) ENTITY ID can be anything and doesn't need to be resolvable, but it must match what you registered in Google.

- YOURENTITYID: `http://keystone.example.com`

## Google Side

- [https://console.developers.google.com/apis/credentials](https://console.developers.google.com/apis/credentials)
- Create Credentials => OAuth Client ID => Application Type: Web Application
- Name: YOURENTITYID
- Authorized JavaScript origins: YOURENTITYID
- Authorized redirect URIs: `YOURENTITYID:5000/v3/auth/OS-FEDERATION/websso/oidc/redirect`

## Mapping

```json
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
                    "name": "google_group"
                }
            }
        ],
        "remote": [
            {
                "type": "OIDC-email"
            }
        ]
    }
]
```

## OpenStack Side

```bash
apt-get install libapache2-mod-auth-openidc #on keystone machine
openstack group create google_group
openstack project create google_project
openstack role add admin --group google_group --project google_project
openstack mapping create google_mapping --rules google_mapping.json
openstack identity provider create google_idp --remote-id https://accounts.google.com
openstack federation protocol create oidc --identity-provider google_idp --mapping google_mapping
```

## Keystone.conf

```ini
insecure_debug = True

[auth]
methods = external,password,token,oauth1,oidc
oidc = keystone.auth.plugins.mapped.Mapped

[oidc]
remote_id_attribute = HTTP_OIDC_ISS

[federation]
remote_id_attribute = HTTP_OIDC_ISS
trusted_dashboard = http://YOURHORIZON/horizon/auth/websso/
```

## Apache

```apache
LoadModule auth_openidc_module /usr/lib/apache2/modules/mod_auth_openidc.so

<VirtualHost *:5000>
    OIDCClaimPrefix "OIDC-"
    OIDCResponseType "id_token"
    OIDCScope "openid email profile"
    OIDCProviderMetadataURL https://accounts.google.com/.well-known/openid-configuration
    OIDCOAuthVerifyJwksUri https://www.googleapis.com/oauth2/v3/certs
    OIDCClientID xxx-xxx.apps.googleusercontent.com
    OIDCClientSecret GOCSPX-xxxx
    OIDCCryptoPassphrase ?
    OIDCRedirectURI http://YOURENTITYID:5000/v3/OS-FEDERATION/identity_providers/google_idp/protocols/oidc/auth

    <LocationMatch /v3/OS-FEDERATION/identity_providers/.*?/protocols/oidc/auth>
        AuthType openid-connect
        Require valid-user
        LogLevel debug
    </LocationMatch>

    <Location /v3/auth/OS-FEDERATION/websso/oidc>
        Require valid-user
        AuthType openid-connect
        LogLevel debug
    </Location>

    <Location /v3/auth/OS-FEDERATION/identity_providers/google_idp/protocols/oidc/websso>
        Require valid-user
        AuthType openid-connect
        LogLevel debug
    </Location>
</VirtualHost>
```

## Policy.json

If the following are not added:

```json
"identity:get_auth_catalog": "",
"identity:get_auth_projects": "",
"identity:get_auth_domains": ""
```

## Horizon

```python
CACHES = {
    'default': {
        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
        'LOCATION': '127.0.0.1:11211',
    },
}

EMAIL_BACKEND = 'django.core.mail.backends.console.EmailBackend'
OPENSTACK_HOST = "YOURENTITYID"
OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

WEBSSO_ENABLED = True
WEBSSO_CHOICES = (
    ("credentials", _("Keystone Credentials")),
    ("oidc", _("Google Login"))
)
WEBSSO_INITIAL_CHOICE = "credentials"
```

## Restart Both Keystone and Horizon

```bash
systemctl restart apache2
```


---
*Originally published at [wabuntu.wordpress.com](https://wabuntu.wordpress.com/2023/04/14/using-google-for-horizon-authentication/).*
