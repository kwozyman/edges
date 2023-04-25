Edges - Edge Seed
===

Quick setup
---

```
git clone https://github.com/kwozyman/edges.git
git clone --branch fdo-collection https://github.com/empovit/fido-device-onboard-demo.git
ansible-playbook -i inventory.yml fido-device-onboard-demo/fdo-servers.yml -e @vars.yml
ansible-playbook -i inventory.yml setup-builder.yml -e @vars.yml
ansible-playbook -i inventory.yml setup-bootinfra.yml -e @vars.yml
```

Roles
---

In order to deploy Edge Seed, you'll need to edit the global variables. In the examples above, these are contained in the `vars.yml` file:

```
---
bootinfra_ip: 192.168.122.1
manufacturing_server_rendezvous_info_ip_address: "{{ bootinfra_ip }}"
owner_onboarding_server_owner_addresses_ip_address: "{{ bootinfra_ip }}"
master_ssh_key: "<< sshkey >>"
```

These values are propagated to all hosts.

In order to deploy Edge Seed, you need to edit your inventory and potentially edit the Ansible variables. An example is provided in `inventory-example.yml`.

### libvirt

To setup the virtual machines a host named `hypervisor` is required, with the following configuration:

  * `libvirt_setup_enable` - install and configure libvirt on the hypervisor
  * `libvirt_pool_path` - path on the hypervisor to store virtual machine images
  * `libvirt_networks` - a list of virtual networks to be created:
    * `name` - network name
    * `cidr` - subnet
    * `forward_mode` - set to the type of libvirt forward mode wanted. Usually this means `nat` for network address translation (preferrable in all in one or other types of demos) or `bridge` for bridging to an existing physical device on the hypervisor
    * `bridge_name` - only used to name the interface to bridge to in case of `forward_mode=bridge`
    * `ip` - set in order to also setup ip on hypervisor (required for `forward_mode=nat`)
  * `libvirt_isos` - a list of iso images to be made available to the hypervisor
    * `name` - iso name
    * `url` - url for download
  * `libvirt_vms` - what virtual machines to create
    * `name` - vm name
    * `iso` - what iso image to use -- see `libvirt_isos`
    * `memory` - memory size in MB
    * `cpu` - number of CPUs
    * `disk` - disk size in GB
    * `networks` - what network interfaces to configure
      * `name` - libvirt network name -- see `libvirt_networks`
      * `ip` - network ip
    * `passwd` - root password
    * `sshkey` - root ssh key

### community.fdo

