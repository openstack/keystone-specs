..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode


============
Project Tags
============

Blueprint `project-tags
<https://blueprints.launchpad.net/keystone/+spec/project-tags>`_

Allow projects in keystone to be taggable with simple strings. This will
make projects more easily categorizable and filterable. This specification
follows closely with neutron's [0]_ resource tag implementation and will
follow the guidelines for using tags set by the OpenStack API Working
Group [1]_.


Problem Description
===================

Today, operators must rely on naming conventions or the extras implementation
in order to tag or categorize projects within keystone. This extras field is
not clearly defined and does not provide a API standard way to retrieve or
filter this data.


Use Case: Project Organization via Tagging
------------------------------------------

In the case of a large private cloud containing many projects with lifespans
that can range from days to a much longer time, there is a need to be able to
tag and categorize projects based on how they are intended to be used, who is
using them, and how long they will exist. The operator would be able to query
all projects with a specific tag without resorting to naming projects with
particular conventions. Additionally, on project modification or deletion,
proper cleanup of these tags could occur without the need of setting up an
external tracking system. Currently, using an external system for keeping
track of tags for projects is difficult for larger scale deployments, where
projects change frequently and are often deleted within a few days.


Proposed Changes
================

* Add a new table ``project_tag`` to map strings to projects, uniqueness will
  be enforced between ``project_id`` and ``name``:

  .. code:: SQL

     CREATE TABLE `project_tag` (
        `project_id`       varchar(64) NOT NULL,
        `name`             varchar(60) NOT NULL
     )

* There is a limit of 50 tags on a project and each tag cannot exceed 60
  characters.  See [2]_.

* Add tags field as part of default response when listing projects and
  showing a single project.

* Add new API calls to interact with both project tags for basic CRUD
  functionality.

* Tags are URL-safe and should match the following regular expression:

  .. code:: bash

      ^[^,/]*$

Tags will have the following restrictions as set by the API Working
Group [1]_:

.. Note::

    * Tags are case sensitive
    * ‘/’ is not allowed to be in a tag name
    * Commas are not allowed to be in a tag name in order to simplify
      requests that specify lists of tags


The schema for project tags would be:

.. code-block:: json

   {
       "type": "array",
       "items": {
           "type": "string",
           "minLength": 1,
           "maxLength": 60,
           "pattern": "^[^,/]*$"
       },
       "maxItems": 50
   }

where if a tag would be added to a project and the max number of items
is exceeded, a 400 Bad Request would be returned.

New API Calls
=============

List all tags for a project
---------------------------

**Request:** ``GET /v3/projects/{project_id}/tags``

**Parameters**

* ``project_id`` - The project ID.

**Response**

* 200 - OK
* 404 - Does not exist

**Response Body**

.. code:: json

    {
      "tags": ["foo", "bar"]
    }

Check if a project contains a specified tag
-------------------------------------------

**Request:** ``GET /v3/projects/{project_id}/tags/{value}``

**Parameters**

* ``project_id`` - The project ID.
* ``value`` - The tag value.

**Response**

* 204 - No Content
* 404 - Tag or Project does not exist

Add single tag to a project
---------------------------

Creates the specified tag and adds it to the list in the project

**Request:** ``PUT /v3/projects/{project_id}/tags/{value}``

**Parameters**

* ``project_id`` - The project ID.
* ``value`` - The tag value.

**Response**

* 201 - Created
* 404 - Project does not exist

**Response Header**

* `Location: http://identity:5000/v3/projects/{project_id}/tags/{value}`

Modify tag list for a project
-----------------------------

Modifies the tags for a project. Any existing tags not
specified will be deleted.

**Request:** ``PUT /v3/projects/{project_id}/tags``

.. code:: json

    {
      "tags": ["foo", "bar"]
    }

**Parameters**

* ``project_id`` - The project ID.

**Response**

* 200 - OK
* 404 - Project does not exist

**Response Body**

.. code:: json

    {
      "links": {
        "next": null,
        "previous": null,
        "self": "http://identity:5000/v3/projects"
      },
      "projects": [
        {
          "description": "Test Project",
          "domain_id": "default",
          "enabled": true,
          "id": "3d4c2c82bd5948f0bcab0cf3a7c9b48c",
          "links": {
            "self": "http://identity:5000/v3/projects/3d4c2c82bd5948f0bcab0cf3a7c9b48c"
          },
          "name": "demo",
          "tags": ["foo", "bar"]
        }
      ]
    }

Delete single tag from project
------------------------------

Remove a single tag from a project.

**Request:** ``DELETE /v3/projects/{project_id}/tags/{value}``

**Parameters**

* ``project_id`` - The project ID.
* ``value`` - The tag to be deleted

**Response**

* 204 - Tags deleted
* 404 - Tag or Project was not found

Remove all tags from a project
------------------------------

Remove the entire tag list from the given project.

**Request:** ``DELETE /v3/projects/{project_id}/tags``

**Parameters**

* ``project_id`` - The project ID.

**Response**

* 204 - Tags deleted
* 404 - Project was not found


