#  Basic Configuration

The following provides instructions on how to configure an installed POD.

##Fabric

This section describes how to apply a basic configuration to a freshly installed fabric. The fabric needs to be configured to forward traffic between the different components of the POD. More info about how to configure the fabric can be found here.

##Configure Routes on the Compute Nodes

Each leaf switch on the fabric corresponds to a separate IP subnet.
Routes must be manually configured on the compute nodes so that traffic between nodes on different leaves will be forwarded via the local spine switch.

Run commands of this form on each compute node:

```
sudo ip route add <remote-leaf-subnet> via <local-spine-ip>
```

The recommended configuration is a POD with two leaves; the leaf1 subnet is `10.6.1.0/24` and the leaf2 subnet is `10.6.2.0/24`.  In this configuration, on the compute nodes attached to leaf1, run:

```
sudo ip route add 10.6.2.0/24 via 10.6.1.254
```

Likewise, on the compute nodes attached to leaf2, run:

```
sudo ip route add 10.6.1.0/24 via 10.6.2.254
```

>NOTE: it’s strongly suggested to add it as a permanent route to the compute node, so the route will still be there after a reboot

##Configure the Fabric:  Overview

On the head node there is a service able to generate an ONOS network configuration to control the leaf and spine network fabric. This configuration is generated querying ONOS for the known switches and compute nodes and producing a JSON structure that can be posted to ONOS to implement the fabric.

The configuration generator can be invoked using the CORD generate command, which print the configuration at screen (standard output).

##Remove Stale ONOS Data

Before generating a configuration you need to make sure that the instance of ONOS controlling the fabric doesn't contain any stale data and that has processed a packet from each of the switches and computes nodes.

ONOS needs to process a packet because it does not have a mechanism to automatically discover the network elements. Thus, to be aware of a device on the network ONOS needs to first receive a packet from it.

To remove stale data from ONOS, the ONOS CLI `wipe-out` command can be used:

```
ssh -p 8101 onos@onos-fabric wipe-out -r -j please
Warning: Permanently added '[onos-fabric]:8101,[10.6.0.1]:8101' (RSA) to the list of known hosts.
Password authentication
Password:  (password rocks)
Wiping intents
Wiping hosts
Wiping Flows
Wiping groups
Wiping devices
Wiping links
Wiping UI layouts
Wiping regions
```

>NOTE: When prompt, use password "rocks".

To ensure ONOS is aware of all the switches and the compute nodes, you must have each switch "connected" to the controller and let each compute node ping over its fabric interface to the controller.

##Connect the Fabric Switches to ONOS

If the switches are not already connected, the following command on the head node CLI will initiate a connection.

```
for s in $(cord switch list | grep -v IP | awk '{print $3}'); do
ssh -i ~/.ssh/cord_rsa -qftn root@$s ./connect -bg 2>&1  > $s.log
done
```

You can verify ONOS has recognized the devices using the following command:

```
ssh -p 8101 onos@onos-fabric devices

Warning: Permanently added '[onos-fabric]:8101,[10.6.0.1]:8101' (RSA) to the list of known hosts.
Password authentication
Password:
id=of:0000cc37ab7cb74c, available=true, role=MASTER, type=SWITCH, mfr=Broadcom Corp., hw=OF-DPA 2.0, sw=OF-DPA 2.0, serial=, driver=ofdpa, channelId=10.6.0.23:58739, managementAddress=10.6.0.23, protocol=OF_13
id=of:0000cc37ab7cba58, available=true, role=MASTER, type=SWITCH, mfr=Broadcom Corp., hw=OF-DPA 2.0, sw=OF-DPA 2.0, serial=, driver=ofdpa, channelId=10.6.0.20:33326, managementAddress=10.6.0.20, protocol=OF_13
id=of:0000cc37ab7cbde6, available=true, role=MASTER, type=SWITCH, mfr=Broadcom Corp., hw=OF-DPA 2.0, sw=OF-DPA 2.0, serial=, driver=ofdpa, channelId=10.6.0.52:37009, managementAddress=10.6.0.52, protocol=OF_13
id=of:0000cc37ab7cbf6c, available=true, role=MASTER, type=SWITCH, mfr=Broadcom Corp., hw=OF-DPA 2.0, sw=OF-DPA 2.0, serial=, driver=ofdpa, channelId=10.6.0.22:44136, managementAddress=10.6.0.22, protocol=OF_13
```

