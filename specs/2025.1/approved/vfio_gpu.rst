..
 This work is licensed under a Creative Commons Attribution 3.0 Unported
 License.

 http://creativecommons.org/licenses/by/3.0/legalcode

=============================================================================
Allow vfio gpu support with Vendor-Specific VFIO framework
=============================================================================

Problem description
===================

Starting with `kernel 5.16`__ and continuing in subsequent kernels, such as
those in Ubuntu 24.04 (Noble Numbat) and future RHEL 10 releases, the SR-IOV
mechanism for sharing Virtual Functions (VFs) with a guest has transitioned
from using mediated devices (mdev) to the Vendor-Specific VFIO framework.

As a result, Nova should update its GPU device support to accommodate this
new mechanism.

.. __: https://github.com/torvalds/linux/commit/fda49d97f2c4


Use Cases
---------

- As an operator, I want to use SR-IOV GPUs on Linux distributions that
  require the Vendor-Specific VFIO Framework.

- As an operator, I want legacy GPUs without SR-IOV support to remain
  compatible with the mediated device (mdev) framework.

Proposed change
===============

It appears that SR-IOV GPUs, using the Vendor-Specific VFIO framework,
can be integrated with Nova by leveraging the current PCI passthrough and
SR-IOV support, along with several changes proposed in this specification.

Following the GPU documentation, such as the one provided by `Nvidia`__
, users should have VGUs configured and accessible as PCI (VF) devices
identified by a PCI address.

Subsequently, by following the `Nova documentation on attaching physical PCI
devices to guests`__, users should arrive at a main configuration PCI section
that specifies device attributes and aliases.

.. note::

   pci.report_in_placement needs to be enable to use traits.

As an example:

.. code-block:: shell

  [pci]
  device_spec = { "vendor_id": "10de", "product_id": "25b6", "address": "0000:25:00.4", "traits": "NVIDIA_A16_16A"}

  alias = { "vendor_id": "10de", "product_id": "25b6", "device_type": "type-VF", "name": "a16_16a", "traits": "NVIDIA_A16_16A" }

Creating a VM based on the configuration above will include the following
snippet in the XML definition:

.. code-block:: shell

  <hostdev mode='subsystem' type='pci' managed='yes'>
    <driver name='vfio'/>
    <source>
      <address domain='0x0000' bus='0x25' slot='0x00' function='0x4'/>
    </source>
    <alias name='hostdev0'/>
    <address type='pci' domain='0x0000' bus='0x00' slot='0x05' function='0x0'/>
  </hostdev>


Fixing managed mode:

The managed mode must be changed from "managed='yes'" to "managed='no'"
otherwise the server crashes.

The proposed solution is to add a ``migratable`` tag to the device
specification. This tag will serve the following purposes:

- Set managed='no' in the XML definition.
- Identify that the device supports live migration (details of this
  functionality will be covered in another specification).

When this tag is encountered by the `PCI resource tracker`__, the
corresponding information will be stored in the respective PciDevice
object under the extra_info field. This will enable the code responsible
for generating the XML definition to set the managed mode to no.

Additionally, it will record an internal trait,
``COMPUTE_PCI_DEVICE_MIGRATABLE``, in the resource provider.

.. note::

  By using this solution, the PciDevice object remains unchanged, which
  simplifies potential feature backports.


Sanitize device specification:

As part of the initialization process, checks are performed to ensure the
correctness of the device specification. Given that many GPUs can be added
to this configuration, these checks will be extended to ensure that PCI
device addresses are not duplicated.

Device specification resource class:

Since the resource class can be defined by the user, no changes will be made.

- Should we provide guidelines for naming resource classes?
- Should we document a recommendation to avoid using vgpu as a resource
  class name?

Device specification traits:

Documentation will be added to provide guidelines for naming traits. We
may recommend that trait names include the vGPU types for better clarity
and organization.


Display management:
  An optional display attribute may be used to enable using a vgpu device
  as a display device for the guest. Supported values are either on or off
  (default). There is also an optional ramfb attribute with values of either
  on or off (default). When enabled, the ramfb attribute provides a memory
  framebuffer device to the guest. This framebuffer allows the vgpu to be used
  as a boot display before the gpu driver is loaded within the guest. ramfb
  requires the display attribute to be set to on.

There is a constraint to activate these settings for only one VGPU, even
if multiple VGUs are attached to a VM.

In this initial implementation, we have chosen not to address this constraint
and instead align with the existing mdev implementation.




.. __: https://docs.nvidia.com/vgpu/latest/grid-vgpu-user-guide/index.html#creating-sriov-vgpu-device-vendor-specific-vfio-framework
.. __: https://docs.openstack.org/nova/latest/admin/pci-passthrough.html
.. __: https://github.com/openstack/nova/blob/f98f414f971b6c897bf48781a579730419b5a93d/nova/compute/pci_placement_translator.py#L597-L600


Alternatives
------------

Instead of using a tag in the device specification, it may be possible to
detect live migratability in a generic way through sysfs or libvirt. This
capability could then be added to the pci_passthrough_devices JSON returned
by the libvirt driver to the compute manager, and subsequently stored in
the PciDevice object's extra_info field.

TBD: explain possible solution without pci in placement


REST API impact
---------------

NA

Data model impact
-----------------

NA

Security impact
---------------

NA

Notifications impact
--------------------

NA


Other end user impact
---------------------

The user is fully responsible for configuring the following:

- vGPU types.
- Device specifications and aliases.
- Flavors.

Performance Impact
------------------

TBD need to speak about "Gibi's bugs"

Other deployer impact
---------------------

None

Developer impact
----------------

None

Upgrade impact
--------------

Users should review the following:

- The configuration of vGPU types.
- The configuration of device specifications and aliases.
- Update their flavors to specify the correct aliases."

Implementation
==============

Assignee(s)
-----------

Primary assignee:
  uggla (Rene Ribaud)

Main contributors:
  bauzas (Sylvain Bauzas)

Feature Liaison
---------------

Feature liaison:
  uggla

Work Items
----------

TBD

Dependencies
============

None

Testing
=======

- TBD
- Functional libvirt driver tests
- Integration Tempest tests ?

Documentation Impact
====================

Extensive admin and user documentation will be provided.

References
==========

History
=======

.. list-table:: Revisions
   :header-rows: 1

   * - Release Name
     - Description
   * - Epoxy
     - Introduced
