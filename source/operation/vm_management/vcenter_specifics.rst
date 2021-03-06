.. _vcenter_specifics:

================================================================================
vCenter Specifics
================================================================================ 

vCenter VM and VM Templates
================================================================================

To learn how to use VMs and VM Templates you can read the :ref:`Managing Virtual Machines Instances <vm_guide_2>` and :ref:`Managing Virtual Machine Templates <vm_guide>`, but first take into account the following considerations.

.. _vm_template_definition_vcenter:

In order to manually create a VM Template definition in OpenNebula that represents a vCenter VM Template, the following attributes are needed:

+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|     Operation      |                                                                                                                                                                     Note                                                                                                                                                                     |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| CPU                | Physical CPUs to be used by the VM. This does not have to relate to the CPUs used by the vCenter VM Template, OpenNebula will change the value accordingly                                                                                                                                                                                   |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| MEMORY             | Physical Memory in MB to be used by the VM. This does not have to relate to the CPUs used by the vCenter VM Template, OpenNebula will change the value accordingly                                                                                                                                                                           |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| NIC                | Check :ref:`VM template reference <template_network_section>`. Valid MODELs are: virtuale1000, virtuale1000e, virtualpcnet32, virtualsriovethernetcard, virtualvmxnetm, virtualvmxnet2, virtualvmxnet3.                                                                                                                                      |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| DISK               | Check :ref:`VM template reference <reference_vm_template_disk_section>`. Take into account that all images are persistent, as explained in :ref:`vCenter Datastore Setup <vcenter_ds>`.                                                                                                                                                      |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| GRAPHICS           | Multi-value - Only VNC supported, check the  :ref:`VM template reference <io_devices_section>`.                                                                                                                                                                                                                                              |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| PUBLIC_CLOUD       | Multi-value. TYPE must be set to vcenter, and VM_TEMPLATE must point to the uuid of the vCenter VM that is being represented                                                                                                                                                                                                                 |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| SCHED_REQUIREMENTS | NAME="name of the vCenter cluster where this VM Template can instantiated into a VM". See :ref:`VM Scheduling section <vm_scheduling_vcenter>` for more details.                                                                                                                                                                             |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| CONTEXT            | All :ref:`sections <template_context>` will be honored except FILES. You can find more information about contextualization in the :ref:`vcenter Contextualization <vcenter_contextualization>` section.                                                                                                                                      |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| KEEP_DISKS_ON_DONE | (Optional) Prevent OpenNebula from erasing the VM disks upon reaching the done state (either via shutdown or cancel)                                                                                                                                                                                                                         |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| VCENTER_DATASTORE  | By default, the VM will be deployed to the datastore where the VM Template is bound to. This attribute allows to set the name of the datastore where this VM will be deployed.  This can be overwritten explicitly at deployment time from the CLI or Sunstone. More information in the :ref:` vCenter Datastore Setup Section <vcenter_ds>` |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| RESOURCE_POOL      | By default, the VM will be deployed to the default resource pool. If this attribute is set, its value will be used to confine this the VM in the referred resource pool. Check :ref:`this section <vcenter_resource_pool>` for more information.                                                                                             |
+--------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

After a VM Template is instantiated, the life-cycle of the resulting virtual machine (including creation of snapshots) can be controlled through OpenNebula. Also, all the operations available in the :ref:`vCenter Admin view <vcenter_view>` can be performed, including:

- network management operations like the ability to attach/detach network interfaces
- capacity (CPU and MEMORY) resizing
- VNC connectivity
- Attach/detach VMDK images as disks

The following operations are not available for vCenter VMs:

- migrate
- livemigrate

The monitoring attributes retrieved from a vCenter VM are:

- ESX_HOST
- GUEST_IP
- GUEST_STATE
- VMWARETOOLS_RUNNING_STATUS
- VMWARETOOLS_VERSION
- VMWARETOOLS_VERSION_STATUS

VM Template Cloning Procedure
--------------------------------------------------------------------------------

OpenNebula uses VMware cloning VM Template procedure to instantiate new Virtual Machines through vCenter. From the VMware documentation:

-- Deploying a virtual machine from a template creates a virtual machine that is a copy of the template. The new virtual machine has the virtual hardware, installed software, and other properties that are configured for the template.

A VM Template is tied to the host where the VM was running, and also the datastore(s) where the VM disks where placed. By default, the VM will be deployed in that datastore where the VM Template is bound to, although another datastore can be selected at deployment time. Due to shared datastores, vCenter can instantiate a VM Template in any of the hosts belonging to the same cluster as the original one.

OpenNebula uses several assumptions to instantiate a VM Template in an automatic way:

- **diskMoveType**: OpenNebuls instructs vCenter to "move only the child-most disk backing. Any parent disk backings should be left in their current locations.". More information `here <https://www.vmware.com/support/developer/vc-sdk/visdk41pubs/ApiReference/vim.vm.RelocateSpec.DiskMoveOptions.html>`__

- Target **resource pool**: OpenNebula uses the default cluster resource pool to place the VM instantiated from the VM template, unless VCENTER_RESOURCE_POOL variable defined in the OpenNebula host template, or the tag RESOURCE_POOL is present in the VM Template inside the PUBLIC_CLOUD section.

Saving a VM Template: Save As Persistent
--------------------------------------------------------------------------------

At the time of deploying a VM Template, a flag can be used to create a new VM Template out of the VM.

