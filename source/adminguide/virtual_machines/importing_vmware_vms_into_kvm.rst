.. Licensed to the Apache Software Foundation (ASF) under one
   or more contributor license agreements.  See the NOTICE file
   distributed with this work for additional information#
   regarding copyright ownership.  The ASF licenses this file
   to you under the Apache License, Version 2.0 (the
   "License"); you may not use this file except in compliance
   with the License.  You may obtain a copy of the License at
   http://www.apache.org/licenses/LICENSE-2.0
   Unless required by applicable law or agreed to in writing,
   software distributed under the License is distributed on an
   "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
   KIND, either express or implied.  See the License for the
   specific language governing permissions and limitations
   under the License.

.. note:: This functionality requires **virt-v2v** (https://www.libguestfs.org/virt-v2v.1.html) binary installed on destination cluster hosts (needs to be installed manually as it's not a dependency of the CloudStack agent during the agent installation).

Requirements on the KVM hosts
-----------------------------

The CloudStack agent does not install the virt-v2v binary as a dependency. The virt-v2v binary must be installed manually on KVM hosts, or the migration will fail.

The virt-v2v output (progress) is logged in the CloudStack agent logs, to help administrators track the progress on the Instance conversion processes. The verbose mode for virt-v2v can be enabled by adding the following line to /etc/cloudstack/agent/agent.properties and restart cloudstack-agent:

    ::

        dnf install virt-v2v

        echo "virtv2v.verbose.enabled=true" >> /etc/cloudstack/agent/agent.properties  
    
        systemctl restart cloudstack-agent


Installing virt-v2v on Ubuntu KVM hosts does not install nbdkit which is required in the conversion of VMware VCenter guests. To install it, please execute:

    ::

        apt install nbdkit


Supported Distributions for KVM Hypervisor:


.. cssclass:: table-striped table-bordered table-hover

========================    ========================
Linux Distribution          Supported Versions
========================    ========================
Alma Linux                  8, 9
Red Hat Enterprise Linux    8, 9
Rocky Linux                 8, 9
Ubuntu                      22.04 LTS, 24.04 LTS
========================    ========================


Importing Windows VMs from VMware requires installing the virtio drivers for Windows on the hypervisor hosts for the virt-v2v conversion.

On (RH)EL hosts:

    ::

        yum install virtio-win

You can also install the RPM manually from https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.noarch.rpm


For Debian-based distributions:

Ubuntu don’t seem to ship the virtio-win package with drivers, which causes virt-v2v not to convert the VMWare Windows guests to virtio profiles. This could result in slow IDE drives and Intel E1000 NICs. As a workaround, we can follow the below steps to install the package from the RPM on all KVM hosts running the virt-v2v:

    ::

        apt install virtio-win (if the package is not available, then manual steps will be required to install the virtio drivers for windows)
        
        wget https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.noarch.rpm

        # install “alien” which can convert rpms to debs
        apt -y install alien

        # the conversion, can take a while
        alien -d virtio-win.noarch.rpm

        # install the resulting deb
        dpkg -i virtio-win*.deb

In addition to this, we need to install the below package as well to avoid the error “virt-v2v: error: One of rhsrvany.exe or pvvxsvc.exe is missing in /usr/share/virt-tools“.

    :: 
     
        wget -nd -O srvany.rpm https://kojipkgs.fedoraproject.org//packages/mingw-srvany/1.1/4.fc38/noarch/mingw32-srvany-1.1-4.fc38.noarch.rpm

        alien -d srvany.rpm

        dpkg -i *srvany*.deb


The OVF tool (ovftool) must be installed on the destination KVM hosts if the hosts should export VM files (OVF) from vCenter. If not, the management server exports them (the management server doesn't require ovftool installed).

Steps to install ovftool

Download the ovftool from https://developer.broadcom.com/tools/open-virtualization-format-ovf-tool/latest

    ::
       
       unzip VMware-ovftool-4.6.3-24031167-lin.x86_64.zip -d /usr/local/
       
       #create a soft link 

       ln -s /usr/local/ovftool/ovftool /usr/local/bin/ovftool

If you are hitting the following error when running ovftool, install the dependecy

./ovftool.bin: error while loading shared libraries: libnsl.so.1: cannot open shared object file: No such file or directory

     ::
     
        dnf install libnsl


Usage
-----

In the UI, Virtual Machines to import from VMware are listed in *Tools > Import-Export Instances* section, selecting:

.. cssclass:: table-striped table-bordered table-hover

==================================================== =================================
Select Import-Export Source Hypervisor               Action  
==================================================== =================================
VMware                                               Migrate existing instances to KVM
==================================================== =================================

|import-vm-from-vmware-to-kvm.png|

Selecting the Destination cluster
---------------------------------

CloudStack administrators must select a KVM cluster to import the VMware Virtual Machines (right side of the image above). Once a KVM cluster is selected, the VMware Datacenter selection part is displayed.

Selecting the VM from a VMware Datacenter
-----------------------------------------

CloudStack administrators must select the Source VMware Datacenter:

    - Existing: The existing zones are listed, and for each zone, CloudStack will list if there is any VMware Datacenter associated with it. In case it is, it can be selected.
    - External: CloudStack allows listing Virtual Machines from a VMware Datacenter that is not associated with any CloudStack zone. To do so, the vCenter IP address, the datacenter name, and username and password credentials are needed to log in to the vCenter. To import from a standalone VMware host, you can use the default datacenter name (ha-datacenter or other) along with the host credentials (Only stopped VMs are supported).

Once the VMware Datacenter is selected, click on List VMware Instances to display the list of Virtual Machines in the Datacenter. You must then choose the VMware Instance for import and click on Import Instance.

Converting and importing a VMware VM
------------------------------------

.. note:: CloudStack allows importing Running Linux Virtual Machines, but it is generally recommended that the Virtual Machine to import is powered off and has been gracefully shut down before the process starts. In case a Linux VM is imported while running, it will be converted in a "crash consistent" state. For Windows Virtual Machines, it is not possible to import them while running, they must be shut down gracefully so the filesystem is in a clean state.

.. note:: You can configure the parallel import of VM disk files on KVM host and management server, using the global settings: threads.on.kvm.host.to.import.vmware.vm.files and threads.on.ms.to.import.vmware.vm.files respectively.

In the UI import wizard, you can optionally select a KVM host and temporary destination storage (default is Secondary Storage, but if using Primary Storage - only NFS pools are supported) for the conversion, where VM files (OVF) will be copied to. This can be done by a random (or explicitly chosen) KVM host (if the ovftools are installed), otherwise, the management server will export/copy the VM files (optionally, you can force this action to be done by the management server even the KVM hosts have the ovftools installed in it). Irrelevant if the KVM host or the management server performs the copy of the VM files (OVF), you can further either let CloudStack choose which KVM host should do the conversion of the VM files using virt-v2v and which host will import the files to the destination Primary Storage Pool, or you can explicitly choose these KVM hosts for each of the 2 mentioned operations.

|import-vm-from-vmware-to-kvm-options.png|

When importing an instance from VMware to KVM, CloudStack performs the following actions:

    - Export the VM files (OVF) of the instance to a temporary storage location
      (which can be selected by the administrator). The export is performed by a
      KVM host if ovftool is installed or management server (can be forced by the 
      administrator, doesn't need ovftool installed on the management server).
      The existence of ovftool on KVM host is checked using 
      ``ovftool --version`` command.

      - If the instance on VMware is in **running** state, we clone the instance on
        VMware and use the new cloned instance to export OVF files.
        The cloning process may take some time to complete and is used to ensure data consistency,
        disk consolidation, etc.
      - If the instance on VMware is in **stopped** state, we directly use the
        instance to export its OVF files.
    - Converts the OVF on the temporary storage location to KVM using
      **virt-v2v**. CloudStack (or the administrator) selects a running and
      enabled KVM host to perform the conversion (of the previously exported OVF files) from VMware to KVM using
      **virt-v2v**. If the binary is not installed, then the host will fail to convert the Instance.
      In case it is installed, it will perform the conversion into
      the temporary location to store the converted QCOW2 disks of the instance.
      The virt-v2v conversion is a long-lasting process which can be set to
      time out by the global setting ``convert.vmware.instance.to.kvm.timeout``.
      The conversion process takes a long time because virt-v2v creates a
      temporary instance to inspect the source VM and generate the converted
      disks with the correct drivers. Additionally, it needs to copy the
      converted disks into the temporary location.
    - The converted instance (i.e. QCOW2 files) is then imported into the chosen KVM cluster.
      Administrator can choose the KVM host to perform the import or let CloudStack choose it. Only enabled 
      cluster and enabled hosts are considered.

.. note:: Please do not restart the management servers while migration is in progress as it will lead to the interruption of the process and you will need to start again.

.. note:: As mentioned above, the migration/conversion process uses an external tool, virt-v2v, which supports most but not all the operating systems out there (this is true for both the host on which the virt-v2v tool is running as well as the guest OS of the instances being migrated by the tool). Thus, the success of the import process will, almost exclusively, depend on the success of the virt-v2v conversion. In other words, the success will vary based on factors such as the current OS version, installed packages, guest OS setup, file systems, and others. Success is not guaranteed. We strongly recommend testing the migration process before proceeding with production deployments.

.. note:: The resulting imported VM uses the default Guest OS type: **CentOS 4.5 (32-bit)**. After importing the VM, please Edit the Instance to change the Guest OS Type accordingly.

VMware CBT migration mode
-------------------------

VMware Changed Block Tracking (CBT) migration mode provides a replication-style
VMware-to-KVM migration workflow. Instead of exporting the complete VM through
OVF and converting it in one operation, CloudStack creates an initial disk
replica on the destination primary storage, then copies only VMware CBT changed
blocks in later sync cycles. The source VMware VM can remain powered on during
the initial sync and normal delta sync cycles. The source VM must be powered off
before final cutover.

CBT migration mode is intended for large VMs where a single long conversion
window is undesirable. It does not provide live migration. The final cutover is
a controlled outage: the operator shuts down the source VMware VM, CloudStack
runs one final CBT delta sync, finalizes the replicated disks with ``virt-v2v``,
and imports the converted VM into the selected KVM cluster.

High-level CBT workflow
~~~~~~~~~~~~~~~~~~~~~~~

The CBT migration workflow has these stages:

#. The administrator selects VMware migration mode ``CBT`` in the
   *Tools > Import-Export Instances* VMware import wizard.
#. CloudStack validates the source VM, selected destination cluster, selected
   compute offering, target primary storage, and conversion host capability.
#. CloudStack creates a VMware snapshot and records the source disks and
   VMware CBT change IDs.
#. The KVM conversion host performs the initial full sync into a QCOW2 replica
   under the selected primary storage.
#. The administrator runs one or more delta sync cycles. Each cycle queries
   VMware CBT changed block ranges and copies only those ranges into the
   existing destination replica.
#. When the migration satisfies the configured quiet-cycle policy, or reaches
   the configured maximum number of cycles, CloudStack marks it ready for
   cutover.
#. The operator gracefully shuts down the source VMware VM.
#. CloudStack runs the final delta sync while the source is powered off.
#. CloudStack finalizes the replicated disks with ``virt-v2v``.
#. CloudStack moves the finalized QCOW2 disks to the selected primary storage
   root with generated volume UUID names and imports the VM into KVM.

CBT does not power off or gracefully shut down the source VMware VM. This is an
operator action. If cutover is attempted while the source VM is still powered
on, CloudStack rejects the request.

Supported source and destination
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

CBT migration mode is for VMware source VMs and KVM destination clusters.
CloudStack supports both source vCenter modes used by the VMware import wizard:

- Existing vCenter/datacenter registered with CloudStack.
- External vCenter/datacenter supplied in the import wizard.

For QCOW2 file targets, CloudStack stores the initial replica under the selected
primary storage:

::

   /mnt/<pool-uuid>/cloudstack-cbt/<migration-uuid>/<disk>.qcow2

After successful cutover finalization, the KVM agent moves each finalized disk
to the primary storage root and assigns a generated UUID filename:

::

   /mnt/<pool-uuid>/<generated-volume-uuid>

The relocated path is returned to the management server in the cutover result
and persisted in the CBT migration disk record. No manual database update is
required for normal operation.

KVM host requirements
~~~~~~~~~~~~~~~~~~~~~

The selected KVM conversion host must satisfy the same VMware import host
requirements as the OVF/VDDK migration modes documented above. In particular,
``virt-v2v`` is still required and is not installed by the CloudStack agent.
VMware VDDK access must also be configured for the host in the same way as for
VDDK-based VMware import.

CBT migration mode adds block-level replication and finalization requirements
on top of that baseline. At a minimum, the selected conversion host must have:

- ``virt-v2v`` available for final guest conversion;
- ``qemu-img`` for image inspection and conversion;
- ``qemu-nbd`` for attaching the replicated QCOW2 disks;
- ``qemu-io`` for changed-block writes;
- ``nbdkit`` with the VDDK plugin for VMware snapshot access;
- VMware VDDK libraries configured for the CloudStack agent;
- Windows VirtIO driver support when migrating Windows guests.

After changing conversion-host packages or VDDK configuration, restart the
CloudStack agent so the host capability details are refreshed.

The management server stores the host capability checks in host details. An
administrator can inspect them with a query similar to:

::

   SELECT h.id, h.name, hd.name, hd.value
   FROM cloud.host h
   JOIN cloud.host_details hd ON hd.host_id = h.id
   WHERE hd.name IN (
     'host.vddk.support',
     'host.vddk.version',
     'vddk.lib.dir',
     'host.qemu.img.version',
     'host.qemu.nbd.version',
     'host.qemu.io.version',
     'host.virtv2v.in.place.version',
     'host.vmware.cbt.support',
     'host.vmware.cbt.in.place.finalization.support'
   )
   ORDER BY h.name, hd.name;

A CBT-capable host reports ``host.vddk.support=true`` and
``host.vmware.cbt.support=true``. A host that can finalize the CBT replica
without a second full disk copy reports
``host.vmware.cbt.in.place.finalization.support=true``.

Windows guest requirement
~~~~~~~~~~~~~~~~~~~~~~~~~

Windows VMware guests have the same VirtIO Windows driver requirement as the
OVF/VDDK VMware import modes described above. The conversion host must provide
the Windows VirtIO driver files in the location expected by ``virt-v2v``. The
exact installation method is distribution-specific, so follow the same
``virtio-win`` guidance used for VDDK import rather than assuming a single
package command works on every KVM host OS.

If the Windows VirtIO driver files are missing, Windows conversion fails during
preflight or ``virt-v2v`` conversion with an error indicating that the VirtIO
Windows driver package is not available on the conversion host.

In-place finalization and fallback finalization
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

After the final CBT delta sync, CloudStack must run ``virt-v2v`` finalization.
The preferred path is in-place finalization, because it modifies the existing
replica instead of creating another full converted copy.

CloudStack detects in-place support on the KVM host in this order:

#. ``virt-v2v-in-place`` binary, if present in ``PATH``.
#. ``virt-v2v --in-place``, if the installed ``virt-v2v`` supports the option.

If neither in-place method is available, CloudStack can use regular
``virt-v2v -o local`` fallback finalization only when explicitly allowed by the
global setting:

::

   vmware.cbt.allow.non.inplace.finalization=true

The default is ``false``. With the default value, a KVM host that cannot perform
in-place finalization is rejected for CBT cutover. This avoids an unexpected
extra full-disk conversion copy.

When fallback finalization is enabled, CloudStack stages both ``TMPDIR`` and
``virt-v2v`` output under the migration directory on the selected primary
storage and validates free space before running the command. Required free
space is estimated conservatively as two times the summed disk capacity. After
successful fallback finalization, the fallback output is moved to the primary
storage root using the same final path layout as in-place finalization.

Global settings
~~~~~~~~~~~~~~~

The CBT migration policy is controlled by these global settings:

.. list-table::
   :header-rows: 1
   :widths: 40 15 45
   :class: table-striped table-bordered table-hover

   * - Name
     - Default
     - Description
   * - ``vmware.cbt.migration.min.cycles``
     - ``1``
     - Minimum completed delta cycles before quiet-cycle readiness can be
       evaluated.
   * - ``vmware.cbt.migration.max.cycles``
     - ``5``
     - Maximum delta cycles before the migration is considered ready for
       cutover.
   * - ``vmware.cbt.migration.quiet.cycles``
     - ``2``
     - Number of consecutive quiet cycles required for ready-for-cutover.
   * - ``vmware.cbt.migration.quiet.bytes``
     - ``1073741824``
     - Maximum changed bytes in a cycle for that cycle to be considered quiet.
   * - ``vmware.cbt.migration.quiet.dirty.rate``
     - ``16777216``
     - Maximum dirty rate in bytes per second for a quiet cycle.
   * - ``vmware.cbt.migration.agent.command.timeout``
     - ``86400``
     - Agent command timeout, in seconds, for long-running CBT sync and
       cutover operations.
   * - ``vmware.cbt.allow.non.inplace.finalization``
     - ``false``
     - Allows regular ``virt-v2v -o local`` fallback finalization when in-place
       finalization is unavailable.

The UI shows the cutover action only when the migration reaches a cutover-ready
state. This normally happens after the minimum number of cycles and the
configured quiet-cycle policy are satisfied, or after the maximum cycle count is
reached.

Compute offering validation
~~~~~~~~~~~~~~~~~~~~~~~~~~~

CloudStack validates the selected compute offering against the source VMware VM
sizing before starting a CBT migration. The selected offering must be able to
represent the source VM CPU count, CPU speed, and RAM. This is the same
principle used by the VMware OVF/VDDK import flow: the validation is intended
to fail early, before a long migration is started, but it should not be
stricter than the final VM import path.

Custom constrained offerings are valid when the source VM sizing is within the
offering's allowed range. Custom unconstrained offerings can be used by passing
custom CPU and RAM values. For powered-off source VMs where VMware reports CPU
speed as zero, CloudStack uses a default CPU speed value for validation and
custom offering prefill. The default CPU speed used for this case is
``2000 MHz``.

APIs
~~~~

CBT migration mode uses dedicated APIs:

- ``startVmwareCbtMigration`` starts a new CBT migration and returns an async
  job.
- ``listVmwareCbtMigrations`` returns migration state, current step, current
  step duration, disk target paths, disk state, completed cycles, quiet cycles,
  changed bytes, dirty rate, and last error details.
- ``syncVmwareCbtMigration`` starts an additional delta sync and returns an
  async job.
- ``cutoverVmwareCbtMigration`` performs the final sync, finalization, and VM
  import. It returns an async job.
- ``cancelVmwareCbtMigration`` cancels an active migration and runs cleanup.
- ``deleteVmwareCbtMigration`` deletes a terminal migration record. Cleanup of
  destination replica files is controlled by the API cleanup option.

The list API is the detailed progress and status source. The async job API
indicates whether a start, sync, or cutover request is still running or has
finished.

UI behaviour
~~~~~~~~~~~~

The CBT tab is shown under *Tools > Import-Export Instances* when the VMware
import workflow is active and the relevant APIs are available.

The CBT migrations table shows:

- migration state and current step;
- current step duration;
- conversion host;
- source vCenter, datacenter, host, and cluster;
- completed and quiet delta cycles;
- last changed bytes, last dirty rate, and total changed bytes;
- disk target paths and disk state;
- per-cycle changed bytes, dirty rate, duration, and description;
- last error when a migration fails.

The UI polls async jobs for start, sync, and cutover operations. It also
refreshes the CBT migration list while migrations are active, so operators can
see state transitions, step duration, and delta-cycle updates without pressing
the refresh button manually.

Source VM power state and cutover
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Initial sync and normal delta sync cycles can run while the VMware source VM is
powered on. Cutover requires the source VM to be powered off. CloudStack checks
the VMware power state immediately before cutover and rejects the operation if
the VM is still powered on.

CloudStack does not attempt to shut down the guest OS or power off the VMware
VM. This avoids surprising the operator and avoids encoding site-specific guest
shutdown policy in the migration path.

Credentials and cleanup
~~~~~~~~~~~~~~~~~~~~~~~

For external vCenter migrations, CloudStack stores the vCenter connection
details needed to continue sync and cutover operations. After successful
completion, CloudStack clears the stored per-migration credentials.

Cancel and delete operations are intended to remove only CBT migration state and
temporary migration artifacts. They do not delete the source VMware VM. A
successfully imported destination VM and its CloudStack volumes are not deleted
by deleting the completed CBT migration record.

Operational notes and limitations
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

- CBT depends on VMware CBT and VDDK behaviour. The initial sync may still read
  a large amount of data when VMware reports a large changed or allocated range,
  especially for some linked-clone or previously non-zeroed disk layouts.
- Delta cycles should normally be much smaller than the initial sync, because
  they use VMware CBT changed block ranges after the baseline change ID.
- The conversion step depends on ``virt-v2v`` guest support. Some guest
  operating systems, filesystems, boot layouts, or driver combinations may not
  be convertible.
- The target KVM root disk controller follows the controller selected by the
  ``virt-v2v`` output, including the SCSI controller used for CBT finalization.
- Do not restart management servers or the selected KVM conversion host while a
  migration operation is in progress.
- Agent logs contain the detailed ``qemu-img``, ``qemu-nbd``, ``qemu-io``,
  VDDK, and ``virt-v2v`` output. Management server logs and the CBT list API
  surface the operation-level error details.

.. |import-vm-from-vmware-to-kvm.png| image:: /_static/images/import-vm-from-vmware-to-kvm.png
   :alt: Import VMware Virtual Machines into KVM.
   :width: 800 px

.. |import-vm-from-vmware-to-kvm-options.png| image:: /_static/images/import-vm-from-vmware-to-kvm-options.png
   :alt: Import VMware Virtual Machines into KVM Options.
   :width: 800 px
