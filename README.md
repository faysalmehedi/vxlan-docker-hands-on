## Understanding multi-container host networking using `VxLAN overlay networks`. This hands-on demo will provide an overview of container communication between `multi-node or multi container daemon` under the hood.

#### What is Underlay and Overlay Network?

`Underlay Network` is physical infrastructure above which overlay network is built. It is the underlying network responsible for delivery of packets across networks. Underlay networks can be Layer 2 or Layer 3 networks. Layer 2 underlay networks today are typically based on Ethernet, with segmentation accomplished via VLANs. The Internet is an example of a Layer 3 underlay network.

`An Overlay Network` is a virtual network that is built on top of underlying network infrastructure (Underlay Network). Actually, “Underlay” provides a “service” to the overlay. Overlay networks implement network virtualization concepts. A virtualized network consists of overlay nodes (e.g., routers), where Layer 2 and Layer 3 tunneling encapsulation (VXLAN, GRE, and IPSec) serves as the transport overlay protocol.

#### What is VxLAN? 

`VxLAN — or Virtual Extensible LAN` addresses the requirements of the Layer 2 and Layer 3 data center network infrastructure in the presence of VMs in a multi-tenant environment. It runs over the existing networking infrastructure and provides a means to "stretch" a Layer 2 network.  In short, VXLAN is a Layer 2 overlay scheme on a Layer 3 network.  Each overlay is termed a VXLAN segment.  Only VMs within the same VXLAN segment can communicate with each other.  Each VXLAN segment is identified through a 24-bit segment ID, termed the "VNI".  This allows up to 16 M VXLAN segments to coexist within the same administrative domain.

#### What is VNI?

Unlike VLAN, VxLAN does not have ID limitation. It uses a 24-bit header, which gives us about 16 million VNI’s to use. A VNI `VXLAN Network Identifier (VNI)` is the identifier for the LAN segment, similar to a VLAN ID. With an address space this large, an ID can be assigned to a customer, and it can remain unique across the entire network.

#### What is VTEP?

VxLAN traffic is encapsulated before it is sent over the network. This creates stateless tunnels across the network, from the source switch to the destination switch. The encapsulation and decapsulation are handled by a component called a `VTEP (VxLAN Tunnel End Point)`. A VTEP has an IP address in the underlay network. It also has one or more VNI’s associated with it. When frames from one of these VNI’s arrives at the Ingress VTEP, the VTEP encapsulates it with UDP and IP headers.  The encapsulated packet is sent over the IP network to the Egress VTEP. When it arrives, the VTEP removes the IP and UDP headers, and delivers the frame as normal.


### Packet Walk

#### How traffic passes through a simple VxLAN network.

