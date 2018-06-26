Federation of WebSSO
====================

### Some important definitions:

**Service Provider (SP)**

A system entity that provides services to principals or other system entities, in this case, OpenStack Identity is the Service Provider.

**Identity Provider (IdP)**

A directory service, such as LDAP, RADIUS and Active Directory, which allows users to login with a user name and password, is a typical source of authentication tokens (e.g. passwords) at an identity provider.

**SAML assertion**

Contains information about a user as provided by an IdP. It is an indication that a user has been authenticated.

**Mapping**

Adds a set of rules to map Federation protocol attributes to Identity API objects. An Identity Provider has exactly one mapping specified per protocol.

**Protocol**

Contains information that dictates which Mapping rules to use for an incoming request made by an IdP. An IdP may support multiple protocols. There are three major protocols for federated identity: OpenID, SAML, and OAuth.

**Unscoped token**

Allows a user to authenticate with the Identity service to exchange the unscoped token for a scoped token, by providing a project ID or a domain ID.

**Scoped token**

Allows a user to use all OpenStack services apart from the Identity service.


## Keystone as an Identity Provider
   + config /etc/hosts
      ```
      ...
      10.20.0.100 idp.keystone.demo
      10.20.0.200 sp.keystone.demo
      ```

   + install xmlsec1
      ```
      # yum install xmlsec1 xmlsec1-openssl -y
      ```

   + configure keystone.conf
      ```
      [saml]
      idp_entity_id = http://idp.keystone.demo:5000/idp
      idp_sso_endpoint = http://idp.keystone.demo:5000/login
      ```

   + generate a PKI key and add the cert and key to the paths given in the keystone config
      ```
      # mkdir -p /etc/keystone/ssl/{certs,private}
      # openssl req -x509 -newkey rsa:4096 \
      -keyout /etc/keystone/ssl/private/signing_key.pem \
      -out /etc/keystone/ssl/certs/signing_cert.pem \
      -days 365 -nodes
      ```

   + generate the metadata
      ```
      # keystone-manage saml_idp_metadata > /etc/keystone/saml2_idp_metadata.xml
      ```

   + restart the keystone service
      ```
      # systemctl restart httpd
      ```

   + create service provider(keystone as sp here) resource in identity provider
      ```
      # openstack service provider create keystonesp \
      --auth-url http://sp.keystone.demo:5000/v3/OS-FEDERATION/identity_providers/keystoneidp/protocols/saml2/websso \
      --service-provider-url http://sp.keystone.demo:5000/Shibboleth.sso/SAML2/POST

      +--------------------+-----------------------------------------------------------------------------+
      | Field              | Value                                                                       |
      +--------------------+-----------------------------------------------------------------------------+
      | auth_url           | http://sp.keystone.demo:5000/v3/OS-                                         |
      |                    | FEDERATION/identity_providers/keystoneidp/protocols/saml2/auth/websso       |
      | description        | None                                                                        |
      | enabled            | True                                                                        |
      | id                 | keystonesp                                                                  |
      | relay_state_prefix | ss:mem:                                                                     |
      | sp_url             | http://sp.keystone.demo:5000/Shibboleth.sso/SAML2/POST                      |
      +--------------------+-----------------------------------------------------------------------------+
      ```

