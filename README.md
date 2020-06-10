
# dremio-aws-cloudformation

Deploy the Dremio Data Lake Engine as an AWS Cloudformation Stack

This git repo describes how to launch the Dremio Data Lake Engine into an AWS Cloudformation stack using the AWS CLI commands on a Mac computer or a Windows computer.

## Launch Dremio Cluster from a Windows Client

Click on the "windows" link above and follow the instructions

## Launch Dremio Cluster from a Mac Client

Click on the "mac" link above and follow the instructions

## Advanced Dremio Configuration

The following sections illustrate some advanced configuration.

### Enable AD/LDAP Logging
See: https://docs.dremio.com/deployment/amazon-eks/eks-custom.html?h=logback

```
vi /etc/dremio/logback.xml

<logger name=“com.dremio.extusr.ExternalUserGroupService”>
  <level value=“debug”/>
</logger>
<logger name=“com.dremio.extusr.ldap.LdapUserProvider”>
  <level value=“debug”/>
</logger>
```

### Enable AD/LDAP in dremio.conf
See: https://docs.dremio.com/security/authentication.html

```
vi /etc/dremio/dremio.conf

services: {
  ...
  coordinator.web.auth.type: "ldap",
  coordinator.web.auth.ldap_config: "ad.json",
  ...
}
```

```
vi /etc/dremio/ad.json

{
  "connectionMode": "PLAIN",
  "servers": [{
    "hostname": "<AD/LDAP Server Hostname>",
    "port": 389
  }],
  "names": {
    "bindDN": "CN=admin_user,OU=users,OU=ad,DC=acme,DC=com",
    "bindPassword": "<bind user password>",
    "userDNs": ["uid={0},cn=users,dc=acme,dc=com"],
    "groupDNs": ["cn={0},cn=groups,dc=acme,dc=com"],
    "userFilter": "&(objectClass=posixAccount)",
    "groupFilter": "|(objectClass=posixGroup)(objectClass=sub)",
    "userAttributes": {
      "firstname": "givenName",
      "lastname": "sn",
      "email": "mail"
    },
    "autoAdminFirstUser": true,
    "userGroupRelationship": "GROUP_ENTRY_LISTS_USERS",
    "groupEntryListsUsers": {
      "userEntryUserIdAttribute": "uid",
      "groupEntryUserIdAttribute": "memberUid"
    }
  }
}
```


### Enable Dremio Web UI Encrypted SSL Connections
See: https://docs.dremio.com/deployment/wire-encryption-config.html?h=coordinator.web.ssl.keyStore#full-wire-encryption-enterprise-edition-only

```
vi /etc/dremio/dremio.conf

services: {
...
  coordinator.web.ssl.enabled: true,
  coordinator.web.ssl.auto-certificate.enabled: false,
  coordinator.web.ssl.keyStore: "/etc/dremio/keystore.jks",
  coordinator.web.ssl.keyStorePassword: "file password",
  coordinator.web.port: 443
...
}
```

### Enable Dremio JDBC/ODBC Encrypted SSL Connections
See: https://docs.dremio.com/deployment/wire-encryption-config.html?h=coordinator.web.ssl.keyStore#odbcjdbc-client-encryption--enterprise-edition-only

vi /etc/dremio/dremio.conf

services: {
  ...
  services.coordinator.client-endpoint.ssl.enabled: true
  services.coordinator.client-endpoint.ssl.auto-certificate.enabled: false

  services.coordinator.client-endpoint.ssl.keyStore: "/etc/dremio/keystore.jks"
  services.coordinator.client-endpoint.ssl.keyStorePassword: "file password"
  services.coordinator.client-endpoint.ssl.keyPassword: "key password"
  ...
}


### Enable Dremio Intra-cluster Communication Encrypted SSL Connections
See: https://docs.dremio.com/deployment/wire-encryption-config.html?h=coordinator.web.ssl.keyStore#intracluster-encryption-enterprise-edition-only

vi /etc/dremio/dremio.conf

services: {
  ...
  services.fabric.ssl.enabled: true
  services.fabric.ssl.auto-certificate.enabled: false

  services.fabric.ssl.keyStoreType: "type" # optional; default: JKS
  services.fabric.ssl.keyStore: "/etc/dremio/keystore.jks"
  services.fabric.ssl.keyStorePassword: "file password"
  services.fabric.ssl.keyPassword: "key password"
  ...
}

---
Please direct questions or comments to greg@dremio.com

