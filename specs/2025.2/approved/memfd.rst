..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

==========================================
Enabling memfd Support in OpenStack Nova
==========================================

Spec for blueprint: memfd-support
https://blueprints.launchpad.net/nova/+spec/memfd-support


Shared memory plays a crucial role in enabling advanced features such as
OVS-DPDK and virtio-fs in OpenStack Nova. These features depend heavily on
efficient memory sharing between processes to achieve optimal performance.

The primary objective of this specification is to introduce support for the
`memfd` source type for memory backing for instances using shared memory.

Problem description
===================

Currently, Nova supports two mechanisms for configuring shared memory:

- Huge pages
- File-backed memory

File-backed memory involves using a regular file on the compute node, and this
feature is configured at the compute service level (i.e., it applies to all
instances on the host).

With the introduction of QEMU 5.0.0 and libvirt 6.9.0, memory backing can
now be configured via file descriptors using Linux's `memfd` mechanism. This
approach no longer requires NUMA-specific configuration and can be enabled on
a per-instance basis.

Because `memfd` does not require any special host setup, it offers a simpler
and more flexible alternative, making it an ideal candidate for the default
memory backing configuration.

Key advantages include:

- No memory oversubscription limitations as seen with hugepages.
- Avoidance of NUMA-related complexities that hugepages often introduce.
- Reduced operational overhead compared to file-backed memory, which
  typically requires host aggregates to isolate file-backed and non-file-backed
  nodes—since it's an all-or-nothing setup at the host level.

Additionally, memfd offers better performance than traditional file-backed
memory due to the absence of filesystem overhead.

Use Cases
---------

- As an operator, I want to use shared memory on a per-instance basis without
  requiring additional host-level configuration.

- As an operator, I want the flexibility to choose between different types of
  memory backing (e.g., regular files or memfd).

Proposed change
===============

Description:
------------

Traits
******

The `COMPUTE_MEM_BACKING_FILE` trait was introduced by:
`libvirt-virtiofs-attach-manila-shares.html`__
It indicates that the compute node is configured with `file_backed_memory`.

This spec proposes introducing the following new traits:

- `COMPUTE_MEMFD`: Indicates that QEMU ≥ 5.0.0 and libvirt ≥ 6.9.0 are
  available, and thus `memfd` is supported.
  Note: This is supported by default. The trait is introduced to keep a
  consistent approach for all memory backing settings.

- `COMPUTE_HUGE_PAGES`: Indicates that huge pages are configured and usable
  on this compute host.

.. __: https://specs.openstack.org/openstack/nova-specs/specs/2023.2/approved/libvirt-virtiofs-attach-manila-shares.html


These traits will be used by a scheduler utility function to translate the
user's memory backing preferences into the appropriate required/forbidden
trait filters, ensuring that instances are placed only on compatible hosts.

See the sections below for details on which hosts will be considered or
rejected based on the selected memory backing configuration.

Support for memfd-backed memory
*******************************

To explicitly select `memfd` as the memory backend, users can specify either:

- `hw:memory_backend=memfd` as a flavor extra spec
- `hw_memory_backend=memfd` as an image property

This will result in the following being added to the instance's domain XML:

.. code-block:: xml

  <memoryBacking>
    <source type='memfd'/>
    <access mode='shared'/>
  </memoryBacking>

Host selection:

- Hosts with the `COMPUTE_MEMFD` trait will be selected.
- Hosts with the `COMPUTE_MEM_BACKING_FILE` trait will be rejected.
- Hosts with the `COMPUTE_HUGE_PAGES` trait will be selected only if the
  instance explicitly requests a hugepage-based memory configuration (e.g.,
  hw:mem_page_size=1GB or 2MB).

Support for file-backed memory
******************************

To explicitly select file-backed memory using regular files, users can specify:

- `hw:memory_backend=file` as a flavor extra spec
- `hw_memory_backend=file` as an image property

Resulting domain XML:

.. code-block:: xml

  <memoryBacking>
    <source type='file'/>
    <access mode='shared'/>
    <allocation mode="immediate"/>
    <discard/>
  </memoryBacking>

Host selection:

- Only hosts with the `COMPUTE_MEM_BACKING_FILE` trait will be selected.

