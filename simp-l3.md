simple_l3
=========

This directory contains a simple program, called simple_l3.p4, complete
with the sample bfrt_python setup script

The program
===========

The program `simple_l3.p4` can only forward IPv4 packets. It contains just two
tables, `ipv4_host` and `ipv4_lpm`. Both tables match on the IPv4 destination
address in the packet (the program checks that the packet contains an IPv4
header and applies the tables only if it is present). First, `ipv4_host`
table performs an exact match on the IPv4 Destination Address. If there was no
match, then the lookup is performed in `ipv4_lpm` table.

Both tables can execute only two very simple actions: `send()` and `drop()`.
The former switches the packet to a specified port, while the latter simply
drops it.

The user can program these tables to achieve very simple network configurations,
which are still practical.

For example, it can program entries for specific IP addresses to forward
packets to the ports to which corresponding hosts are connected, e.g.

| IP Address    | Action | Parameters |
----------------|:------:|------------|
|192.168.1.1    | send   | port = 1   |
|192.168.1.2    | send   | port = 2   |
|192.168.1.3    | discard|            |
|192.168.1.0/24 | send   | port = 64  |
|192.168.0.0/16 | discard|            |
|0.0.0.0/0      | send   | port = 64  |.

While simple, this program serves as a basis for more elaborate examples and
is also very easy to understand. It provides a good learning exercise for the
tools.

Directory organization:
=======================
```
simple_l3/
   p4src/
      simple_l3.p4 -- the P4 code for the program
      
    bfrt_python/
      setup.py     -- A sample table programming script for run_pd_rpc.py
                      This script can be used with any target

    pkt/
      send.py      -- A simple utility to send a packet with a given
                      IPv4 Destination Address into Tofino-Model
                      
```

Compiling the program
=====================
To compile the program for Tofino simply use p4_build.sh scripts from the tools
directory:
```
~/tools/p4_build.sh ~/labs/01-simple_l3/p4src/simple_l3.p4
```

Running the program
===================

Set up Virtual Ethernet Interfaces
----------------------------------
This step is required if you plan to run the program on both Tofino-Model and 
BMv2-Tofino. Both models use the same port mapping, which works as follows:

```
---------------+
     Model     |
     =====     |
               | veth0               veth1
       Port 0  +--------------------------+
               |
               | veth2               veth3
       Port 1  +--------------------------+
               |
               | veth4               veth5
       Port 2  +--------------------------+    
               |
                                                  This side is for PTF,
     . . .                                        Scapy and other tools

               | veth16             veth17
       Port 8  +--------------------------+
               |
     . . .

               | veth250           veth251
 Port 64 (CPU) +--------------------------+
               |
---------------+
```

To set up virtual ethernet interfaces simply execute the following command
```
sudo $SDE/install/bin/veth_setup.sh
```

Running on Tofino-Model
-----------------------

Change the directory to $SDE

### Run the model
Run Tofino-model in one window:
```
. ~/tools/set_sde.bash
sde
./run_tofino_model.sh -p simple_l3
```

### Run the driver
Run the driver process (switchd) in another window:
```
. ~/tools/set_sde.bash
sde
./run_switchd.sh -p simple_l3
```

### Run bfrt_python

The driver runs a CLI, called `bfshell`. It is actually a "gateway" to several
different interactive shells. A tool with the same name (usually invoked by
running the script `$SDE/run_bfshell.sh`  allows you to access
bfshell via a telnet-like session from another window. The advantage of this
approach is that your session will not be obscured by driver messages.


#### Run the CLI (bfshell) in the third window:
```
. ~/tools/set_sde.bash
sde
./run_bfshell.sh
```
If you have a script, prepared ahead of time, you can run it like so:
```
./run_bfshell.sh -b ~/labs/01-simple_l3/bfrt_python/setup.py [-i]
```

Use -i to stay in the interactive mode after the script has been executed.

#### Injecting and sniffing  the packets

The easiest way to inject the packets is to use scapy. It is important to run
it as `root`, otherwise you will not get access to network interfaces:
```
$ sudo scapy
WARNING: No route found for IPv6 destination :: (no default route?)
Welcome to Scapy (2.2.0-dev)
>>> p=Ether()/IP(dst="192.168.1.1", src="10.10.10.1")/UDP(sport=7,dport=7)/"Payload"
>>> sendp(p, iface="veth1")
.
Sent 1 packets.
```

There is also a convenience script provided for the simple_l3_program:
```
$ sudo python 01-simple_l3/pkt/send.py 192.168.1.1
WARNING: No route found for IPv6 destination :: (no default route?)
Sending IP packet to 192.168.1.1
.
Sent 1 packets.
```

To sniff packets you can use Wireshark (see an icon on your desktop)

Additional experiments and projects
===================================

1. Try to manipulate the size of the tables and see what you can fit. Use the
   visualizations, produced by the compiler to assist you in that. To see
   visualizations, use the following command:
```
show_vis simple_l3
```
or
```
show_p4i simple l3
```
2. Use run_pd_rpc.py tool to experiment with table capacity by filling the
   tables with random entries until the first failure. The code might look
   like this:
```
count=0
while True:
  try:
    e = ipv4_host.add_with_discard(random.randint(0, 0xFFFFFFFF)))
    count += 1
    if count % 1000 == 0:
       print count, "entries added"
  except Exception as e:
    if e.sts == 4:  # Ignore duplicate key error
       continue
    print "Failed to add entry number", count + 1
    break
```
You can also find this code in `run_pd_rpc/fill_ipv_host.py`

3. You can repeat the same experiment, but with batching:
```
count=0
bfrt.batch_begin()
while True:
  try:
    e = ipv4_host.add_with_discard(random.randint(0, 0xFFFFFFFF)))
    count += 1
    if count % 1000 == 0:
       print count, "entries added"
  except Exception as e:
    if e.sts == 4:  # Ignore duplicate key error
       continue
    print "Failed to add entry number", count + 1
    break
bfrt.batch_end()
```
and compare the time it required to add entries