.. prompt:: bash $ auto

  $ onetemplate instantiate <tid> --persistent

Whenever the VM life-cycle ends, OpenNebula will instruct vCenter to create a new vCenter VM Template out of the VM, with the settings of the VM including any new disks or network interfaces added through OpenNebula. Any new disk added to the VM will be saved as part of the template, and when a new VM is spawnm from this new VM Template the disk will be cloned by OpenNebula (ie, it will no longer be persistent).

A new OpenNebula VM Template will also be created pointing to this new VM Template, so it can be instantiated through OpenNebula.

This functionality is very useful to create new VM Templates from a original VM Template, changing the VM configuration and/or installing new software, to create a complete VM Template catalog.

.. _vm_scheduling_vcenter:

VM Scheduling
--------------------------------------------------------------------------------

OpenNebula scheduler should only chose a particular OpenNebula host for a OpenNebula VM Template representing a vCenter VM Template, since it most likely only would be available in a particular vCenter cluster.

Since a vCenter cluster is an aggregation of ESX hosts, the ultimate placement of the VM on a particular ESX host would be managed by vCenter, in particular by the `Distribute Resource Scheduler (DRS) <https://www.vmware.com/es/products/vsphere/features/drs-dpm>`__.

In order to enforce this compulsory match between a vCenter cluster and a OpenNebula/vCenter VM Template, add the following to the OpenNebula VM Template:

.. code::

    SCHED_REQUIREMENTS = "NAME=\"name of the vCenter cluster where this VM Template can instantiated into a VM\""

In Sunstone, a host abstracting a vCenter cluster will have an extra tab showing the ESX hosts that conform the cluster.

.. image:: /images/host_esx.png
    :width: 90%
    :align: center


vCenter Images
================================================================================

You can follow the :ref:`Managing Images Section <img_guide>` to learn how to manage images, considering that all images in vCenter are persistent and that VMDK snapshots are not supported as well as the following considerations.

vCenter VMDK images managed by OpenNebula are always persistent, ie, OpenNebula won't copy them for new VMs, but rather the originals will be used. This means that only one VM can use one image at the same time.

vCenter VM Templates with already defined disks will be imported without this information in OpenNebula. These disks will be invisible for OpenNebula, and therefore cannot be detached from the VMs. The imported Templates in OpenNebula can be updated to add new disks from VMDK images imported from vCenter (please note that these will always be persistent).

There are three ways of adding VMDK representations in OpenNebula:

- Upload a new VMDK from the local filesystem
- Register an existent VMDK image already in the datastore
- Create a new empty datablock

The following image template attributes need to be considered for vCenter VMDK image representation in OpenNebula:

+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|    Attribute     |                                                                                                                                                                                                   Description                                                                                                                                                                                                   |
+==================+=================================================================================================================================================================================================================================================================================================================================================================================================================+
| ``PERSISTENT``   | Must be set to 'YES'                                                                                                                                                                                                                                                                                                                                                                                            |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``PATH``         | This can be a i) local filesystem path to a VMDK to be uploaded or ii) path of an existing VMDK file in the vCenter datastore. In case ii) a vcenter:// prefix must be used (for instance, an image win10.vmdk in a Windows folder should be set to vcenter://Windows/win10.vmdk)                                                                                                                               |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``ADAPTER_TYPE`` | Possible values (careful with the case): lsiLogic, ide, busLogic.                                                                                                                                                                                                                                                                                                                                               |
|                  | More information `in the VMware documentation <http://pubs.vmware.com/vsphere-60/index.jsp#com.vmware.wssdk.apiref.doc/vim.VirtualDiskManager.VirtualDiskAdapterType.html>`__                                                                                                                                                                                                                                   |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| ``DISK_TYPE``    | The type of disk has implications on performance and occupied space. Values (careful with the case): delta,eagerZeroedThick,flatMonolithic,preallocated,raw,rdm,rdmp,seSparse,sparse2Gb,sparseMonolithic,thick,thick2Gb,thin. More information `in the VMware documentation <http://pubs.vmware.com/vsphere-60/index.jsp?topic=%2Fcom.vmware.wssdk.apiref.doc%2Fvim.VirtualDiskManager.VirtualDiskType.html>`__ |
+------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

VMDK images in vCenter datastores can be:

- Cloned
- Deleted
- Hotplugged to VMs

Images can be imported from the vCenter datastore using the **onevcenter** tool:

.. prompt:: text $ auto

    $ onevcenter images datastore1 --vuser oneadmin@vsphere.local --vpass Pantufl4. --vcenter vcenter.vcenter3

    Connecting to vCenter: vcenter.vcenter3...done!

    Looking for Images...done!

      * Image found:
          - Name      : win-test-context-fixed2 - datastore1
          - Path      : win-test-context-fixed2/win-test-context-fixed2.vmdk
          - Type      : VmDiskFileInfo
        Import this Image [y/n]? n

      * Image found:
          - Name      : windows-2008R2 - datastore1
          - Path      : windows/windows-2008R2.vmdk
          - Type      : VmDiskFileInfo
        Import this Image [y/n]? y
        OpenNebula image 0 created!

..warning: Images spaces are not allowed for import

.. note: By default, OpenNebula checks the datastore capacity to see if the image fits. This may cause a "Not enough space in datastore" error. To avoid this error, disable the datastore capacity check before importing images. This can be changes in /etc/one/oned.conf, using the DATASTORE_CAPACITY_CHECK set to "no".

