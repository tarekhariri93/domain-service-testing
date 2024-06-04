The following user guide outlines the steps to utilize the API in order to establish East-West Connectivity between Data Processing Units (DPUs) and tenant isolation

==============================
East West Traffic Isolation
==============================

**Create interfaces and interface config for each VM**

    ::
        MAC_H1PF0VF0=[VM1 vf0 mac address on first x86 host]

        MAC_H1PF0VF1=[VM2 vf1 mac address on first x86 host]

        MAC_H2PF0VF0=[VM3 vf0 mac address on second x86 host]

        MAC_H2PF0VF1=[VM4 vf1 mac address on second x86 host]

 - Interface on DPU-A::

        #pf0vf0
        grpcurl -plaintext -d '{"name":"h1pf0vf0", "dpu":"DPU-A", "mac_address":"${MAC_H1PF0VF0}", "pf_id":"0"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterface

        # Note the name here, this is the name we gave in the external_ids for the port in OVS
        # interface that is attached to VM1
        grpcurl -plaintext -d '{"name":"bf1pf0vf0", "interface":"h1pf0vf0", "dpu":"DPU-A", "segment":"ns1", "address":"192.168.0.2"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterfaceConfig

        #pf0vf1
        grpcurl -plaintext -d '{"name":"h1pf0vf1", "dpu":"DPU-A", "mac_address":"${MAC_H1PF0VF1}", "pf_id":"0"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterface

        # Note the segment is ns3 as it will be used by VM2
        grpcurl -plaintext -d '{"name":"bf1pf0vf1", "interface":"h1pf0vf1", "dpu":"DPU-A", "segment":"ns3", "address":"10.152.0.2"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterfaceConfig

Prerequisites
---------------------

Deployment
    Ensure you have deployed the system comprising the control plane, DPU and compute nodes. For the deployment procedure, refer to the OVN 
    `Isolation Deployment guide <https://gitlab-master.nvidia.com/sdn/ovn-isolation-deployment>`_.

Setup grpc::

    wget https://github.com/fullstorydev/grpcurl/releases/download/v1.8.9/grpcurl_1.8.9_linux_x86_64.tar.gz
    tar -xvzf grpcurl_1.8.9_linux_x86_64.tar.gz
    mv grpcurl /usr/local/bin/
    or
    go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest


From the Control-Plane node, check you can reach the service and list endpoints::

    OVN_DOMAIN_SVC=localhost

    grpcurl -plaintext ${OVN_DOMAIN_SVC}:5067 list ovnisolation.OVNIsolation


Domain Isolation - East West Connectivity
------------------------------------------

Use case
---------

describing a use case where an administrator intends to establish a network for 4 VMs, VM1-VM4, two domains, each domain with 2 virtual machines.

First domain contains VM1 and VM3 each under different network segments.

Second domain contains VM2 and VM4 under the same network segment.

VM1 and VM2 are hosted on DPU-A.

VM3 and VM4 are hosted on DPU-B.

The goal is to enable network connectivity and data transfer between VM1 and VM3, VM2 and VM4 using the appropriate API calls.

## Partial API diagram used

![image](/uploads/420ca085d00d695089429444a55eeca9/image.png)

Solution
---------

assuming all the VMs/network namespaces existing on the x86 host and thier corresponding interface representors connected to OVS bridge.
on the deployment we should have 2 vfs on each x86 host.
on each DPU we should have two interface reperesontors connected to ovs bridge br-int.

Create two domains: `domain-red` and `domain-blue`
---------------------------------------------------
`domain-red` will contain VM1 and VM3
`domain-blue` will contain VM2 and VM4

![image](/uploads/c3811a6674021c8ff3c057bf96303ffc/image.png)

::
    grpcurl -plaintext -d '{"name":"domain-red"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateDomain
    grpcurl -plaintext -d '{"name":"domain-blue"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateDomain

For domain `domain-red` create two network segments
------------------------------------------------------
specify a unique name for the network segment, subnet of the network segment,
the domain it is attached to and routed to true to enable connectivity between the
network segments under the same domain::

    grpcurl -plaintext -d '{"name":"ns1", "domain": "domain-red", "subnet": "192.168.0.0/24", "routed":true}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateNetworkSegment
    grpcurl -plaintext -d '{"name":"ns2", "domain": "domain-red", "subnet": "192.168.1.0/24", "routed":true}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateNetworkSegment

For domain `domain-blue` create a single network segment
---------------------------------------------------------
specify a unique name for the network segment, subnet of the network segment,
the domain it is attached to, default value of routed is false as we don't have any other network segments
under `domain-blue`::

    grpcurl -plaintext -d '{"name":"ns3", "domain": "domain-blue", "subnet": "10.152.0.0/16"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateNetworkSegment

