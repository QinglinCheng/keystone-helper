Federation
==========

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
      idp_sso_endpoint = http://idp.keystone.demo:5000/websso
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

   + create service provider resource in identity provider
      ```
      # openstack service provider create keystonesp \
      --auth-url http://sp.keystone.demo:5000/identity/v3/OS-FEDERATION/identity_providers/keystoneidp/protocols/saml2/auth \
      --service-provider-url http://sp.keystone.demo:5000/Shibboleth.sso/SAML2/ECP

      +--------------------+-----------------------------------------------------------------------------+
      | Field              | Value                                                                       |
      +--------------------+-----------------------------------------------------------------------------+
      | auth_url           | http://sp.keystone.demo:5000/identity/v3/OS-                                |
      |                    | FEDERATION/identity_providers/keystoneidp/protocols/saml2/auth              |
      | description        | None                                                                        |
      | enabled            | True                                                                        |
      | id                 | keystonesp                                                                  |
      | relay_state_prefix | ss:mem:                                                                     |
      | sp_url             | http://sp.keystone.demo:5000/Shibboleth.sso/SAML2/ECP                       |
      +--------------------+-----------------------------------------------------------------------------+
      ```

## Keystone as a Service Provider
   + config /etc/hosts
      ```
      ...
      10.20.0.100 idp.keystone.demo
      10.20.0.200 sp.keystone.demo
      ```

   + install [Shibboleth](https://kb.wisc.edu/helpdesk/page.php?id=20454&no_frill=1)

      ```
      # wget http://download.opensuse.org/repositories/security://shibboleth/CentOS_7/security:shibboleth.repo -O /etc/yum.repos.d/shibboleth.repo
      # yum install -y shibboleth.x86_64

      mkdir -p /var/log/shibboleth
      chown -R shibd:shibd /var/log/shibboleth
      systemctl start shibd
      ```

   + configure shibboleth

      **/etc/shibboleth/shibboleth2.xml**
      ```
      <ApplicationDefaults entityID="http://sp.keystone.demo:5000/Shibboleth.sso/SAML2/ECP"
                            REMOTE_USER="eppn persistent-id targeted-id">
      <SSO entityID="http://idp.keystone.demo:5000/idp">
                 SAML2 SAML1
      </SSO>
      <MetadataProvider type="XML" uri="http://idp.keystone.demo:5000/v3/OS-FEDERATION/saml2/metadata"/>
      ```

      **/etc/shibboleth/attribute-map.xml**
      ```
      <Attribute name="openstack_user" id="openstack_user"/>
      <Attribute name="openstack_roles" id="openstack_roles"/>
      <Attribute name="openstack_project" id="openstack_project"/>
      <Attribute name="openstack_user_domain" id="openstack_user_domain"/>
      <Attribute name="openstack_project_domain" id="openstack_project_domain"/>
      ```

   + configure apache httpd

      **/etc/httpd/conf.d/wsgi-keystone.conf**
      ```
      <VirtualHost *:5000>
      ...
         Proxypass Shibboleth.sso !
         <Location /Shibboleth.sso>
             SetHandler shib
         </Location>
         <Location /v3/OS-FEDERATION/identity_providers/keystoneidp/protocols/saml2/auth>
             AuthType shibboleth
             Require valid-user
             ShibRequestSetting requireSession 1
             ShibExportAssertion Off
         </Location>
      </VirtualHost>
      <VirtualHost *:35357>
      ...
         Proxypass Shibboleth.sso !
         <Location /Shibboleth.sso>
             SetHandler shib
         </Location>
         <Location /v3/OS-FEDERATION/identity_providers/keystoneidp/protocols/saml2/auth>
             AuthType shibboleth
             Require valid-user
             ShibRequestSetting requireSession 1
             ShibExportAssertion Off
         </Location>
      </VirtualHost>
      ```

   + config keystone

      **/etc/keystone/keystone.conf**
      ```
      [auth]
      methods = external,password,token,oauth1,application_credential,saml2
      ...
      ```

   + create identity provider resource in service provider

      ```
      # openstack group create federated_users
      # openstack domain create federated_domain
      # openstack role add --group federated_users --domain federated_domain _member_
      ```

      rules.json
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
                              "name": "federated_domain"
                          },
                          "name": "federated_users"
                      }
                  }
              ],
              "remote": [
                  {
                      "type": "openstack_user"
                  }
              ]
          }
      ]
      ```

      ```
      # openstack identity provider create keystoneidp \
      --remote-id http://idp.keystone.demo:5000/idp

      # openstack mapping create --rules rules.json k2kmap
      ```

   + restart services
      ```
      # systemctl restart shibd httpd
      ```

## Testing it all out

   ```
   import os

   from keystoneauth1 import session
   from keystoneauth1.identity import v3
   from keystoneauth1.identity.v3 import k2k

   auth = v3.Password(auth_url=os.environ.get('OS_AUTH_URL'),
                      username=os.environ.get('OS_USERNAME'),
                      password=os.environ.get('OS_PASSWORD'),
                      user_domain_name=os.environ.get('OS_USER_DOMAIN_NAME'),
                      project_name=os.environ.get('OS_PROJECT_NAME'),
                      project_domain_name=os.environ.get('OS_PROJECT_DOMAIN_NAME'))
   password_session = session.Session(auth=auth)
   k2ksession = k2k.Keystone2Keystone(password_session.auth, 'keystonesp',
                                      domain_name='federated_domain')
   auth_ref = k2ksession.get_auth_ref(password_session)
   scoped_token_id = auth_ref.auth_token
   print('Scoped token id: %s' % scoped_token_id)
   ```



## Reference

+ [Federated Identity](https://docs.openstack.org/keystone/latest/advanced-topics/federation/federated_identity.html#configure-apache-to-use-a-federation-capable-authentication-method)
+ [Federated keystone](https://docs.openstack.org/security-guide/identity/federated-keystone.html)
+ [Keystone to Keystone federation](https://specs.openstack.org/openstack/keystone-specs/specs/keystone/juno/keystone-to-keystone-federation.html#problem-description)
+ [Using rpm](https://www.centos.org/docs/5/html/Deployment_Guide-en-US/s1-rpm-using.html)
+ [Demystifying keystone federation](http://www.gazlene.net/demystifying-keystone-federation.html) by [Colleen Murphy](http://www.gazlene.net/)
+ [Shibboleth Service Provider (SP) 2.6 Installation Guide](https://www.switch.ch/aai/guides/sp/installation/?os=centos7)
+ [OpenID Connect Relying Party and OAuth 2.0 Resource Server for Apache HTTP Server 2.x](https://github.com/zmartzone/mod_auth_openidc)
+ [Setup OpenID Connect](https://docs.openstack.org/keystone/latest/advanced-topics/federation/openidc.html)
