# Instanciate cloud image VM on KVM using cloud-init

This guide is about how to instanciate a cloud-image like CentOS-7-x86_64-GenericCloud.qcow2 on a KVM host without orchestration like Openstack. 

## Create a thin provising hard drive for your VM

We can create an hard drive which is backed by the original qcow2 image, just the same
way Openstack does in the back scene. 

The following command will create a new file named my_new_vm_drive.qcow2 that will
only contains the diff from the original CentOS qcow2. This drive will be presented
as a 100G drive to the operating system, but will not use it on the host until data is wrtten. 

```
qemu-img create -f qcow2 -b /var/lib/libvirt/images/CentOS-7-x86_64-GenericCloud.qcow2 /var/lib/libvirt/images/my_new_vm_drive.qcow2 100G
````

## Create a config drive for cloud-init

All cloud images runs cloud-init at the boot sequence to retreive useful informations 
such as hostname, ip address, user definitions, packages installation and so on. On Openstack, cloud-init access those informations using metadata-agent that respond on a 
HTTP call to http://169.254.169.254/openstack/...

When using straight KVM, there is no such agent. We have to provide the informations on what we call, config-drive. This drive is an ISO file that contains the informations and that is mounted on a emulated CDROM to the VM. 

The minimal ISO require 3 files for config-drive. 
- meta-data
- network-config
- user-data

### meta-data file

The meta-data file contains, at least, the hostname of the virtual machine. Here is an example of a minimal meta-data file

```
instance-id: 123345
local-hostname: my_new_vm_hostname
```

### network-config file

The network-config file may vary according to the needed network configuration you need. 
This file may contains bridge informations, static IP or other. 

The following example is the most common where the main network interface uses DHCP to
retreive it's configuration. 

Note that the following format uses Network Config Version 2

See: 
https://cloudinit.readthedocs.io/en/latest/topics/network-config-format-v2.html

```
network:
  version: 2
  ethernets:
    eno1:
      dhcp4: true
```

### user-data file

The user-data file contains the script to configure the virtual machine in regards to 
users, ssh keys, packages, and any other configuration that needs to be automated. 

The following is a very basic example of a user-data that will create a user named
"myname" with the password "strongpassword". 

```
#cloud-config
ssh_pwauth: True
users:
  - default
  - name: myname
    groups: wheel
    sudo: ["ALL=(ALL) NOPASSWD:ALL"]
chpasswd:
  list: |
    root:strongpassword
    myname:strongpassword
  expire: False
```

### Package the 3 files into a ISO image

Once all 3 files are created, we need to package the files into a ISO file that will be 
called "config-drive". This drive will be added to the libvirt XML definition of the new VM.

```
genisoimage  -output /var/lib/libvirt/images/my_new_vm_config.iso -volid cidata -joliet -rock user-data network-config meta-data
```

## Create the libvirt VM definition XML

libvirt is used to manage QEMU virtual machines, of course accelerated by KVM kernel module. 

Here is a basic example of a VM definition

Note that the following example assume that there is an Linux bridge named br0 created
on the host. 

To quick start, create a file named my_new_vm.xml
```
<domain type='kvm'>
    <name>my_new_vm</name>
    <memory unit='G'>2</memory>
    <vcpu placement='static'>2</vcpu>
    <os>
        <type arch='x86_64'>hvm</type>
        <boot dev='cdrom'/>
        <boot dev='hd'/>
    </os>
    <features>
        <acpi/>
        <apic/>
        <pae/>
    </features>
    <clock offset='utc'/>
    <on_poweroff>destroy</on_poweroff>
    <on_reboot>restart</on_reboot>
    <on_crash>restart</on_crash>
    <devices>
        <emulator>/usr/bin/kvm</emulator>
        <disk type='file' device='disk'>
            <driver name='qemu' type='qcow2' cache='none' io='threads'/>
            <source file='/var/lib/libvirt/images/my_new_vm_drive.qcow2'/>
            <target dev='vda' bus='virtio'/>
        </disk>
        <disk type='file' device='cdrom'>
            <driver name='qemu' type='raw'/>
            <source file='/var/lib/libvirt/images/my_new_vm_config.iso'/>
            <target dev='hdc' bus='ide'/>
            <readonly/>
        </disk>
        <interface type='bridge'>
            <source bridge='br0'/>
            <target dev='mynewvm'/>
            <model type='virtio'/>
        </interface>
        <serial type='pty'>
            <target port='0'/>
        </serial>
        <console type='pty'>
            <target type='serial' port='0'/>
        </console>
        <graphics type='vnc' port='-1' autoport='yes' listen='0.0.0.0'>
            <listen type='address' address='0.0.0.0'/>
        </graphics>
        <video>
            <model type='vmvga' vram='16384' heads='1'/>
        </video>
        <memballoon model='virtio'>
        </memballoon>
    </devices>
</domain>
```

## Register the new VM on libvirt

```
virsh define my_new_vm.xml
```

## Check that the new VM is well defined with the right parameters

The new VM should be present in a shut-down state.
```
virsh list --all
```

Check that parameters are correctly set (CPU, RAM, etc)
```
virsh dominfo my_new_vm
```

## Start the VM

```
virsh start my_new_vm
```

## Access the console

```
virsh console my_new_vm
```


