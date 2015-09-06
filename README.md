EmPOWER
=======

EmPOWER is a SDN/NFV framework fo Enterprise WLANs. EmPOWER is an open source
project providing a WiFi datapath implementation, a reference Controller and a
Python-based SDK. 

Terminology
-----------

An EmPOWER network is composed of:

- One Controller. This must be reachable by the other network elements. The 
controller will run your applications.

- One or more Wireless Termination Points or WTPs. WTPs are the actual point
of radio attachment for WiFi client, i.e. they are essentially WiFi Access 
Points implementing a split-mac architecture using the EmPOWER WiFi data-path.

- One or more Click Packet Processor or CPPs. CPPs are general purpose 
computers typically equipped with multiple Ethernet interfaces. They combine
the switching capabilities of an OpenFlow switch with the processing 
capabilities of a server.

Note: If you want to run EmPOWER as a WLAN Controller you do not need CPPs. 
Converselly if you want to us EmPOWER as a NFV orchestrator you do not need 
WTPs.

Requirements
-----------

For the controller:

- One server-class machine running a recent Linux distribution (I'm currently 
using Fedora 22 and Debian Wheezy). 

For the WTPs:

- One or more WiFi Access Points capable of running OpenWRT Barrier Breaker 
(14.07) with at least one Atheros-based Wireless NIC supported by the ath9k 
driver (I'm currently using PCEngines ALIX 2D boards).

For the CPPs:

- One or more embedded PC running a recent linux distribution and supporting
OpenVSwitch with at least two Ethernet ports (I'm currently using Soekris 
6501-70 boards with 12 Gigabit Ethernet Ports).


Building a WTP from sources
---------------------

Clone the EmPOWER repository and pull all the submodules:

```
  $: git clone http://github.com/rriggio/empower
  $: cd empower
  $: git submodule init
  $: git submodule update
```

Change to the "empower-openwrt" directory, update, and install all feeds:


```
  $: cd empower-openwrt
  $: ./scripts/feeds update -a
  $: ./scripts/feeds install -a
```

EmPOWER Access Points need to be connected to the controller for all their 
operations. However, the default configuration does not specify the address
of the controller. In order to avoid having to configure manually all access
points, you can automatically create an image with the configuration that
suits your needs (note, the following configuration is for a PCEngines ALIX 2D
board equipped with a single Wireless NIC). In order to do so, create a 
directory named "files":

```
  $: mkdir files
```

The OpenWRT buildroot will copy all the content of the "files" directory to the
files image. 

Create a "etc/config" directory:

```
  $: mkdir -p files/etc/config
```

Then create the following files:

 - EmPOWER configuration (etc/config/empower)

```
config empower general
    option "master_ip" "<IP ADDRESS OF YOUR CONTROLLER>"
    option "master_port" "4433"
    option "network" "wan"
    list "ifname" "empower0"
    list "debugfs" "/sys/kernel/debug/ieee80211/phy0/ath9k/bssid_extra"
```

 - Network configuration (etc/config/network)

```
config interface 'loopback'
	option ifname 'lo'
	option proto 'static'
	option ipaddr '127.0.0.1'
	option netmask '255.0.0.0'

config device
	option name 'br-ovs'
	option type 'ovs'
	list ifname 'eth0'

config interface 'wan'
	option ifname 'br-ovs'
	option proto 'dhcp'

config interface lan
	option ifname	eth1
	option proto	static
	option ipaddr	192.168.250.1
	option netmask	255.255.255.0
```

 - Wireless configuration (etc/config/wireless)

```
config wifi-device  radio0
        option type     mac80211
        option channel  36
        option hwmode   11a
        option path     'pci0000:00/0000:00:0c.0'

config wifi-iface empower0
        option device   radio0
        option mode     monitor
        option ifname   moni0
```

Run the configuration application:

```
  $: make menuconfig
```

From the menu select the Target System (e.g. x86) and the Subtarget (e.g. 
PCEngines alix2). Then select "Network -> empower-agent" and 
"Network -> openvswitch-switch".

Save the configuration and exit, then start the compilation. If you have a 
multi core machine you can increase the compilation speed by increasing the
number of parallel builds with th "-j" option.

```
  $: make -j 2
```

Once the compilation is done, the compiled image can be found in the 
bin/<target> directory. For example in the case of a PCEngines Alix platform,
the image will be:

```
  bin/x86/openwrt-x86-alix2-combined-squashfs.img
```

The method of loading the image on the wireless router (flashing) changes 
according to the brand/model of the wireless router. You can find the flashing 
instruction for all supported models [here](http://wiki.openwrt.org/toh/start). 

In the case of the PCEngines Alix board you need to insert the Compact Flash
card in a compact flash reader attached to your laptop and then run:

```
  $: dd if=./bin/x86/openwrt-x86-alix2-combined-squashfs.img of=/dev/sdb
```

Note this assumes that the compact flash device is "/dev/sdb"


Building a CPP
---------------------

The following instructions will guide you trough the process of configuring
a generic Linux machine as a CPP. The instructions are tested on Ubuntu 15.04
server. First update the package repository and install some dependecies:

```
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install build-essential openvswitch-switch python3-websocket
```

Then clone the empower-agent git repository

```
git clone https://github.com/rriggio/empower-agent.git
```

Compile and install click

```
cd empower-agent
./configure --disable-linuxmodule --enable-userlevel --enable-wifi --enable-empower --enable-lvnfs
make 
sudo make install
```

You now need to configure the system to use OpenVSwitch. Open the network
configuration file with

```
sudo vi /etc/network/interfaces 
```

and replace all its content with what follows. Note the this configurationi is
meant for one CPP equipped with 4 Ethernet interfaces and that the first 
interface is used as management interface and requires an active DHCP server.

```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet dhcp

auto allow-ovs br0
iface br0 inet manual
    ovs_type OVSBridge
    ovs_ports eth1 eth2 eth3

allow-br0 eth1
iface eth1 inet manual
    ovs_bridge br0
    ovs_type OVSPort

allow-br0 eth2
iface eth2 inet manual
    ovs_bridge br0
    ovs_type OVSPort

allow-br0 eth3
iface eth3 inet manual
    ovs_bridge br0
    ovs_type OVSPort
```

Running EmPOWER
---------------

The EmPOWER WLAN controller must be executed from a central server that can be 
reached from all wireless APs. From the main repository enter the controller
directory:

```
  $: cd empower-runtime
```

create a directory for the database named deploy:

```
  $: mkdir deploy
```

and start the controller:


```
  $: ./empower-runtime.py
```

If all the python dependencies are satisfied you should see an output like
this:

```
INFO:core:Starting EmPOWER Runtime
INFO:core:Generating default accounts
INFO:core:Loading EmPOWER Runtime defaults
INFO:root:Importing module: empower.core.restserver
INFO:root:Importing module: empower.scylla.lvnfp.lvnfpserver
INFO:root:Importing module: empower.charybdis.lvapp.lvappserver
INFO:root:Importing module: empower.core.energinoserver
INFO:root:Importing module: empower.charybdis.link_stats.link_stats
INFO:root:Importing module: empower.charybdis.events.wtpdown
INFO:root:Importing module: empower.charybdis.events.wtpup
INFO:root:Importing module: empower.charybdis.events.lvapleave
INFO:root:Importing module: empower.charybdis.events.lvapjoin
INFO:root:Importing module: empower.charybdis.counters.packets_counter
INFO:root:Importing module: empower.charybdis.counters.bytes_counter
INFO:root:Importing module: empower.charybdis.maps.ucqm
INFO:root:Importing module: empower.charybdis.maps.ncqm
INFO:root:Importing module: empower.charybdis.triggers.rssi
INFO:root:Importing module: empower.charybdis.triggers.summary
INFO:root:Importing module: empower.scylla.events.cppdown
INFO:root:Importing module: empower.scylla.events.cppup
INFO:root:Importing module: empower.scylla.events.lvnfjoin
INFO:root:Importing module: empower.scylla.events.lvnfleave
INFO:root:Importing module: empower.scylla.handlers.read_handler
INFO:root:Importing module: empower.scylla.handlers.write_handler
INFO:root:Importing module: empower.scylla.stats.lvnf_stats
INFO:core:Registering 'empower.core.restserver'
INFO:core.restserver:REST Server available at 8888
INFO:core:Registering 'empower.scylla.lvnfp.lvnfpserver'
INFO:scylla.lvnfp.lvnfpserver:LVNF Server available at 4422
INFO:core:Registering 'empower.charybdis.lvapp.lvappserver'
INFO:charybdis.lvapp.lvappserver:LVAP Server available at 4433
INFO:core:Registering 'empower.core.energinoserver'
INFO:core.energinoserver:Energino Server available at 5533
INFO:core:Registering 'empower.charybdis.link_stats.link_stats'
INFO:core:Registering 'empower.charybdis.events.wtpdown'
INFO:core:Registering 'empower.charybdis.events.wtpup'
INFO:core:Registering 'empower.charybdis.events.lvapleave'
INFO:core:Registering 'empower.charybdis.events.lvapjoin'
INFO:core:Registering 'empower.charybdis.counters.packets_counter'
INFO:core:Registering 'empower.charybdis.counters.bytes_counter'
INFO:core:Registering 'empower.charybdis.maps.ucqm'
INFO:core:Registering 'empower.charybdis.maps.ncqm'
INFO:core:Registering 'empower.charybdis.triggers.rssi'
INFO:core:Registering 'empower.charybdis.triggers.summary'
INFO:core:Registering 'empower.scylla.events.cppdown'
INFO:core:Registering 'empower.scylla.events.cppup'
INFO:core:Registering 'empower.scylla.events.lvnfjoin'
INFO:core:Registering 'empower.scylla.events.lvnfleave'
INFO:core:Registering 'empower.scylla.handlers.read_handler'
INFO:core:Registering 'empower.scylla.handlers.write_handler'
INFO:core:Registering 'empower.scylla.stats.lvnf_stats'
```

If some Python package is missing you should install it using either the 
Python package managed (pip) or your distirbution package mananager. Make sure
to install the Python3 version of the dependencies.

The controller web interface can be reached at the following URL:

```
http://127.0.0.1:8888/
```

Configuring the Controller
--------------------------

The first time the controller is initialized, three accounts are automatically
created:

```
USERNAME PASSWORD
root     root
foo      foo
bar      bar
```
The first account has administrator privileges, while the other two are user
account.

Log in as root and click on the WTP tab (WTP stands for Wireless Termination 
Points and, in the EmPOWER terminology, is the equivalent to a Wireless AP).

You must then specify the MAC addresses of the WTPs that are authorized to
connect to this controller. You can find the MAC address by logging into the
WTPs using ssh and by using the ifconfig command. The EmPOWER platform uses
the MAC address of the OpenVSwitch bridge (br-ovs) as unique identified for
wach WTP. Add all your WTPs using this procedure.

After all WTPs have been added and if the controller is reachable by the WTPs 
on the ip address specified in the "etc/config/empower" then you should see an 
output like this in the controller terminal:

```
INFO:lvapp.lvappserver:Incoming connection from ('192.168.0.133', 54476)
INFO:lvapp.lvappserver:Hello from 192.168.0.133 seq 46852
INFO:lvapp.lvappserver:Sending caps request to 04:F0:21:09:F9:93
INFO:lvapp.lvappserver:Received caps response from 04:F0:21:09:F9:93
```

Creating your first Slice
-------------------------

The Controller can run multiple virtual networks on top of the same physical 
infrastructure. A virtual network has own set of WTPs/CPPs.

Virtual Networks are requested by regular users and must be approved by a 
network administrator. You can request your first slice using the controller 
web interface by logging with one of the default user account (e.g. foo).

After logging you will be presented with the list of your Virtual Networks 
(which should be empty). Click on the "+" icon and then specify a name (this 
will be your WLAN SSID), an optional short description. Then click on the 
button "Request Virtual Network".

You should now logout from the web interface and login again as administrator.
Click on the "Request" tab and approve the request.

You can now login again as regular user. Your new virtual network should be 
listed in the "Tenants" tab. Click on the virtual network id and you will
be redirected to the Virtual Network management page. Here from the WTP tab
you can add the WTPs to youe virtual network.

Finally, the open source version of EmPOWER does not support WPA authentication,
so you must specify the list of MAC address that can connect to any of the 
Virtual Networks defined in the controller (by default anybody can associate). 
In order to do so click on the "ACL" tab and specify the clients MAC addresses 
that are allowed and/or denied to use the network.

At this point any of the clients that are authorized to use the network should
see the new SSID and should be able to associate to it and to reach the public
Internet (if the wireless APs have a backhaul that is connected to the 
Internet).