Support for huge pages
**********************

To use huge pages, users can specify:

- `hw:memory_backend=hugepage` (extra spec)
- `hw_memory_backend=hugepage` (image property)

A scheduler filter will ensure the instance is scheduled to a host with
the `COMPUTE_HUGE_PAGES` trait.

Enabling this option will implicitly set hw:mem_page_size=large to ensure
the use of large memory pages.

Host selection:

- Only hosts with the `COMPUTE_HUGE_PAGES` trait will be selected.


.. note::

  The `MEMORY_PAGE_SIZE_SMALL` and `MEMORY_PAGE_SIZE_LARGE` traits, as
  described in this `specification`__, could be introduced later as child
  traits of `COMPUTE_HUGE_PAGES`.


.. __: https://specs.openstack.org/openstack/nova-specs/specs/victoria/approved/numa-topology-with-rps.html#numa-nodes-being-nested-resource-providers

Disabling shared memory backing
*******************************

Users who prefer to avoid memory-backed storage can set this option to
anonymous, which disables all shared memory backends, including memfd
(the default in the future):

- `hw:memory_backend=anonymous` (extra spec)
- `hw_memory_backend=anonymous` (image property)

This results in no `<memoryBacking>` section in the domain XML.

Host selection:

- Only hosts with the `COMPUTE_MEMFD` trait will be selected.
- Hosts with `COMPUTE_HUGE_PAGES` or `COMPUTE_MEM_BACKING_FILE` will be
  excluded


Metadata
********

The hw_memory_backend option and its configured value will be exposed in
the instance metadata. This allows users to understand how the guest memory
is backed on the host, which can help with performance analysis, debugging,
and adapting application behavior based on the memory characteristics.


Alternatives
------------

By default, instances will use shared memory with `memfd` as the backing type.
This would generate the following domain XML:

.. code-block:: xml

  <memoryBacking>
    <source type='memfd'/>
    <access mode='shared'/>
  </memoryBacking>

Users can opt out by setting `hw:memory_backend=anonymous`.

Data model impact
-----------------

None.

REST API impact
---------------

None.

Security impact
---------------

None.

Notifications impact
--------------------

None.

Other end user impact
---------------------

None.

Performance Impact
------------------

None.

Other deployer impact
---------------------

None.

Developer impact
----------------

None.

Upgrade impact
--------------

This spec proposes introducing new traits that will be unconditionally reported
by the libvirt driver when the corresponding feature is supported on the host
(e.g., memfd is available, hugepages are configured, or file-backed memory
is enabled).

Since anonymous memory backing remains the default, the behavior is designed
to be safe and non-disruptive:

- No upgrade impact: Only flavors that explicitly use the new hw_memory_backend
  extra spec will trigger the new trait-based scheduling logic.
- Opt-in behavior: The feature is activated via extra specs. During rolling
  upgrades, only upgraded hosts will be considered for instances using it.
- No need to opt out: Because the default remains anonymous, there’s no
  changed behavior unless explicitly requested.
- Compatibility preserved: Existing configurations that use
  traits:COMPUTE_MEM_BACKING_FILE=required will continue to function without
  changes.
- Manila users: Manila share-based workloads will not automatically benefit
  from this change, which is acceptable given their distinct requirements.


Implementation
==============

Assignee(s)
-----------

Primary assignee:

- Uggla (René Ribaud)

Other contributors:
  N/A

Feature Liaison
---------------

Feature liaison:
  N/A

Work Items
----------

- Add logic to generate the appropriate libvirt domain XML
- Ensure compatibility with migration and rebuild workflows (if memfd is used
  as default)
- Add validation for conflicting memory backend settings

Dependencies
============

Current minimum QEMU and libvirt versions support this feature.

Testing
=======

- Functional tests for the libvirt driver, including domain XML generation.

Documentation Impact
====================

Extensive documentation updates for both operators and users will be provided,
covering new image and flavor properties and default behavior.

References
==========

`https://libvirt.org/formatdomain.html#memory-backing`__ reference
documentation.

.. __: https://libvirt.org/formatdomain.html#memory-backing

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - 2025.2 Flamingo
     - Introduced
