---
description: The configuration of standalone OVN on FCOS
---

# OVN + FCOS

## Installation

```bash
sudo rpm-ostree install --apply-live ovn ovn-host ovn-central openvswitch
```

### Configure VM

Add the below to the `virt-install` command.

```bash
--network bridge=br-int,virtualport_type=openvswitch
```

### Configure interface in OVN

```bash
IFACE=$(sudo virsh -q domiflist <VM_NAME> | awk '{print $1}')
IFACE_ID=$(sudo ovs-vsctl get interface {IFACE} external_ids:iface-id | sed s/\"//g)
sudo ovn-nbctl list Logical_Switch
sudo ovn-nbctl lsp-add <SWITCH_NAME> ${IFACE_ID}
MAC=$(sudo ovs-vsctl get interface vnet0 external_ids:attached-mac | sed s/\"//g)
sudo ovn-nbctl lsp-set-addresses ${IFACE_ID} ${MAC}
```

### Configure DHCP in the logical switch

```bash
sudo ovn-nbctl dhcp-options-create 192.168.130.0/24
DHCP_CIDR_UUID=$(sudo ovn-nbctl --bare --columns=_uuid find dhcp_options cidr="192.168.130.0/24")
sudo ovn-nbctl dhcp-options-set-options ${DHCP_CIDR_UUID} lease_time=3600 router=192.168.130.1 server_id=192.168.130.1 server_mac=52:54:00:00:00:01
IFACE=$(sudo virsh -q domiflist <VM_NAME> | awk '{print $1}')
IFACE_ID=$(sudo ovs-vsctl get interface {IFACE} external_ids:iface-id | sed s/\"//g)
sudo ovn-nbctl lsp-set-dhcpv4-options ${IFACE_ID} ${DHCP_CIDR_UUID}
```

### Configure management port

```bash
sudo ovs-vsctl add-port br-int onp-mp0 -- set interface onp-mp0 type=internal
sudo ovs-vsctl set Interface onp-mp0 external_ids:iface-id=onp-mp0
sudo ovn-nbctl set logical_switch_port onp-mp0 up=true
MAC=$(ip a s onp-mp0 | grep ether | awk '{print $2}')
IP=$(ip a s onp-mp0 | grep inet | awk '{print $2}' | cut -d/ -f1)
sudo ovn-nbctl lsp-set-addresses onp-mp0 "${MAC} ${IP}"
```
