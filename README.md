Red Hat - Intel ECI lab
===

Infra setup
---

In order to run virtual machines bridged to physical networks, we need to setup bridges on the host server. There is an Ansible role `net-bridge` which does this via NetworkManager, the variables used are:

```
bridges:
  - name: eci
    ip: 192.168.1.18/24
    interface: ens786f1
```

To run:

```
ansible-playbook -i infra.yml infra-bridges.yml
```

Libvirt setup is also required:

```
libvirt_pool_name: eci
libvirt_pool_path: /var/lib/libvirt/eci-images
```

To run:

```
ansible-playbook -i infra.yml infra-libvirt.yml
```