See [the playbooks](https://github.com/empovit/fido-device-onboard-demo/tree/fdo-collection)

### builder

To setup the builder machine, a host names `image-builder` is required, with the following possible configuration:

  * `admin_password`: unencrypted password to add to generated images (optional)
  * `sshkey`: public ssh key to be added to generated images
  * `builder_blueprints`:
    * `name`: blueprint/image name
    * `description`: text with image description
    * `sshkey`: custom ssh key per blueprint
    * `fdo`: true if this is an FDO image
    * `installation_device`: block device to deploy image to
    * `manufacturing_server`: for FDO images, the manufacturing server ip
    * `manufacturing_port`: for non-standard FDO manufacturing server port
    * `image_type`: usually `edge-simplified-installer` for FDO
    * `ref`: ostree reference. For RHEL `rhel/9/x86_64/edge`
    * `packages`: list with additional packages to add to image

### bootinfra

To setup boot infrastructure, a host named `bootinfra` is required, with the following configuration:

  * `eci_configdir` - configuration files path, defaults to `/etc/eci`
  * `bootinfra_ip` - on what ip to listen for DHCP
  * `tftpboot_enabled` - enable tftpboot, defaults to `true`
  * `tftpboot_download_enable` - download iso images, defaults to `true`
  * `tftpboot_iso` - a list of bootable iso images
    * `name` - iso name
    * `url` - url for download
    * `files` - a list of files to extract from the iso
    * `shim` - bootable shim file name
    * `kernel` - kernel file to boot
    * `kargs` - kernel arguments
    * `initrd` - a list of initrd files
    * `rootfs` - root filesystem file
    * `rootfs_dir` - relative directory to hard link rootfs to -- this is a workaround required for RHEL images
  * `dhcp_enabled` - enable DHCP, defaults to `true`
  * `dhcp_hosts` - list of DHCP hosts
    * `name` - host name
    * `mac` - MAC address
    * `boot` - what is image to boot -- see `tftpboot_iso`
  * `dhcp_range` - DHCP ip range in dnsmasq format
  * `dns_enabled` - enable DNS, defaults to `true`
  * `dns_static` - list of static DNS entries
    * `name` - hostname
    * `ip` - ip

Example
---

The following configuration will create a virtual machine `bootinfra` running RHEL 9.1 and two network interfaces, one `192.168.101.2` in NAT libvirt network `eci-mgmt` and one `192.168.6.1` in isolated libvirt network `eci`. The OS is automatically deployed with the specified ssh key.

```
    hypervisor:
      ansible_host: hypervisor
      bridges:
        - name: eci
          ip: 192.168.5.1/24
          interface: enp7s0
      libvirt_setup_enable: true
      libvirt_pool_path: /var/eci/vm
      libvirt_networks:
        - name: eci-mgmt
          cidr: 192.168.101.0/24
          forward_mode: nat
          ip: 192.168.101.1
        - name: eci
          cidr: 192.168.6.0/24
      libvirt_isos:
        - name: rhel91
          #url: https://developers.redhat.com/content-gateway/file/rhel/9.1/rhel-baseos-9.1-x86_64-dvd.iso
          url: https://download.rockylinux.org/pub/rocky/9/isos/x86_64/Rocky-9.1-x86_64-dvd.iso
      libvirt_vms:
        - name: bootinfra
          iso: rhel91
          memory: 2048
          cpu: 2
          disk: 50
          networks:
            - name: eci-mgmt
              ip: 192.168.101.2
            - name: eci
              ip: 192.168.6.1
          sshkey: <<sshkey>>
```

After the previous virtual machine completes setup and first boot, we can configure it to automatically boot other hosts. The following configuration will setup boot infrastructure -- dhcp, dns, tftp, http on the interface with ip `192.168.6.1` -- note this is the ip configured for network `eci` in the previous step. In the iso section, we describe what files to extract and what kernel to boot. The root filesystem is made available via a web server and we can instruct the kernel to use it via `inst.stage2` argument. Finally, we add two hosts: `dhcp-test` and `testinfra` with their respective configuration.

```
   bootinfra:
      ansible_host: bootinfra
      ansible_user: root
      eci_configdir: /etc/eci
      bootinfra_ip: 192.168.6.1
      tftpboot_enabled: true
      tftpboot_download_enable: true
      tftpboot_iso: 
        - name: rhel-install
          url: https://download.rockylinux.org/pub/rocky/9/isos/x86_64/Rocky-9.1-x86_64-dvd.iso
          files:
            - images/pxeboot/vmlinuz
            - images/pxeboot/initrd.img
            - images/install.img
            - EFI/BOOT/BOOTX64.EFI
            - EFI/BOOT/grubx64.efi
          shim: BOOTX64.EFI
          kernel: vmlinuz
          kargs: "inst.stage2=http://{{ bootinfra_ip }}/rhel-install rd.live.check"
          initrd:
            - initrd.img
          rootfs: install.img
          rootfs_dir: images/
      dhcp_enabled: true
      dhcp_hosts:
        - name: dhcp-test
          mac: 52:54:00:29:24:96
          ip: 192.168.6.50
          boot: rhel-install
        - name: testinfra
          mac: 52:54:00:70:30:b0
          ip: 192.168.6.51
          boot: rhel-install
      dhcp_range: "{{ bootinfra_ip }},static"
      dns_enabled: true
      dns_static:
        - name: eci.rh-intel.com
          ip: 192.168.1.18
        - name: apps.eci.rh-intel.com
          ip: 192.168.1.18
```

After the configuration is complete, if we PXE boot two **UEFI** hosts with the correctly configured mac addresses on the same network, they should boot into RHEL setup.


