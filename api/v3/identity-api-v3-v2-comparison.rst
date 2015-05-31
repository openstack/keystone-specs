OpenStack Identity API v3 and v2.0 comparison
=============================================

This document compares several Identity API v3 and v2.0 operations.

API v3 adds the domains feature.

- **Note**: In API v2.0, all operations are on a default domain. So,
  for example, if you create a user, group, or project, it is created
  in the default domain.

Domains represent collections of users, groups, and projects. Each user,
group, and project is owned by exactly one domain. Users, however, can be
associated with multiple projects by granting roles to the user on a project,
including projects owned by other domains. Users can be associated with
multiple domains in the same way. The concepts of ownership and association
are orthogonal.

Create project
--------------

In API v2.0, projects were known as tenants.

API v2.0 tenants are visible as projects in API v3.

API v3 projects are visible as tenants in API v2.0.

cURL (API v2.0)
^^^^^^^^^^^^^^^

::

    curl http://localhost:35357/v2.0/tenants -H "X-Auth-Token: ADMIN" \
      -H "Content-type: application/json" \
      -d '{"tenant":{"name": "tenant02","description": "tenant02 tenant","enabled": true}}'

cURL (API v3)
^^^^^^^^^^^^^

::

    curl http://localhost:5000/v3/projects -H "X-Auth-Token: ADMIN" \
      -H "Content-type: application/json" \
      -d '{"project":{"description": "project02 project","domain_id": "123","enabled": \
           true,"name": "project02"}}'

List users
----------

In API v3, you can request the list of users for either all domains or a
single domain.

cURL (API v2.0)
^^^^^^^^^^^^^^^

::

  curl http://locahost:35357/v2.0/users -H "X-Auth-Token: ADMIN"

cURL (API v3)
^^^^^^^^^^^^^

::

  curl http://localhost:5000/v3/users?domain_id=default -H "X-Auth-Token: ADMIN"

Create user
-----------

cURL (API v2.0)
^^^^^^^^^^^^^^^

::

  curl http://localhost:35357/v2.0/users \
      -d '{"user": {"tenantId": "263fd9","name": "jqsmith","email": \
            "john.smith@example.org","enabled": true}}' \
      -H "Content-type: application/json" -H "X-Auth-Token: ADMIN"

cURL (API v3)
^^^^^^^^^^^^^

::

  curl http://localhost:5000/v3/users \
      -d '{"user": {"default_project_id": "263fd9","description": \
           "jqsmith user","domain_id": "1789d1","enabled": true,"name": \
           "jqsmith","password": "secretsecret"}}' \
      -H "Content-type: application/json" -H "X-Auth-Token: ADMIN"

Create role
-----------

Roles that you create in API v3 or API v2.0 can be seen and used
interchangeably.

cURL (API v2.0)
^^^^^^^^^^^^^^^

::

    curl http://localhost:35357/v2.0/OS-KSADM/roles \
      -H "X-Auth-Token: ADMIN" \
      -d '{"role":{"name":"SysAdm","description":"Role for sys admin"}}' \
      -H "Content-type: application/json"

cURL (API v3)
^^^^^^^^^^^^^

::

    curl http://localhost:5000/v3/roles -H "X-AUTH-TOKEN: ADMIN" \
      -H "Content-type: application/json" \
      -d '{"role":{"name":"SysAdmin","description":"Role for sys admin"}}'

