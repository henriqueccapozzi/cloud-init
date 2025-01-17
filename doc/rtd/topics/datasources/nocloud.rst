.. _datasource_nocloud:

NoCloud
=======

The data source ``NoCloud`` allows the user to provide user-data and meta-data
to the instance without running a network service (or even without having a
network at all).

You can provide meta-data and user-data to a local vm boot via files on a
`vfat`_ or `iso9660`_ filesystem. The filesystem volume label must be
``cidata`` or ``CIDATA``.

Alternatively, you can provide meta-data via kernel command line or SMBIOS
"serial number" option. The data must be passed in the form of a string:

::

  ds=nocloud[;key=val;key=val]

or

::

  ds=nocloud-net[;key=val;key=val]

The permitted keys are:

- ``h`` or ``local-hostname``
- ``i`` or ``instance-id``
- ``s`` or ``seedfrom``

With ``ds=nocloud``, the ``seedfrom`` value must start with ``/`` or
``file://``.  With ``ds=nocloud-net``, the ``seedfrom`` value must start
with ``http://`` or ``https://``.

e.g. you can pass this option to QEMU:

::

  -smbios type=1,serial=ds=nocloud-net;s=http://10.10.0.1:8000/

to cause NoCloud to fetch the full meta-data from http://10.10.0.1:8000/meta-data
after the network initialization is complete.

These user-data and meta-data files are expected to be in the following format.

::

  /user-data
  /meta-data

Both files are required to be present for it to be considered a valid seed ISO.

Basically, user-data is simply user-data and meta-data is a YAML formatted file
representing what you'd find in the EC2 metadata service.

You may also optionally provide a vendor-data file in the following format.

::

  /vendor-data

Given a disk ubuntu cloud image in 'disk.img', you can create a
sufficient disk by following the example below.

::

    ## 1) create user-data and meta-data files that will be used
    ## to modify image on first boot
    $ echo "instance-id: iid-local01\nlocal-hostname: cloudimg" > meta-data
    $ echo "#cloud-config\npassword: passw0rd\nchpasswd: { expire: False }\nssh_pwauth: True\n" > user-data

    ## 2a) create a disk to attach with some user-data and meta-data
    $ genisoimage  -output seed.iso -volid cidata -joliet -rock user-data meta-data

    ## 2b) alternatively, create a vfat filesystem with same files
    ## $ truncate --size 2M seed.iso
    ## $ mkfs.vfat -n cidata seed.iso

    ## 2b) option 1: mount and copy files
    ## $ sudo mount -t vfat seed.iso /mnt
    ## $ sudo cp user-data meta-data /mnt
    ## $ sudo umount /mnt

    ## 2b) option 2: the mtools package provides mcopy, which can access vfat
    ## filesystems without mounting them
    ## $ mcopy -oi seed.iso user-data meta-data

    ## 3) create a new qcow image to boot, backed by your original image
    $ qemu-img create -f qcow2 -b disk.img -F qcow2 boot-disk.img

    ## 4) boot the image and login as 'ubuntu' with password 'passw0rd'
    ## note, passw0rd was set as password through the user-data above,
    ## there is no password set on these images.
    $ kvm -m 256 \
       -net nic -net user,hostfwd=tcp::2222-:22 \
       -drive file=boot-disk.img,if=virtio \
       -drive driver=raw,file=seed.iso,if=virtio

**Note:** that the instance-id provided (``iid-local01`` above) is what is used
to determine if this is "first boot".  So if you are making updates to
user-data you will also have to change that, or start the disk fresh.

Also, you can inject an ``/etc/network/interfaces`` file by providing the
content for that file in the ``network-interfaces`` field of metadata.

Example metadata:

::

    instance-id: iid-abcdefg
    network-interfaces: |
      iface eth0 inet static
      address 192.168.1.10
      network 192.168.1.0
      netmask 255.255.255.0
      broadcast 192.168.1.255
      gateway 192.168.1.254
    hostname: myhost


Network configuration can also be provided to cloud-init in either
:ref:`network_config_v1` or :ref:`network_config_v2` by providing that
YAML formatted data in a file named ``network-config``.  If found,
this file will override a ``network-interfaces`` file.

See an example below.  Note specifically that this file does not
have a top level ``network`` key as it is already assumed to
be network configuration based on the filename.

.. code:: yaml

  version: 1
  config:
     - type: physical
       name: interface0
       mac_address: "52:54:00:12:34:00"
       subnets:
          - type: static
            address: 192.168.1.10
            netmask: 255.255.255.0
            gateway: 192.168.1.254


.. code:: yaml

  version: 2
  ethernets:
    interface0:
      match:
        mac_address: "52:54:00:12:34:00"
      set-name: interface0
      addresses:
        - 192.168.1.10/255.255.255.0
      gateway4: 192.168.1.254


.. _iso9660: https://en.wikipedia.org/wiki/ISO_9660
.. _vfat: https://en.wikipedia.org/wiki/File_Allocation_Table
.. vi: textwidth=79
