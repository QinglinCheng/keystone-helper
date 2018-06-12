Application Credential
======================

**Only project scoped identity can crate application credential**

   ```
   export OS_USERNAME=cql
   export OS_PASSWORD=passw0rd
   export OS_USER_DOMAIN_NAME=Default
   export OS_PROJECT_NAME=admin
   export OS_PROJECT_DOMAIN_NAME=Default
   export OS_AUTH_URL=http://10.20.0.33:35357/v3
   export OS_IDENTITY_API_VERSION=3
   ```


   ```
   OS_TOKEN=$(openstack token issue -f value -c id)
   ```

### Create

   ```
   curl -g -H "Content-Type: application/json" \
   -H "X-Auth-Token: $OS_TOKEN" \
   -X POST http://10.20.0.33:35357/v3/users/03a1b1c4cecc405dbcee3970ff1653a0/application_credentials -d '
   {
       "application_credential": {
           "name": "cql_app_creds1",
           "description": "Application credential for test from user admin.",
           "expires_at": "2018-06-09T18:30:59Z",
           "roles": [
               {"name": "admin"},
               {"name": "_member_"}
           ],
           "unrestricted": false
       }
   }' | python -mjson.tool
   ```

### List

   ```
   curl -g -H "Content-Type: application/json" \
   -H "X-Auth-Token: $OS_TOKEN" \
   -X GET http://10.20.0.33:35357/v3/users/03a1b1c4cecc405dbcee3970ff1653a0/application_credentials \
   | python -mjson.tool
   ```


### Show

   ```
   curl -g -H "Content-Type: application/json" \
   -H "X-Auth-Token: $OS_TOKEN" \
   -X GET http://10.20.0.33:35357/v3/users/1503b85a0f444377b1ebcb0d3a671ed0/application_credentials/29ec6e113fd04d0dbfec149e60d94070 \
   | python -mjson.tool
   ```

### Delete

   ```
   curl -g -H "Content-Type: application/json" \
   -H "X-Auth-Token: $OS_TOKEN" \
   -X DELETE http://10.20.0.33:35357/v3/users/1503b85a0f444377b1ebcb0d3a671ed0/application_credentials/e6418e59685b461dbcfe61af532064d9 \
   | python -mjson.tool
   ```

### Auth Token Issue

   ```
   curl -g -i -H "Content-Type: application/json" -X POST http://10.20.0.33:35357/v3/auth/tokens -d '
   {
       "auth": {
           "identity": {
               "methods": [
                   "application_credential"
               ],
               "application_credential": {
                   "id": "29ec6e113fd04d0dbfec149e60d94070",
                   "secret": "ipSC8EAd6ATTNgseqW2Wz0CIOXTTH8FlZcXm-rlcYNrVLnvJ2AiTc3696d142MLYSNMEtsZc29cNpIedRGuf4g"
               }
           }
       }
   }'
   ```

```
#!/usr/bin/env bash

export OS_AUTH_TYPE=v3applicationcredential
export OS_AUTH_URL={{ auth_url }}
export OS_IDENTITY_API_VERSION=3
export OS_REGION_NAME="{{ region|shellfilter }}"
export OS_INTERFACE={{ interface }}
export OS_APPLICATION_CREDENTIAL_ID={{ application_credential_id }}
export OS_APPLICATION_CREDENTIAL_SECRET={{ application_credential_secret }}
```

# Reference

+ [Openstac keystone specs](http://specs.openstack.org/openstack/keystone-specs/)
+ [Review of Application Credential](https://review.openstack.org/#/q/status:merged+project:openstack/keystone+branch:master+topic:bp/application-credentials)
+ [Openrc file](https://review.openstack.org/#/c/552145/24/openstack_dashboard/dashboards/identity/application_credentials/templates/application_credentials/openrc.sh.template)
