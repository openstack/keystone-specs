=================================================
OpenStack Identity API v2.0 Paginated collections
=================================================

To reduce load on the service, list operations return a maximum number
of items at a time. The maximum number of items returned is determined
by the Identity provider. To navigate the collection, you can set the
*``limit``* and *``marker``* parameters in the URI. For example,
?\ *``limit``*\ =100&\ *``marker``*\ =1234. The *``marker``* parameter
is the ID of the last item in the previous list. Items are sorted by
update time. When an update time is not available they are sorted by ID.
The *``limit``* parameter sets the page size. Both parameters are
optional. If the client requests a *``limit``* beyond that which is
supported by the deployment an overLimit (413) fault might be thrown. A
marker with an invalid ID returns an itemNotFound (404) fault.

.. note::

    Paginated collections never return itemNotFound (404) faults when the
    collection is empty - clients should expect an empty collection.

For convenience, collections contain atom "next" and "previous" links.
The first page in the list does not contain a ``previous`` link, the
last page in the list does not contain a ``next`` link. The following
examples illustrate three pages in a collection of tenants. The first
page was retrieved through a **GET** to
``http://identity.api.openstack.org/v2.0/1234/tenants?limit=1``. In
these examples, the *``limit``* parameter sets the page size to a single
item. Subsequent ``next`` and ``previous`` links honor the initial page
size. Thus, a client might follow links to traverse a paginated
collection without having to input the *``marker``* parameter.

**Example: Tenant collection, first page:**

.. code:: javascript

    {
        "tenants": [
            {
                "id": "1234",
                "name": "ACME corp",
                "description": "A description ...",
                "enabled": true
            }
        ],
        "tenants_links": [
            {
                "rel": "next",
                "href": "http://identity.api.openstack.org/v2.0/tenants?limit=1&marker=1234"
            }
        ]
    }



**Example: Tenant collection, second page:**

.. code:: javascript

    {
        "tenants": [
            {
                "id": "3645",
                "name": "Iron Works",
                "description": "A description ...",
                "enabled": true
            }
        ],
        "tenants_links": [
            {
                "rel": "next",
                "href": "http://identity.api.openstack.org/v2.0/tenants?limit=1&marker=3645"
            },
            {
                "rel": "previous",
                "href": "http://identity.api.openstack.org/v2.0/tenants?limit=1"
            }
        ]
    }



**Example: Tenant collection, last page:**

.. code:: javascript

    {
        "tenants": [
            {
                "id": "9999",
                "name": "Bigz",
                "description": "A description ...",
                "enabled": true
            }
        ],
        "tenants_links": [
            {
                "rel": "previous",
                "href": "http://identity.api.openstack.org/v2.0/tenants?limit=1&marker=1234"
            }
        ]
    }



Paginated collections contain a values property that contains the items in the
collections. Links are accessed via the links property. The approach allows for
extensibility of both the collection members and of the paginated collection
itself. It also allows collections to be embedded in other objects as
illustrated below.  Here, a subset of groups are presented within a user.
Clients must follow the "next" link to continue to retrieve additional groups
belonging to a user.

**Example: Paginated roles in user:**

.. code:: javascript

    {
        "user": {
            "OS-ROLE:roles": [
                {
                    "tenantId": "1234",
                    "id": "Admin"
                },
                {
                    "tenantId": "1234",
                    "id": "DBUser"
                }
            ],
            "OS-ROLE:roles_links": [
                {
                    "rel": "next",
                    "href": "http://identity.api.openstack.org/v2.0/tenants/1234/users/u1000/roles?marker=Super"
                }
            ],
            "id": "u1000",
            "username": "jqsmith",
            "email": "john.smith@example.org",
            "enabled": true
        }
    }