![Project Diagram](https://github.com/faayam/vxlan-docker-hands-on/blob/main/vxlan_PacketWalk.png)

_the diagrom is taken from networkdirection blog_

- A frame arrives on a switch port from a host. This port is a regular untagged (access) port, which assigns a VLAN to the traffic - The switch determines that the frame needs to be forwarded to another location. The remote switch is connected by an IP network It may be close or many hops away.
- The VLAN is associated with a VNI, so a VxLAN header is applied. The VTEP encapsulates the traffic in UDP and IP headers. UDP port 4789 is used as the destination port. The traffic is sent over the IP network 
- The remote switch receives the packet and decapsulates it. A regular layer-2 frame with a VLAN ID is left 
- The switch selects an egress port to send the frame out. This is based on normal MAC lookups. The rest of the process is as normal.

#### What are we going to cover in this hands-on demo?
- We will use two VM for this, will install docker for running container. 
- We have to create separate subnet and assign static IP address for simplicity.
- Then we will create vxlan bridge using linux "ip link vxlan" feature. 
- Then bind the vxlan to the docker bridge to crate the tunnel
- Hopefully then we see the response from another host conatiner.


### Get an overview of the hands-on from the diagram below
![Project Diagram](https://github.com/faayam/vxlan-docker-hands-on/blob/main/vxlan-diagram.png)

## Let's start...

**_Step 0:_** For this demo, anyone can deploy two VM on any hypervisor or virtualization technology. Make sure they are on the same network thus hosts can communicate each other. I launched two ec2 instance(ubuntu) which is on same VPC from AWS to simulate this hands-on. In case of AWS, please allow all traffic in security group to avoid connectivity issues. 

**_Step 1:_** Install docker client and create separate subnet using docker network utility

##### For Host-01
```bash
# update the repository and install docker
sudo apt update
sudo apt install -y docker.io

# create a separate docker bridge network 
sudo docker network create --subnet 172.18.0.0/16 vxlan-net

c43287381077769a873105e49d34d45f5426e42adf52ef7a92394fe0192715b1

# list all networks in docker
sudo docker network ls

NETWORK ID     NAME        DRIVER    SCOPE
982c9e8e9e40   bridge      bridge    local
2c2b3714bca9   host        host      local
223b98ebd6ef   none        null      local
c43287381077   vxlan-net   bridge    local

# Check interfaces
ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:f5:a4:e1:3a:c2 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.4/24 brd 10.0.1.255 scope global dynamic eth0
       valid_lft 2529sec preferred_lft 2529sec
    inet6 fe80::8f5:a4ff:fee1:3ac2/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:9f:18:ff:df brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: br-c43287381077: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:f2:77:0d:ab brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-c43287381077
       valid_lft forever preferred_lft forever

```
##### For Host-02

```bash
sudo apt update
sudo apt install -y docker.io

sudo docker network create --subnet 172.18.0.0/16 vxlan-net

c485be328b349a64ab32f1743658e09d16d39b3fef83946bbfad662831a92a33

sudo docker network ls

NETWORK ID     NAME        DRIVER    SCOPE
6e1b56c794c3   bridge      bridge    local
d1f9b80443c4   host        host      local
922f1ff12422   none        null      local
c485be328b34   vxlan-net   bridge    local

ip a

1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc fq_codel state UP group default qlen 1000
    link/ether 0a:05:59:dc:14:18 brd ff:ff:ff:ff:ff:ff
    inet 10.0.1.41/24 brd 10.0.1.255 scope global dynamic eth0
       valid_lft 1984sec preferred_lft 1984sec
    inet6 fe80::805:59ff:fedc:1418/64 scope link 
       valid_lft forever preferred_lft forever
3: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:57:11:86:a0 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
4: br-c485be328b34: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:29:42:6b:94 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global br-c485be328b34
       valid_lft forever preferred_lft forever

```
**_Step 2:_** Run docker container on top of newly created docker bridge network and try to ping docker bridge 

##### For Host-01
```bash
# running ubuntu container with "sleep 3000" and a static ip
sudo docker run -d --net vxlan-net --ip 172.18.0.11 ubuntu sleep 3000

a90cbd31d1a20b5e8369f211e65e6fdabf611d01353cb4dc3ff646166ce18e26

# check the container running or not
sudo docker ps

CONTAINER ID   IMAGE     COMMAND        CREATED         STATUS         PORTS     NAMES
a90cbd31d1a2   ubuntu    "sleep 3000"   7 seconds ago   Up 6 seconds             jolly_khayyam

# check the IPAddress to make sure that the ip assigned properly
sudo docker inspect a9 | grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "",
                    "IPAddress": "172.18.0.11",

# ping the docker bridge ip to see whether the traffic can pass
ping 172.18.0.1 -c 2
PING 172.18.0.1 (172.18.0.1) 56(84) bytes of data.
64 bytes from 172.18.0.1: icmp_seq=1 ttl=64 time=0.047 ms
64 bytes from 172.18.0.1: icmp_seq=2 ttl=64 time=0.044 ms

--- 172.18.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1010ms
rtt min/avg/max/mdev = 0.044/0.045/0.047/0.001 ms

```
##### For Host-02
```bash
sudo docker run -d --net vxlan-net --ip 172.18.0.12 ubuntu sleep 3000

7755c18d3550d1fd80d0d00d9dce85dfd8370b2d26cbc088c97bd1b91ac95be7

sudo docker ps

CONTAINER ID   IMAGE     COMMAND        CREATED         STATUS         PORTS     NAMES
7755c18d3550   ubuntu    "sleep 3000"   9 seconds ago   Up 8 seconds             affectionate_black

sudo docker inspect 77 | grep IPAddress

            "SecondaryIPAddresses": null,
            "IPAddress": "",
                    "IPAddress": "172.18.0.12",

ping 172.18.0.1 -c 2

PING 172.18.0.1 (172.18.0.1) 56(84) bytes of data.
64 bytes from 172.18.0.1: icmp_seq=1 ttl=64 time=0.047 ms
64 bytes from 172.18.0.1: icmp_seq=2 ttl=64 time=0.044 ms

--- 172.18.0.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1010ms
rtt min/avg/max/mdev = 0.044/0.045/0.047/0.001 ms

```

**_Step 3:_** Now access to the running container and try to ping another hosts running container via IP Address. Though hosts can communicate each other, conatiner communication should fail because there is no tunnel or anything to carry the traffic.

##### For Host-01
```bash
# enter the running container using exec 
sudo docker exec -it a9 bash
# Now we are inside running container
# update the package and install net-tools and ping tools
apt-get update
apt-get install net-tools
apt-get install iputils-ping

# Now ping the another container
ping 172.18.0.12 -c 2

PING 172.18.0.12 (172.18.0.12) 56(84) bytes of data.
--- 172.18.0.12 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 2028ms
```
##### For Host-02
```bash
sudo docker exec -it 77 bash

# inside container
apt-get update
apt-get install net-tools
apt-get install iputils-ping

# Now ping the another container
ping 172.18.0.11 -c 2

PING 172.18.0.11 (172.18.0.11) 56(84) bytes of data.
--- 172.18.0.11 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 2028ms
```

**_Step 4:_**  It's time to create a VxLAN tunnel to establish communication between two hosts running containers. Then attch the vxlan to the docker bridge. Make sure the VNI ID is the same for both hosts.

##### For Host-01
```bash
# check the bridges list on the hosts
brctl show

bridge name	bridge id		STP enabled	interfaces
br-c43287381077		8000.0242f2770dab	no		veth725a704
docker0		8000.02429f18ffdf	no

# create a vxlan
# 'vxlan-demo' is the name of the interface, type should be vxlan
# VNI ID is 100
# dstport should be 4789 which a udp standard port for vxlan communication
# 10.0.1.41 is the ip of another host
sudo ip link add vxlan-demo type vxlan id 100 remote 10.0.1.41 dstport 4789 dev eth0		

# check interface list if the vxlan interface created
ip a | grep vxlan
9: vxlan-demo: <BROADCAST,MULTICAST> mtu 8951 qdisc noop state DOWN group default qlen 1000

# make the interface up
sudo ip link set vxlan-demo up

# now attach the newly created vxlan interface to the docker bridge we created
sudo brctl addif br-c43287381077 vxlan-demo

# check the route to ensure everything is okay. here '172.18.0.0' part is our concern part.
route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.1.1        0.0.0.0         UG    100    0        0 eth0
10.0.1.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.0.1.1        0.0.0.0         255.255.255.255 UH    100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-c43287381077
```
##### For Host-02
```bash
brctl show

bridge name	bridge id		STP enabled	interfaces
br-c485be328b34		8000.024229426b94	no		veth478bcd1
docker0		8000.0242571186a0	no	

# 10.0.1.4 is the ip of another host
# make sure VNI ID is the same on both hosts, this is important
sudo ip link add vxlan-demo type vxlan id 100 remote 10.0.1.4 dstport 4789 dev eth0

ip a | grep vxlan

9: vxlan-demo: <BROADCAST,MULTICAST> mtu 8951 qdisc noop state DOWN group default qlen 1000

sudo ip link set vxlan-demo up
sudo brctl addif br-c485be328b34 vxlan-demo

route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.1.1        0.0.0.0         UG    100    0        0 eth0
10.0.1.0        0.0.0.0         255.255.255.0   U     0      0        0 eth0
10.0.1.1        0.0.0.0         255.255.255.255 UH    100    0        0 eth0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
172.18.0.0      0.0.0.0         255.255.0.0     U     0      0        0 br-c485be328b34
```
**_Step 5:_** Now test the connectivity. It should work now. A Vxlan Overlay Network Tunnel has been created. 
##### For Host-01
```bash
sudo docker exec -it a9 bash

# ping the other container IP
ping 172.18.0.12 -c 2

PING 172.18.0.12 (172.18.0.12) 56(84) bytes of data.
64 bytes from 172.18.0.12: icmp_seq=1 ttl=64 time=0.601 ms
64 bytes from 172.18.0.12: icmp_seq=2 ttl=64 time=0.601 ms

--- 172.18.0.12 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1018ms
rtt min/avg/max/mdev = 0.601/0.601/0.601/0.000 ms

```
##### For Host-02
```bash
sudo docker exec -it 77 bash

ping 172.18.0.11 -c 2

PING 172.18.0.11 (172.18.0.11) 56(84) bytes of data.
64 bytes from 172.18.0.11: icmp_seq=1 ttl=64 time=0.601 ms
64 bytes from 172.18.0.11: icmp_seq=2 ttl=64 time=0.601 ms

--- 172.18.0.11 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1018ms
rtt min/avg/max/mdev = 0.601/0.601/0.601/0.000 ms
```

#### Now the hands on completed; 


#### Resources
- https://datatracker.ietf.org/doc/html/rfc7348
- http://ce.sc.edu/cyberinfra/workshops/Material/SDN/Lab%205%20-Configuring%20VXLAN%20to%20Provide%20Network%20Traffic%20Isolation.pdf
- https://vincent.bernat.ch/en/blog/2017-vxlan-linux