>NOTE: This is a sample output that won’t necessarily reflect your output

>NOTE: When prompt, use password "rocks".

##Connect Compute Nodes to ONOS

To make sure that ONOS is aware of the compute nodes, the following commands will send a ping over the fabric interface on the head node and each compute node.

```
ping -c 1 10.6.1.254
for h in $(cord prov list | grep "^node" | awk '{print $2}'); do
ssh -i ~/.ssh/cord_rsa -qftn ubuntu@$h ping -c 1 10.6.1.254;
done
```

It is fine if the `ping` command fails; the purpose is to register the node with ONOS.  You can verify ONOS has recognized the nodes using the following command:

```
ssh -p 8101 onos@onos-fabric hosts
Warning: Permanently added '[onos-fabric]:8101,[10.6.0.1]:8101' (RSA) to the list of known hosts.
Password authentication
Password:
id=00:16:3E:DF:89:0E/None, mac=00:16:3E:DF:89:0E, location=of:0000cc37ab7cba58/3, vlan=None, ip(s)=[10.6.1.1], configured=false
id=3C:FD:FE:9E:94:28/None, mac=3C:FD:FE:9E:94:28, location=of:0000cc37ab7cba58/4, vlan=None, ip(s)=[10.6.1.2], configured=false
```

>NOTE: When prompt, use password rocks

##Generate the Network Configuration

To modify the fabric configuration for your environment, generate on the head node a new network configuration using the following commands:

```
cd /opt/cord_profile && \
cp fabric-network-cfg.json{,.$(date +%Y%m%d-%H%M%S)} && \
cord generate > fabric-network-cfg.json
```

##Load Network Configuration

Once these steps are done load the new configuration into XOS, and restart the apps in ONOS:

###Install Dependencies

```
sudo pip install httpie
```

###Delete Old Configuration

```
http -a onos:rocks DELETE http://onos-fabric:8181/onos/v1/network/configuration/
```

###Load New Configuration

```
cd /opt/cord_profile && \
docker-compose -p rcord exec xos_ui python /opt/xos/tosca/run.py xosadmin@opencord.org /opt/cord_profile/fabric-service.yaml
```

###Restart ONOS Apps

```
http -a onos:rocks POST http://onos-fabric:8181/onos/v1/applications/org.onosproject.vrouter/active
http -a onos:rocks POST http://onos-fabric:8181/onos/v1/applications/org.onosproject.segmentrouting/active
```

To verify that XOS has pushed the configuration to ONOS, log into ONOS in the onos-fabric VM and run netcfg:

```
$ ssh -p 8101 onos@onos-fabric netcfg
Password authentication
Password:
{
    "devices": {
        "of:0000480fcfaeee2a": {
            "segmentrouting": {
                "name": "device-480fcfaeee2a",
                "ipv4NodeSid": 100,
                "ipv4Loopback": "10.6.0.103",
                "routerMac": "48:0f:cf:ae:ee:2a",
                "isEdgeRouter": true,
                "adjacencySids": []
            }
        },
        "of:0000480fcfaede26": {
            "segmentrouting": {
                "name": "device-480fcfaede26",
                "ipv4NodeSid": 101,
                "ipv4Loopback": "10.6.0.101",
                "routerMac": "48:0f:cf:ae:de:26",
                "isEdgeRouter": true,
                "adjacencySids": []
            }
        },
	...
```

>NOTE: When prompt, use password "rocks"

## Verify Connectivity over the Fabric

Once the new ONOS configuration is active, the fabric interface on each node should be reachable from the other nodes.  From each compute node, ping the IP address of head node's fabric interface (e.g., `10.6.1.1`).
