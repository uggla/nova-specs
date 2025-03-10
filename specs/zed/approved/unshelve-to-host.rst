..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=================================
Allow unshelve to a specific host
=================================

https://blueprints.launchpad.net/nova/+spec/unshelve-to-host

This blueprint proposes to allow administrator to specify ``destination_host``
to unshelve a shelved offloaded server.

Problem description
===================
Currently, an instance can only be unshelved to a specific availability zone.
The proposal is to extend the unshelve behavior allowing an instance to be
unshelved to a specific host.

Use Cases
---------
As a administrator, I want to specify a destination host when executing
unshelve on a shelved-offloaded instance.

Proposed change
===============
Add a new microversion to extend the unshelve API behavior to support a
specific destination host.

Add ``destination_host`` attribute to unshelve Action request body.



Add 2 checks:

- Ensure the user is an administrator.
- Ensure the instance state is ``shelved_offloaded``.


Alternatives
------------
The current proposal rejects unshelve to a specific host if the instance state
is not shelve_offloaded.
Alternatively, a request to unshelve to a specific host would change the
instance state to shelve_offloaded automatically. So the user would not have to
worry about the initial instance state.

Data model impact
-----------------
None

REST API impact
---------------

Change the validation schema making ``availability_zone`` and
``destination_host`` mutually exclusive.
So the api can be called with only 3 ways:

- {'unshelve': null}   (Keep compatibility with previous microversions)

or

- {'unshelve': {'availability_zone': <string>}}

or

- {'unshelve': {'destination_host': <fqdn>}}


Everything else is not allowed, examples:

- {'unshelve': {}}
- {'unshelve': {'destination_host': <fqdn>, 'destination_host': <fqdn>}}
- {'unshelve': {'foo': <string>}}

Security impact
---------------
None

Notifications impact
--------------------
None

Other end user impact
---------------------
The python-novaclient and python-openstackclient will be updated.

Performance Impact
------------------
None

Other deployer impact
---------------------
None

Developer impact
----------------
None

Upgrade impact
--------------
None

Implementation
==============

Assignee(s)
-----------
Primary assignee:
  Uggla (rene-ribaud)

Feature Liaison
---------------
Feature liaison:
  sbauza

Work Items
----------
* Add a new microversion to the unshelve to a specific host
  (unshelve Action) API
* Add related tests

Dependencies
============
None

Testing
=======
- Add related unit tests
- Add related functional tests

Documentation Impact
====================
Add docs that mention unshelve to host after the microversion.

References
==========
None

History
=======
.. list-table:: Revisions
    :header-rows: 1

    * - Release Name
      - Description
    * - Zed
      - Introduced