Filtering and Searching by Tags
===============================

To search projects by their tags, the client should send a GET request to
the collection URL and include query string parameters that define the
query. These arguments can be combined with other arguments, such as those
that perform additional filtering outside of tags. The recommended query
string arguments for filtering tags are:

.. list-table::
   :widths: 100 250
   :header-rows: 1

   * - Tag Query
     - Description
   * - tags
     - Projects that contain all of the specified tags
   * - tags-any
     - Projects that contain at least one of the specified tags
   * - not-tags
     - Projects that do not contain exactly all of the specified tags
   * - not-tags-any
     - Projects that do not contain any one of the specified tags


To request the list of projects that have a single tag, `tags` argument
should be set to the desired tag name. Example will return all projects
with the "foo" tag:

.. code-block:: bash

   GET /v3/projects?tags=foo

To request the list of projects that have two or more tags, the `tags`
argument should be set to the list of tags, separated by commas. In this
situation, the tags given must all be present for a project to be included
in the query result. Example will return all projects that have the "foo"
and "bar" tags:

.. code-block:: bash

   GET /v3/projects?tags=foo,bar

To request the list of projects that have at least one tag from a given list,
the ``tags-any`` argument should be set to the list of tags, separated
by commas. In this situation as long as one of the given tags is present,
the project will be included in the query result. Example that returns the
projects that have the “foo” OR “bar” tag:

.. code-block:: bash

   GET /v3/projects?tags-any=foo,bar

To request the list of projects that do not have a list of tags, the
``not-tags`` argument should be set to the list of tags, separated by commas.
In this situation, the tags given must all be absent for a project to be
included in the query result. Example that returns the projects that
do not have the “foo” nor the “bar” tag:

.. code-block:: bash

   GET /v3/projects?not-tags=foo,bar

To request the list of projects that do not have at least one of a list of
tags, the ``not-tags-any`` argument should be set to the list of tags,
separated by commas. In this situation, as long as one of the given tags
is absent, the project will be included in the query result. Example
that returns the projects that do not have the “foo” tag, or do not have
the “bar” tag:

.. code-block:: bash

   GET /v3/projects?not-tags-any=foo,bar

The ``tags``, ``tags-any``, ``not-tags`` and ``not-tags-any`` arguments can
be combined to build more complex queries. Example that returns any projects
that have the “foo” and “bar” tags, plus at least one of “red” and “blue”.

.. code-block:: bash

   GET /v3/projects?tags=foo,bar&tags-any=red,blue


Alternatives
============

1. Store the tags external to keystone.

   * Pro: No change to keystone required.
   * Con: Requires an external tool or work-around. If using an external
     system, this requires yet another tool to maintain and keep track of.
     Any updates for resources, such as deletion of a project, will require
     the corresponding tag data to be kept up-to-date in the external system.
     For larger scale deployments with many temporary projects that are
     regularly purged, this is both clumsy and difficult to maintain.

2. Store the tags in ``extra`` column.

   * Pro: No additional SQL table modification is needed.
   * Con: The ``extra`` column currently stores some ancillary data,
     e.g. user's email address. Allowing the API to modify this data
     may cause conflicts. There is not a standard API way to manipulate
     this data and the data is not indexed.

3. Use a naming schema for projects to categorize them.

   * Pro: No change in keystone is required.
   * Con: If a project is going to need multiple "tags" in its name, this
     can cause project names to become very large as well as
     ugly/unrecognizable. For a large cloud with many projects, this is
     unrealistic.


Security Impact
===============

Typically, only the project admin should be able to create/edit the tags
for a project. This is to prevent unprivileged users from viewing or changing
any existing tags, which could possibly denote administrative functions.

The policy rules for tags will follow the rules set for /v3/projects.


Notifications Impact
====================

Any added API calls should emit the proper notifications.


Other End User Impact
=====================

New API's will be available to operators with appropriate role(s) to
manipulate keystone resource tags.


Performance Impact
==================

There will be no performance impact on existing APIs.  There may be database
performance impact if operators allow for a large number of tags to be
associated with projects.

Other Deployer Impact
=====================

None.

Developer Impact
================

None.

Implementation
==============

Assignee(s)
-----------

Primary assignee:

  * Gage Hugo - gagehugo@gmail.com (IRC gagehugo)

Other contributors:

  * Samuel Pilla - sp516w@att.com (IRC spilla)
  * Tin Lam - tinlam@gmail.com (IRC lamt)

Work Items
----------

1. Implement the new API calls
2. Add relevent tests
3. Update all appropriate documentation/api-ref
4. Update keystone-client/openstack-cli

Dependencies
============

None.

Documentation Impact
====================

Update ``api-ref`` documents to show the usage of the API's.


References
==========

.. [0] `<http://docs.openstack.org/newton/networking-guide/ops-resource-tags.html>`_

.. [1] `<https://specs.openstack.org/openstack/api-wg/guidelines/tags.html>`_

.. [2] `<https://specs.openstack.org/openstack/nova-specs/specs/kilo/approved/tag-instances.html#rest-api-impact>`_
