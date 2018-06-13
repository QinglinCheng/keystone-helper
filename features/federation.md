Federation
==========

### Some important definitions:

Service Provider (SP)
A system entity that provides services to principals or other system entities, in this case, OpenStack Identity is the Service Provider.
Identity Provider (IdP)
A directory service, such as LDAP, RADIUS and Active Directory, which allows users to login with a user name and password, is a typical source of authentication tokens (e.g. passwords) at an identity provider.
SAML assertion
Contains information about a user as provided by an IdP. It is an indication that a user has been authenticated.
Mapping
Adds a set of rules to map Federation protocol attributes to Identity API objects. An Identity Provider has exactly one mapping specified per protocol.
Protocol
Contains information that dictates which Mapping rules to use for an incoming request made by an IdP. An IdP may support multiple protocols. There are three major protocols for federated identity: OpenID, SAML, and OAuth.
Unscoped token
Allows a user to authenticate with the Identity service to exchange the unscoped token for a scoped token, by providing a project ID or a domain ID.
Scoped token
Allows a user to use all OpenStack services apart from the Identity service.

## Reference

[Federated Identity](https://docs.openstack.org/keystone/latest/advanced-topics/federation/federated_identity.html#configure-apache-to-use-a-federation-capable-authentication-method)
[Federated keystone](https://docs.openstack.org/security-guide/identity/federated-keystone.html)
[Keystone to Keystone federation](https://specs.openstack.org/openstack/keystone-specs/specs/keystone/juno/keystone-to-keystone-federation.html#problem-description)