create two DPUs: `DPU-A` and `DPU-B`::

    grpcurl -plaintext -d '{"name":"DPU-A"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateDPU
    grpcurl -plaintext -d '{"name":"DPU-B"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateDPU

Create interfaces and interface config for each VM
------------------------------------------------------

MAC_H1PF0VF0=[VM1 vf0 mac address on first x86 host]

MAC_H1PF0VF1=[VM2 vf1 mac address on first x86 host]

MAC_H2PF0VF0=[VM3 vf0 mac address on second x86 host]

MAC_H2PF0VF1=[VM4 vf1 mac address on second x86 host]

 - Interface on DPU-A::

        #pf0vf0
        grpcurl -plaintext -d '{"name":"h1pf0vf0", "dpu":"DPU-A", "mac_address":"${MAC_H1PF0VF0}", "pf_id":"0"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterface

        # Note the name here, this is the name we gave in the external_ids for the port in OVS
        # interface that is attached to VM1
        grpcurl -plaintext -d '{"name":"bf1pf0vf0", "interface":"h1pf0vf0", "dpu":"DPU-A", "segment":"ns1", "address":"192.168.0.2"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterfaceConfig

        #pf0vf1
        grpcurl -plaintext -d '{"name":"h1pf0vf1", "dpu":"DPU-A", "mac_address":"${MAC_H1PF0VF1}", "pf_id":"0"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterface

        # Note the segment is ns3 as it will be used by VM2
        grpcurl -plaintext -d '{"name":"bf1pf0vf1", "interface":"h1pf0vf1", "dpu":"DPU-A", "segment":"ns3", "address":"10.152.0.2"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterfaceConfig

 - Interface on DPU-B::

    #pf0vf0
    grpcurl -plaintext -d '{"name":"h2pf0vf0", "dpu":"DPU-B", "mac_address":"${MAC_H2PF0VF0}", "pf_id":"0"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterface

    # interface that will be attached to VM3
    grpcurl -plaintext -d '{"name":"bf2pf0vf0", "interface":"h2pf0vf0", "dpu":"DPU-B", "segment":"ns2", "address":"192.168.1.2"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterfaceConfig

    #pf0vf1
    grpcurl -plaintext -d '{"name":"h2pf0vf1", "dpu":"DPU-B", "mac_address":"${MAC_H2PF0VF1}", "pf_id":"0"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterface

    # Note the segment is ns3 as it will be used by VM4
    grpcurl -plaintext -d '{"name":"bf2pf0vf1", "interface":"h2pf0vf1", "dpu":"DPU-B", "segment":"ns3", "address":"10.152.0.3"}' ${OVN_DOMAIN_SVC}:5067  ovnisolation.OVNIsolation/CreateInterfaceConfig

simulation VMs
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

To simulate the VMs we can create network namespaces on hosts, you should be able to use the VFs inside the network namespaces and talk to other interfaces under the same domain.

on first x86 host create two network namespaces to represent VM1 and VM2::

    VM=vm1
    GW=192.168.0.1
    VF=pf0vf0
    IP=192.168.0.2/24
    ip netns add $VM
    ip link set $VF netns $VM
    ip netns exec $VM ip addr add $IP dev $VF
    ip netns exec $VM ip link set $VF up
    ip netns exec $VM ip route add default via $GW

    VM=vm2
    GW=10.152.0.1
    VF=pf0vf1
    IP=10.152.0.2/16
    ip netns add $VM
    ip link set $VF netns $VM
    ip netns exec $VM ip addr add $IP dev $VF
    ip netns exec $VM ip link set $VF up
    ip netns exec $VM ip route add default via $GW

on second x86 host create two network namespaces to represent VM3 and VM4::

    VM=vm3
    GW=192.168.1.1
    VF=pf0vf0
    IP=192.168.1.2/24
    ip netns add $VM
    ip link set $VF netns $VM
    ip netns exec $VM ip addr add $IP dev $VF
    ip netns exec $VM ip link set $VF up
    ip netns exec $VM ip route add default via $GW

    VM=vm4
    GW=10.152.0.1
    VF=pf0vf1
    IP=10.152.0.3/16
    ip netns add $VM
    ip link set $VF netns $VM
    ip netns exec $VM ip addr add $IP dev $VF
    ip netns exec $VM ip link set $VF up
    ip netns exec $VM ip route add default via $GW

Testing traffic between VM1 and VM3, from the first x86 host::

    ip netns exec vm1 ping 192.168.1.2

Testing traffic between VM2 and VM4, from the first x86 host::

    ip netns exec vm2 ping 10.152.0.3

Verify there is connectivity between VM1 to VM2 as they exist on different domains::

    ip netns exec vm1 ping 10.152.0.2
